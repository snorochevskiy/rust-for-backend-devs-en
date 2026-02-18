# Logging

As we explore logging in Rust, we will focus on the tracing library. This library is an entire ecosystem, but it centers around two core crates:

* [tracing](https://crates.io/crates/tracing) — provides the logging API (acts as a facade).
* [tracing-subscriber](https://crates.io/crates/tracing-subscriber) — the actual logger implementation for

The Rust ecosystem also features another popular logging library — [log](https://crates.io/crates/log) (a facade) and its implementations like [env_logger](https://crates.io/crates/env_logger) and [log4rs](https://crates.io/crates/log4rs). However, we are focusing on tracing for two main reasons:

* tracing was designed with asynchrony in mind, which is crucial for modern backend applications.
* tracing can handle libraries that still use the log crate for their logging.

## A First Look at tracing

To get started, we need to add the following dependencies to our `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "time", "json"] }
```

Now we are ready to use the logger:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init(); // Enable tracing logging
    tracing::info!("Hello"); // Write an info-level message to the log
}
```

If we run the program, we will see a log message in the console like this:

```
$ cargo run
2025-12-15T23:04:17.046455Z  INFO test_rust: Hello
```

Оно состоит из следующих компонентов:

* `2025-12-15T23:04:17.046455Z` — the timestamp of the log message.
* `INFO` — the log level.
* `test_rust` — the name of the program/package.
* `Hello` — the actual log message text.

## Under the Hood

Let’s take another look at our example:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();
    tracing::info!("Hello");
}
```

The line `tracing_subscriber::fmt().init()` is shorthand for several operations. For the sake of clarity, let's break it down:

```rust,noplayground
use tracing_subscriber::FmtSubscriber;
use tracing_subscriber::fmt::SubscriberBuilder;
use tracing_subscriber::util::SubscriberInitExt;

fn main() {
    let builder: SubscriberBuilder = tracing_subscriber::fmt();
    let subscriber: FmtSubscriber = builder.finish();
    subscriber.init(); // Register the subscriber in the global dispatcher

    tracing::info!("Hello");
}
```

Here is what is happening:

1. `tracing_subscriber::fmt()` — creates a [SubscriberBuilder](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.SubscriberBuilder.html), which allows you to configure how logs are handled.
2. `builder.finish()` — creates the ([Subscriber](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.Subscriber.html)) object. This is the component that actually processes log messages. (In the example, the type is `FmtSubscriber`, which is just an alias for the generic `Subscriber` struct).
3. `subscriber.init()` — рregisters the subscriber object with the global dispatcher.
4. `tracing::info!("Hello")` — sends a log event to the global dispatcher, which then routes it to the registered subscriber for processing.

Simplified, the logging mechanism looks like this:

![](img/logging-tracing_subscriber_architecture.svg)

For us as users of the tracing library, the most important element is the `Subscriber` itself.

```rust,noplayground
pub struct Subscriber<
    N = format::DefaultFields, // List of fields in the log message
    E = format::Format<format::Full>, // Format of the log message
    F = LevelFilter, // Log levels: TRACE, DEBUG, INFO, WARN, ERROR
    W = fn() -> io::Stdout, // Output destination (default is STDOUT)
> {
    inner: layer::Layered<F, Formatter<N, E, W>>,
}
```

At first glance, this structure might look intimidating, but by the end of this chapter, we will have explored all its components. For now, let’s move on to the practical side of logging.

> [!NOTE]
> If you decide to dive into the library's source code, keep in mind that there is a trait named [Subscriber](https://docs.rs/tracing-core/latest/tracing_core/subscriber/trait.Subscriber.html) (defined in the tracing crate) and a struct also named [Subscriber](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.Subscriber.html) (in the tracing-subscriber crate). In this chapter, we mostly deal with the `Subscriber` struct. However, if you ever need to implement a highly custom logger (e.g., one that writes logs directly to a database), you would likely need to implement the `Subscriber` trait.

## Log Record Format

The [SubscriberBuilder](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.SubscriberBuilder.html) we've already encountered allows you to customize the logging format for a subscriber. The builder provides the following methods for configuring log record formatting:

* `with_ansi(bool)` — Specifies whether the logger should use ANSI escape codes to highlight different parts of the log entry with colors. This flag should be set to false when redirecting logs from the console to a file. Default: `true`.
* `with_file(bool)` — Whether to include the name of the file where the log originated. Default: `false`.
* `with_line_number(bool)` —  Whether to include the line number where the log originated. Default: `false`.
* `with_target(bool)` — Whether to include the target (program name/module path). Default: `true`.
* `with_thread_ids(bool)` — Whether to include the ID of the thread where the log originated. Default: `false`.
* `with_thread_names(bool)` — Whether to include the name of the thread where the log originated. Default: `false`.
* `with_timer(`[`Format`](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/format/struct.Format.html)`)` — Defines the date and time format in the log record. Default: [UtcTime](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/time/struct.UtcTime.html).
* `without_time()` — Disables date and time display in the log record.
* `json()` — Formats log records as JSON objects. Only available when the "json" feature is enabled.

As an example, let's configure the log format to remove the application name but include the file name and line number. Additionally, instead of wall-clock time, we'll display the elapsed time since the program started.

```rust,noplayground
use tracing_subscriber::fmt::time::Uptime;

fn main() {
    tracing_subscriber::fmt()
        .with_target(false)
        .with_file(true)
        .with_line_number(true)
        .with_timer(Uptime::default())
        .init();

    tracing::info!("Hello");
}
```

Program output:

```
$ cargo run
0.000186881s  INFO src/main.rs:11: Hello
```

Formatting as a JSON object is also worth noting:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().json().init();
    tracing::info!("Hello");
}
```

The program will output:

```
$ cargo run
{"timestamp":"2025-12-17T14:28:29.197314Z","level":"INFO","fields":{"message":"Hello"},"target":"test_rust"}
```

## Log Levels

_tracing_ offers five logging levels: TRACE, DEBUG, INFO, WARN, and ERROR. By default, all messages with level INFO and higher are logged.

This means a program like this:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();

    tracing::trace!("Hello");
    tracing::debug!("Hello");
    tracing::info!("Hello");
    tracing::warn!("Hello");
    tracing::error!("Hello");
}
```

Will output:

```
$ cargo run
2025-12-16T14:57:02.786497Z  INFO test_rust: Hello
2025-12-16T14:57:02.786556Z  WARN test_rust: Hello
2025-12-16T14:57:02.786567Z ERROR test_rust: Hello
```

To set the visible log level, use the [with_max_level](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.SubscriberBuilder.html#method.with_max_level) method from the `SubscriberBuilder`. For instance, if we want to see DEBUG, INFO, WARN, and

```rust,noplayground
tracing_subscriber::fmt()
    .with_max_level(LevelFilter::DEBUG)
    .init();
```

## Message Filtering

If you enable DEBUG or TRACE levels in an application that has dependencies (other Rust libraries), you will likely encounter a flood of messages coming from the dependency code alongside your own.

Fortunately, _tracing-subscriber_ allows you to filter log messages based on criteria such as:

* target — typically the crate name
* module
* log level

Filters are defined on the `SubscriberBuilder` using the [with_env_filter](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.SubscriberBuilder.html#method.with_env_filter) method, which takes an argument of type `impl Into<EnvFilter>`.

While you can construct an [EnvFilter](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/filter/struct.EnvFilter.html) object manually, it’s much easier to define it via a string. This string should contain comma-separated sections in the format `target::module=level`, where:

* target (optional) — the name of the crate originating the log.
* module (optional) — the name of the module within that crate.
* level — the logging level.

For example, the string `"myapp::mod1=debug,myapp::mod2=trace,info"` specifies that the entire application should use the INFO level, but the `mod1` module from the myapp crate should use DEBUG, and `mod2` from `myapp` should use TRACE.

Note that the order of sections in the string doesn't matter: changing it to `"info,myapp::mod1=debug,myapp::mod2=trace"` yields the same result.

Let's look at an example with two modules: one set to DEBUG, another to WARN, and the rest of the application to INFO.

```rust,noplayground
fn main() {
    tracing_subscriber::fmt()
        .with_env_filter("test_rust::mod_a=debug,test_rust::mod_b=warn,info")
        .init();

    mod_a::func();
    mod_b::func();
    tracing::info!("from root: test");
}

mod mod_a {
    pub fn func() {
        // This will be printed because the level for this module is DEBUG
        tracing::debug!("from mod_a: test");
    }
}

mod mod_b {
    pub fn func() {
        // This won't be printed because the level for this module is WARN
        tracing::info!("from mod_b: test");
    }
}
```

Running the program:

```
$ cargo run
2025-12-16T20:09:58.958779Z DEBUG test_rust::mod_a: from mod_a: test
2025-12-16T20:09:58.958855Z  INFO test_rust: from root: test
```

Alternatively, instead of providing filtering rules as a single string, you can create the `EnvFilter` object manually:

```rust,noplayground
let filter = tracing_subscriber::EnvFilter::from_default_env()
    .add_directive("test_rust::mod_a=debug".parse().unwrap())
    .add_directive("test_rust::mod_b=warn".parse().unwrap())
    .add_directive("info".parse().unwrap());

tracing_subscriber::fmt()
    .with_env_filter(filter)
    .init();
```

However, this is rarely done in practice because it is standard in Rust to define log filtering options via the `RUST_LOG` environment variable.

To support this, simply configure the filter like this:

```rust,noplayground
tracing_subscriber::fmt()
    .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
    .init();
```

Now you can set your filtering options through the environment variable:

```
$ export RUST_LOG="test_rust::mod_a=debug,test_rust::mod_b=warn,info"
$ cargo run
2025-12-16T20:09:58.958779Z DEBUG test_rust::mod_a: from mod_a: test
2025-12-16T20:09:58.958855Z  INFO test_rust: from root: test
```

> [!NOTE]
> On Windows:
> 
> ```
> set RUST_LOG="test_rust::mod_a=debug,test_rust::mod_b=warn,info"
> cargo run
> 2025-12-16T20:09:58.958779Z DEBUG test_rust::mod_a: from mod_a: test
> 2025-12-16T20:09:58.958855Z  INFO test_rust: from root: test
> ```

## Writing Logs to a File

By default, logs are written to standard output. However, you can change this by specifying an alternative sink using the [with_writer](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/struct.SubscriberBuilder.html#method.with_writer) method on the `SubscriberBuilder`.

This method accepts any object that implements the [MakeWriter](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/trait.MakeWriter.html) trait. This includes all types that implement [std::io::Write](https://doc.rust-lang.org/std/io/trait.Write.html). Since `std::fs::File` implements `std::io::Write`, a basic file-logging setup can be implemented as follows:

```rust,noplayground
fn main() {
    let file = std::fs::File::create("app.log").unwrap();
    tracing_subscriber::fmt()
        .with_writer(file)
        .with_ansi(false) // Console ANSI escape codes are not needed in a file
        .init();
    tracing::info!("Hello");
}
```

After running this program, you will see that the `app.log` file was indeed created.

Of course, long-running backend applications require more flexible file logging. The tracing ecosystem provides the [tracing-appender](https://crates.io/crates/tracing-appender) library, which supports:

* Non-blocking log recording.
* Automatic rotation to a new log file every day/hour/minute/etc.
* Automatic deletion of old log files.

First, add _tracing-appender_ to your `Cargo.toml`:

```toml,noplayground
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "time"] }
tracing-appender = "0.2"
```

Now, let's configure logging so that logs are written to a logs directory, the files have the prefix app.log, and a new file is created every day.

```rust,noplayground
fn main() {
    let appender = tracing_appender::rolling::daily("logs", "app.log");

    tracing_subscriber::fmt()
        .with_writer(appender)
        .with_ansi(false)
        .init();

    tracing::info!("Hello");
}
```

After running the program, a logs directory should appear in the project root containing a file named in the format `app.log.YYYY-MM-DD`.

If you need more granular configuration, use the [RollingFileAppender](https://docs.rs/tracing-appender/latest/tracing_appender/rolling/struct.RollingFileAppender.html) builder.

```rust,noplayground
use tracing_appender::rolling::{RollingFileAppender, Rotation};

fn main() {
    let appender = RollingFileAppender::builder()
        .rotation(Rotation::DAILY) // Rotate log files daily
        .max_log_files(10) // Keep only the 10 most recent log files
        .filename_prefix("app") // Filename prefix before the date component
        .filename_suffix("log") // Filename suffix after the date component
        .build("logs") // Directory name for log files
        .unwrap();

    tracing_subscriber::fmt()
        .with_writer(appender)
        .with_ansi(false)
        .init();

    tracing::info!("Hello");
}
```

Once executed, a log file named like `app.YYYY-MM-DD.log` (for example, `app.2025-12-17.log`) should appear in the `logs` folder.

If you want to write logs to both a file and the console simultaneously, you can use the [and](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/writer/trait.MakeWriterExt.html#method.and) combinator from the [MakeWriterExt](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/writer/trait.MakeWriterExt.html) utility trait. This allows you to combine two log sinks into one.

```rust,noplayground
use tracing_subscriber::fmt::writer::MakeWriterExt;

fn main() {
    let file_appender = tracing_appender::rolling::daily("logs", "app.log")
        .with_max_level(tracing::Level::INFO);

    let stdout_appender = std::io::stdout
        .with_max_level(tracing::Level::DEBUG);

    tracing_subscriber::fmt()
        .with_writer(stdout_appender.and(file_appender))
        .with_ansi(false)
        .init();

    tracing::info!("Hello");
}
```

If you require even more fine-tuned configuration for each individual sink, you will need to use Layers, which we will discuss later in this chapter.

## Non-blocking Logging

As mentioned in the previous section, the _tracing-appender_ library provides not only flexible file logging but also non-blocking logging functionality.

To transform a synchronous writer into a non-blocking one, use the [non_blocking](https://docs.rs/tracing-appender/latest/tracing_appender/non_blocking/index.html) function:

```rust,noplayground
fn main() {
    let (non_blocking, _guard) = tracing_appender::non_blocking(std::io::stdout());

    tracing_subscriber::fmt()
        .with_writer(non_blocking)
        .init();

    tracing::info!("Hello");
}
```

The `non_blocking` function spawns a separate thread that receives log messages via a channel and performs the actual writing. It returns a tuple of two objects:

1. An object of type [NonBlocking](https://docs.rs/tracing-appender/latest/tracing_appender/non_blocking/struct.NonBlocking.html), which is essentially a wrapper around the channel's log message Sender.
2. A guard object of type [WorkerGuard](https://docs.rs/tracing-appender/latest/tracing_appender/non_blocking/struct.WorkerGuard.html). Its destructor ensures that all messages remaining in the channel are written before shutting down the thread and dropping the channel. This is essential for a clean application shutdown.

> [!WARNING]
> Important! If new log messages arrive faster than the background thread can write them, the channel will overflow, and subsequent messages will be dropped. The default channel capacity is set to 128,000 messages.

## Layers

[Layer](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/trait.Layer.html) is a trait that defines an interface for filtering, formatting, and recording logs—much like a subscriber. However, unlike subscribers, multiple layers can be composed into one, and this combined layer can then be used as a subscriber. Each layer can be configured as flexibly as a standalone subscriber.

For example, we can create one layer for logging to standard output and another for logging to a file. Combining these layers will result in logs being sent to both destinations.

To create a logger from layers, we use the [Registry](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/registry/struct.Registry.html) object, created via `tracing_subscriber::registry()`. This is best understood through an example:

```rust,noplayground
use tracing_subscriber::{fmt::layer, layer::SubscriberExt, util::SubscriberInitExt};

fn main() {
    // Layer for file logging
    let file_layer = layer()
        .with_writer(tracing_appender::rolling::daily("logs", "app.log"))
        .with_ansi(false);

    // Layer for console logging
    let stdout_layer = layer()
        .with_writer(std::io::stdout);

    tracing_subscriber::registry() // registry() вместо fmt() 
        .with(stdout_layer)
        .with(file_layer)
        .init();

    tracing::info!("Hello");
}
```

When running this program, you will see the log message appear in both the console and the log file.

***

At this point, let's take another look at the `Subscriber` structure definition we saw at the beginning of the chapter:

```rust,noplayground
pub struct Subscriber<
    N = format::DefaultFields, // List of log message fields
    E = format::Format<format::Full>, // Log message format
    F = LevelFilter, // Log levels: TRACE, DEBUG, INFO, WARN, ERROR
    W = fn() -> io::Stdout, // Destination, STDOUT by default
> {
    inner: layer::Layered<F, Formatter<N, E, W>>, // Layers
}
```

As you can see, this structure contains an `inner` field of type [Layered](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/struct.Layered.html) — a storage for layers. In fact, all the subscribers from our previous examples were composed of layers; they simply contained only one.

We can now refine our earlier diagram by showing the internal structure of a standard subscriber (from the _tracing-subscriber_ library).

![](img/logging-layers.svg)

This means that when we configure a logger as simply as:

```rust,noplayground
tracing_subscriber::fmt().init()
```

it creates a subscriber with a single layer that prints logs to STDOUT.

## Span

The final concept to introduce is the[span](https://docs.rs/tracing/latest/tracing/span/index.html#the-span-lifecycle).

A **span** represents a specific period of time or a block of code to which a logging context with additional attributes can be attached.

This is much easier to understand with an example:

```rust,noplayground
use tracing::{Span, span::Entered};

fn main() {
    tracing_subscriber::fmt().init();

    // Create a span for INFO logs with an attribute "attr1" set to 5
    let my_span: Span = tracing::span!(tracing::Level::INFO, "span1", attr1 = 5);
    let _enter: Entered<'_> = my_span.enter();
    
    tracing::info!("Hello");
}
```

This program will output:

```
2025-12-17T23:07:23.633043Z  INFO span1{attr1=5}: test_rust: Hello
```

As you can see, `span1{attr1=5}` was added to the log line.

How it works:

* First, we create a [Span](https://docs.rs/tracing/latest/tracing/struct.Span.html) object, specifying that for logs with the INFO level, we want to include an attribute named "attr1" with the value `5`.
* Next, we "activate" the span in the current scope by calling the [enter](https://docs.rs/tracing/latest/tracing/struct.Span.html#method.enter) method. This returns an [Entered](https://docs.rs/tracing/latest/tracing/span/struct.Entered.html) object, representing the active span.
* Within the scope where the `Entered` object is alive, all logs will include the attributes defined in the span.

The following example demonstrates that span attributes are added only within the scope where the `Entered` object exists:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();

    let my_span = tracing::span!(tracing::Level::INFO, "span1", attr1 = 5);
    {
        let mut _enter = my_span.enter();
        tracing::info!("Hello 1");
    }
    tracing::info!("Hello 2");
}
```

Program output:

```
2025-12-17T23:35:55.528112Z  INFO span1{attr1=5}: test_rust: Hello 1
2025-12-17T23:35:55.528148Z  INFO test_rust: Hello 2
```

***

In most situations, we want to activate a span immediately after creating it. However, we cannot write it like this:

```rust,noplayground
let my_span: Entered<'_> = tracing::span!(Level::INFO, "span1", attr1 = 5).enter();
```

This is because the `Entered` object holds a reference to the Span object from which it was created. In the code above, the `Span` is a temporary object that is created and immediately destroyed, leaving the `Entered` object with a reference to nothing.

To solve this, the `Span` type provides another method — [entered](https://docs.rs/tracing/latest/tracing/struct.Span.html#method.entered) which returns an [EnteredSpan](https://docs.rs/tracing/latest/tracing/span/struct.EnteredSpan.html). It behaves similarly to `Entered`, but instead of holding a reference, it takes <ins>ownership</ins> of the `Span` object.

```rust,noplayground
use tracing::{Level, span::EnteredSpan};

fn main() {
    tracing_subscriber::fmt().init();

    let my_span: EnteredSpan = tracing::span!(Level::INFO, "span1", attr1 = 5)
        .entered();
    tracing::info!("Hello 1");
    my_span.exit();
    tracing::info!("Hello 2");
}
```

Program output:

```
2025-12-18T00:34:31.045841Z  INFO span1{attr1=5}: test_rust: Hello 1
2025-12-18T00:34:31.045887Z  INFO test_rust: Hello 2
```

In practice, the `.entered()` method is used more frequently because it is more concise. However, in this chapter, we will continue to use `.enter()` as it makes the underlying mechanics more explicit.

### Span Nesting

When we create a span object in a scope where another span is already active, the currently active span becomes the parent of the newly created one. This means that log messages will include attributes from both the parent and the child spans.

Consider this example:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();

    let span1 = tracing::span!(tracing::Level::INFO, "span1", attr1 = 5);
    let _enter1 = span1.enter();

    let span2 = tracing::span!(tracing::Level::INFO, "span2", attr2 = 7);
    let _enter2 = span2.enter();
    
    tracing::info!("Hello");
}
```

Program output:

```
2025-12-17T23:08:54.836930Z  INFO span1{attr1=5}:span2{attr2=7}: test_rust: Hello
```

Now, let's rewrite this example so that the second span object is created before the first span is activated. In this situation, the parent-child relationship will not be established, and the last activated span will simply overshadow whatever was activated before it.

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();

    let span1 = tracing::span!(tracing::Level::INFO, "span1", attr1 = 5);
    let span2 = tracing::span!(tracing::Level::INFO, "span2", attr2 = 7);

    let _enter1 = span1.enter();
    let _enter2 = span2.enter(); // will overshadow span1
    
    tracing::info!("Hello");
}
```

Program output:

```
2025-12-17T23:09:36.704697Z  INFO span2{attr2=7}: test_rust: Hello
```

Additionally, it is worth noting that the `span!` macro allows you to explicitly specify a parent span using the `parent` parameter.

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();

    let span1 = tracing::span!(tracing::Level::INFO, "span1", attr1 = 5);
    let span2 = tracing::span!(parent: &span1, tracing::Level::INFO, "span2", attr2 = 7);

    let _enter1 = span1.enter();
    let _enter2 = span2.enter();
    
    tracing::info!("Hello");
}
```

This program now outputs attributes from both spans:

```
2025-12-18T00:55:30.040449Z  INFO span1{attr1=5}:span2{attr2=7}: test_rust: Hello
```

### instrument

The _tracing_ library provides the `instrument` macro, which allows you to "instrument" a function definition by wrapping its body in a span.

Consider the following example:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();
    let _ = func(1, "PARAM2");
}

#[tracing::instrument]
fn func(param1: i32, param2: &str) -> String {
    tracing::info!("Hello");
    String::from("RESULT")
}
```

Program output:

```
2025-12-18T15:27:16.330514Z  INFO func{param1=1 param2="PARAM2"}: test_rust: Hello
```

As you can see, instrumentation creates a span with the same name as the function and with attributes matching the function's arguments.

The `instrument` macro offers several parameters for customization:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt()
        .without_time()
        .init();
    let _ = func(1, "PARAM2");
}

#[tracing::instrument(
    name = "span_func", // span name
    level = "info",     // log level
    fields(attr1 = %param1, attr2="xxx"), // additional attributes
    skip(param1), // do not include param1 in attributes
    ret // enable automatic logging of the function result
)]
fn func(param1: i32, param2: &str) -> String {
    tracing::info!("Hello");
    String::from("RESULT")
}
```

Program output:

```
INFO span_func{param2="PARAM2" attr1=1 attr2="xxx"}: test_rust: Hello
INFO span_func{param2="PARAM2" attr1=1 attr2="xxx"}: test_rust: return="RESULT"
```

### Internal Span Mechanics

When a span object is activated, its data is placed in thread-local storage, which allows access to the span object to be maintained even within nested functions.

For example:

```rust,noplayground
fn main() {
    tracing_subscriber::fmt().init();

    let span = tracing::span!(tracing::Level::INFO, "span1", attr1 = 5);
    let _entered = span.enter();
    
    func();
}

fn func() {
    tracing::info!("Hello");
}
```

Program output:

```
2025-12-18T00:58:56.030620Z  INFO span1{attr1=5}: test_rust: Hello
```

As you can see, the attributes from the span were included in the log output from the nested function.

> [!WARNING]
> When writing async code, spans may behave incorrectly if an `await` call is inserted within the scope of an active span object. We will discuss this in more detail later.

