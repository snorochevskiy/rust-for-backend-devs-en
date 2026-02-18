# Prometheus Metrics

The final important topic we will cover in our study of backends is metrics.

In this chapter, we will learn how to collect and expose [Prometheus](https://prometheus.io/docs/introduction/overview/) metrics. The typical workflow looks like this:

![](img/prometheus-system.svg)

The principle is as follows:

* Services from which metrics need to be collected must provide an endpoint (for simplicity, we’ll assume the path is `/metrics`, though it can be anything). this endpoint outputs metrics in the [Prometheus text format](https://prometheus.io/docs/concepts/data_model/).
* The Prometheus service calls this endpoint at specific intervals to retrieve metric values and save them. This process is known as scraping.
* Metrics from the Prometheus service can then be queried in an aggregated form. For example, Grafana is frequently used alongside Prometheus — it’s a web-based service that visualizes metric values through graphs and dashboards.
* Additionally, the Prometheus server can be configured to generate alerts when specific metrics reach defined thresholds. Using the [AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/) service, you can easily integrate Prometheus alerts with services like PagerDuty.

In this chapter, we won't cover the installation and configuration of the Prometheus server itself; instead, we will focus on creating the `/metrics` endpoint that provides data in the format Prometheus expects.

## The metrics Crate

To work with Prometheus metrics, the [prometheus](https://crates.io/crates/prometheus) library is available, which contains:

* Types representing all kinds of Prometheus metrics.
* Functionality for formatting metric values into a format suitable for scraping by a Prometheus server.

However, working directly with the [prometheus](https://crates.io/crates/prometheus) library API isn't always convenient. Therefore, a combination of libraries is typically used:

* [metrics](https://crates.io/crates/metrics) — A facade providing convenient macros for working with metrics.
* [metrics-prometheus](https://crates.io/crates/prometheus) — An implementation of the facade that wraps the [prometheus](https://crates.io/crates/prometheus) crate.

> [!NOTE]
> There is also another crate implementing the _metrics_ facade for Prometheus — [**metrics-exporter-prometheus**](https://crates.io/crates/metrics-exporter-prometheus). It includes a built-in HTTP server, which might be more convenient for applications that do not export their own HTTP API. However, for our examples in this chapter, we will use the _metrics-prometheus_ crate as it is simpler and has a more straightforward API.

To work with metrics, we need to add the following dependencies to `Cargo.toml`:

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"

metrics = "0.24"
metrics-prometheus = "0.11"
prometheus = "0.14"
```

## The Counter Metric

The first metric we will look at is the [Counter](https://prometheus.io/docs/concepts/metric_types/#counter): a numerical counter that represents a single monotonically increasing value that can only increase or be reset to zero on restart.

To work with Counter metrics, the _metrics_ library provides a convenient [counter](https://docs.rs/metrics/latest/metrics/macro.counter.html) macro, which is used as follows:

```rust,noplayground
metrics::counter!("metric_name").increment(1);
```

This call:

* Creates a Counter metric with the given name if it hasn't been registered yet and initializes it to zero.
* Increments the metric value.

Let's look at a basic example:

```rust,noplayground
fn main() {
    // Create a global metrics recorder.
    let recorder = metrics_prometheus::install();

    // Declare that if we have a metric named "first_counter", 
    // its description should be the string "Some description"
    metrics::describe_counter!("first_counter", "Some description");

    // Create and increment a Counter metric named "first_counter"
    metrics::counter!("first_counter").increment(1);
    // Increment "first_counter" by 2
    metrics::counter!("first_counter").increment(2);

    // Create and increment a Counter metric named "second_counter"
    metrics::counter!("second_counter").increment(1);

    // Generate the text representation of the metric values
    // in the format expected by the Prometheus server
    let report: String = prometheus::TextEncoder::new()
        .encode_to_string(&recorder.registry().gather())
        .unwrap();
    
    println!("{report}");
}
```

This program will print:

```properties
# HELP first_counter Some description
# TYPE first_counter counter
first_counter 3
# HELP second_counter second_counter
# TYPE second_counter counter
second_counter 1
```

This is the exact text format used by the Prometheus service when it scrapes metrics from applications by calling the `/metrics` endpoint. This format implies that for each metric, the following is provided:

* `# HELP metric_name description` — текстовое описание метрики.
* `# TYPE metric_name type` — the metric type: `counter` / `gauge` / `histogram`.
* `metric_name value` — the current value of the metric.

***

Now let's look at how to use metrics in conjunction with an Axum server. For this example, we'll modify our "Hello" server by adding a Counter metric that increments every time `/hello` is called.

```rust,noplayground
use std::sync::Arc;
use axum::{Router, extract::State, routing::get};
use metrics_prometheus::Recorder;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    // Initialize the metric collection object. This object is also used to 
    // format the metric values requested by the Prometheus server.
    let recorder = metrics_prometheus::install();

    let state = AppState { recorder };

    let app = Router::new()
        .route("/hello", get(hello))
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    // Increment the counter named "hello_calls"
    metrics::counter!("hello_calls").increment(1);
    "Hello!"
}

// It is assumed that this endpoint is called by a Prometheus scraper.
async fn get_metrics(state: State<Arc<AppState>>) -> String {
    let report: String = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Now, if we start the server and immediately navigate to [http://localhost:8080/metrics](http://localhost:8080/metrics), we will receive an empty response. This happens because the `hello_calls` metric hasn't been initialized with any value yet. However, if we first visit [http://localhost:8080/hello](http://localhost:8080/hello) and then check [http://localhost:8080/metrics](http://localhost:8080/metrics) again, we will see the following:

```properties
# HELP hello_calls hello_calls
# TYPE hello_calls counter
hello_calls 1
```

We can also initialize a metric immediately, for example, to zero. This can be done using the [absolute](https://docs.rs/metrics/latest/metrics/struct.Counter.html#method.absolute) method:

```rust,noplayground
let recorder = metrics_prometheus::install();
// Immediately after creating the Prometheus metrics object,
// initialize the metric to zero
metrics::counter!("hello_calls").absolute(0);
```

## Dimensions

You can add arbitrary key-value attributes to metric values, which are known as dimensions (often referred to as labels in Prometheus).

```rust,noplayground
metrics::counter!("metric_name", "dimension1" => "value1", "dimension2" => "value2")
    .increment(1);
```

Dimensions are typically used to provide additional context or metadata for a metric value.

As an example, let's create a metric that counts the number of requests to each endpoint. To do this, we will create a `number_of_calls` metric with a `"path"` dimension that stores the request's URL path. For convenience, we will handle this logic in a separate middleware.

```rust,noplayground
use std::sync::Arc;
use axum::{Router, extract::{Request, State}, middleware::{Next, from_fn}};
use axum::{response::Response, routing::get};
use metrics_prometheus::Recorder;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    let recorder = metrics_prometheus::install();

    let state = AppState { recorder };

    let app = Router::new()
        .merge(
            // Place the endpoints that should trigger our 
            // metrics middleware into a nested router.
            Router::new()
                .route("/endpoint-1", get(endpoint_1))
                .route("/endpoint-2", get(endpoint_2))
                .layer(from_fn(metrics_middleware))
        )
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn metrics_middleware(request: Request, next: Next) -> Response {
    let path = request.uri().path().to_string();
    metrics::counter!("number_of_calls", "path" => path).increment(1);
    let response = next.run(request).await;
    response
}

async fn endpoint_1() -> &'static str {
    "Endpoint 1"
}

async fn endpoint_2() -> &'static str {
    "Endpoint 2"
}

async fn get_metrics(state: State<Arc<AppState>>) -> String {
    let report = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Run the server and make a request to both [http://localhost:8080/endpoint-1](http://localhost:8080/endpoint-1) and [http://localhost:8080/endpoint-2](http://localhost:8080/endpoint-2). Then, request the metrics from [http://localhost:8080/metrics](http://localhost:8080/metrics).

You should see the following output:

```
# HELP number_of_calls number_of_calls
# TYPE number_of_calls counter
number_of_calls{path="/endpoint-1"} 1
number_of_calls{path="/endpoint-2"} 1
```

## Gauge

Unlike a Counter, a [Gauge](https://prometheus.io/docs/concepts/metric_types/#gauge) metric supports both increasing and decreasing values. This metric is frequently used to represent the current state of a specific value. Examples include: the number of messages currently being processed, CPU load, available disk space, or the number of active database connections.

Here is a simple example demonstrating the capabilities of a Gauge:

```rust,noplayground
fn main() {
    let recorder = metrics_prometheus::install();

    // Set an absolute value for the metric
    metrics::gauge!("my_gauge").set(5);
    // Increment the value
    metrics::gauge!("my_gauge").increment(3);
    // Decrement the value
    metrics::gauge!("my_gauge").decrement(1);

    let report = prometheus::TextEncoder::new()
        .encode_to_string(&recorder.registry().gather())
        .unwrap();

    println!("{report}");
}
```

Program output:

```properties
# HELP my_gauge my_gauge
# TYPE my_gauge gauge
my_gauge 7
```

## Histogram

The [Histogram](https://prometheus.io/docs/concepts/metric_types/#histogram) metric is used to count the frequency distribution of values.

A histogram contains several internal counters, each associated with a specific threshold value. These counters are called **buckets**. By default, a Histogram contains buckets with the following threshold values:\
0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, ∞.

When we record a new value into the metric, a check is performed for every bucket: is the value less than or equal to the bucket's threshold? If it is, the counter for that bucket is incremented.

Thus, we can determine how many recorded values were less than 0.005, how many were less than 0.01, and so on.

![](img/prometheus-histogram.svg)

The Histogram metric is particularly useful for monitoring endpoint response times, database query execution times, etc.

As an example, let's write a middleware that uses a Histogram to measure request processing time. To simulate varying execution times for the request handler, we will use the [rand](https://crates.io/crates/rand) library. Add `rand = "0.9"` to `Cargo.toml`.

```rust,noplayground
use std::{sync::Arc, time::Duration};
use axum::{Router, extract::{Request, State}, middleware::{Next, from_fn}};
use axum::{response::Response, routing::get};
use metrics_prometheus::Recorder;
use tokio::time::Instant;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    let recorder = metrics_prometheus::install();

    let state = AppState { recorder };

    let app = Router::new()
        .merge(
            Router::new()
                .route("/endpoint-1", get(endpoint_1))
                .layer(from_fn(metrics_middleware))
        )
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn metrics_middleware(request: Request, next: Next) -> Response {
    let path = request.uri().path().to_string();
    let start = Instant::now(); // Start measuring execution time
    let response = next.run(request).await;
    let time_in_seconds = start.elapsed().as_secs_f64(); // Record the execution time
    metrics::histogram!("call_duration", "path" => path).record(time_in_seconds);
    response
}

async fn endpoint_1() -> &'static str {
    // Simulate a delay lasting from 0 to 1000 milliseconds
    tokio::time::sleep(Duration::from_millis(rand::random_range(0..1000))).await;
    "Endpoint 1"
}

async fn get_metrics(state: State<Arc<AppState>>) -> String {
    let report = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Now, let's run our server.

First, navigate to [http://localhost:8080/endpoint-1](http://localhost:8080/endpoint-1) in your browser and refresh the page 9 times to reach a total of 10 requests to the endpoint.

Next, go to [http://localhost:8080/metrics](http://localhost:8080/metrics) to retrieve the metrics:

```
# HELP call_duration call_duration
# TYPE call_duration histogram
call_duration_bucket{path="/endpoint-1",le="0.005"} 0
call_duration_bucket{path="/endpoint-1",le="0.01"} 0
call_duration_bucket{path="/endpoint-1",le="0.025"} 1
call_duration_bucket{path="/endpoint-1",le="0.05"} 2
call_duration_bucket{path="/endpoint-1",le="0.1"} 3
call_duration_bucket{path="/endpoint-1",le="0.25"} 3
call_duration_bucket{path="/endpoint-1",le="0.5"} 7
call_duration_bucket{path="/endpoint-1",le="1"} 10
call_duration_bucket{path="/endpoint-1",le="2.5"} 10
call_duration_bucket{path="/endpoint-1",le="5"} 10
call_duration_bucket{path="/endpoint-1",le="10"} 10
call_duration_bucket{path="/endpoint-1",le="+Inf"} 10
call_duration_sum{path="/endpoint-1"} 4.274135234
call_duration_count{path="/endpoint-1"} 10
```

It is easy to see that since the `sleep` call in our endpoint pauses for an interval of 0 to 1000 milliseconds (no longer than 1 second), and we measure execution time in seconds, all buckets after the 1.0 threshold are redundant—we never sleep longer than a second anyway.

In situations where we know the expected value range beforehand, it makes sense to manually define a custom set of buckets. This can be done using the [HistogramOpts](https://docs.rs/prometheus/latest/prometheus/struct.HistogramOpts.html) type:

```rust,noplayground
let custom_buckets = vec![0.1, 0.25, 0.5, 1.0, 5.0];

let opts = HistogramOpts::new("histogram_name", "Metric description")
    .buckets(custom_buckets);
```

Let's look at a simple example where we configure a Histogram metric by specifying that we are interested in buckets for the thresholds: 0.1, 0.25, 0.5, 0.9 и 1.0.

```rust,noplayground
use std::time::Duration;
use prometheus::{Histogram, HistogramOpts};

fn main() {
    let recorder = metrics_prometheus::install();
    // Threshold values of interest for the buckets
    let custom_buckets = vec![0.1, 0.25, 0.5, 0.9, 1.0];
    // Set the configuration (description and buckets) for a Histogram named call_duration
    let opts = HistogramOpts::new("call_duration", "My description")
        .buckets(custom_buckets);
    // Register the configuration for the Histogram named call_duration
    let histogram = Histogram::with_opts(opts).unwrap();
    recorder.register_metric(histogram);

    for _ in 0 .. 10 {
        // Simulate a delay in the range of 0 to 1000 milliseconds
        let latency = Duration::from_millis(rand::random_range(0..1000));
        // Record the next value into the metric
        metrics::histogram!("call_duration").record(latency.as_secs_f64());
    }

    let report = prometheus::TextEncoder::new()
        .encode_to_string(&recorder.registry().gather())
        .unwrap();

    println!("{report}");
}
```

Program output:

```
# HELP call_duration My description
# TYPE call_duration histogram
call_duration_bucket{le="0.1"} 2
call_duration_bucket{le="0.25"} 5
call_duration_bucket{le="0.5"} 6
call_duration_bucket{le="0.9"} 6
call_duration_bucket{le="1"} 10
call_duration_bucket{le="+Inf"} 10
call_duration_sum 4.932
call_duration_count 10
```

As you can see, this Histogram no longer contains "meaningless" buckets.

## Process Metrics

It is often useful to have not only metrics populated directly by the application but also process (program) metrics such as CPU and RAM consumption, the number of running threads, the number of open file descriptors, and so on.

The Rust ecosystem offers many crates that help retrieve host and process metrics. We will use the [metrics-process](https://crates.io/crates/metrics-process) library, which is capable of both fetching process metrics and recording them directly into the corresponding Prometheus metrics.

First, add _metrics-process_ to your `Cargo.toml`:

```toml
metrics-process = "2"
```

The library provides the [metrics\_process::Collector](https://docs.rs/metrics-process/latest/metrics_process/struct.Collector.html) type, which allows you to record process metrics simply by calling:

```rust,noplayground
// Get the collector object
let collector = Collector::default();
// Initialize metric descriptions with default text (optional)
collector.describe();
// Collect process metrics and "push" the values into Prometheus metrics
collector.collect();
```

Let's modify our very first example from this chapter by adding process metrics collection:

```rust,noplayground
use std::sync::Arc;
use axum::{Router, extract::State, routing::get};
use metrics_process::Collector;
use metrics_prometheus::Recorder;

struct AppState {
    recorder: Recorder,
}

#[tokio::main]
async fn main() {
    let recorder = metrics_prometheus::install();
    let state = AppState { recorder };
    let app = Router::new()
        .route("/hello", get(hello))
        .route("/metrics", get(get_metrics))
        .with_state(Arc::new(state));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    metrics::counter!("hello_calls").increment(1);
    "Hello!"
}

async fn get_metrics(state: State<Arc<AppState>>) -> String {
    // Update process metrics
    let collector = Collector::default();
    collector.describe();
    collector.collect();

    let report: String = prometheus::TextEncoder::new()
        .encode_to_string(&state.recorder.registry().gather())
        .unwrap();
    report
}
```

Run the server and navigate to [http://localhost:8080/metrics](http://localhost:8080/metrics). You should see something similar to:

```ini
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 0
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1048576
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 16
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 6397952
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1767575577
# HELP process_threads Number of OS threads in the process.
# TYPE process_threads gauge
process_threads 17
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 1116295168
# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes 0
```

