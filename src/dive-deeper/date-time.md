# Date and Time

## Timers

The Rust standard library does not include built-in functionality for working with calendar dates and astronomical time (such as months or time zones), but it provides the following three types:

* [std::time::Duration](https://doc.rust-lang.org/stable/std/time/struct.Duration.html) — represents a span of time.\
  Example: 5 seconds, 3 minutes, 1 hour.
* [std::time::SystemTime](https://doc.rust-lang.org/std/time/struct.SystemTime.html) — an API for working with the system clock.
* [std::time::Instant](https://doc.rust-lang.org/std/time/struct.Instant.html) — an API for working with a monotonic timer.

#### Duration

The `Duration` type is structured very simply:

```rust
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
pub struct Duration {
    secs: u64,
    nanos: Nanoseconds,
}
```

It allows you to represent very long time intervals (up to 584 billion years) with nanosecond precision.

```rust
use std::time::Duration;

fn main() {
    let d1 = Duration::from_secs(5); // 5 seconds
    let d2 = Duration::from_mins(1); // 1 minute
    let d3 = d2.saturating_sub(d1); // 55 seconds
    println!("{}", d3.as_secs()); // 55
}
```

While the `Duration` type itself has limited functionality, it is used as an input argument for various features. For example, the standard [sleep](https://doc.rust-lang.org/std/thread/fn.sleep.html) function, which pauses the execution of a thread for a specified time, accepts a Duration argument.

#### SystemTime

The system timer tracks astronomical time. Generally, system time is implemented using a chip with a crystal oscillator located on the motherboard, powered by a battery even when the computer is turned off.

If you want to retrieve, for example, the current Unix time (the number of seconds elapsed since January 1, 1970), you should use the system timer.

```rust
use std::time::{Duration, SystemTime};

fn main() {
    // Get the current system time value
    let sys_time_now: SystemTime = SystemTime::now();

    // Convert system time to a Duration relative to 
    // the start of the Unix epoch
    let unix_time_duration: Duration =
            sys_time_now.duration_since(SystemTime::UNIX_EPOCH).unwrap();

    // Convert Duration to the number of seconds
    let unix_time_now: u64 = unix_time_duration.as_secs();

    println!("Now (Unix-time): {unix_time_now}");
}
```

However, there is a catch with the system timer: if you call `SystemTime::now()` and then immediately call `SystemTime::now()` again (especially from another thread), there is a chance that the time returned by the second call will be earlier than the time from the first call. Why this happens is a story for another time, but the key takeaway is: do not use the system timer to measure the execution time of code blocks.

#### Instant

A monotonic timer does not store astronomical time; instead, it tracks the time elapsed since the operating system started. Depending on the computer architecture and operating system, a monotonic timer may be implemented differently. For example, on x86-64 processors, it is often implemented via the TSC (Time Stamp Counter) register.

The most important thing for us is that a monotonic timer guarantees that every subsequent request for the current time will return a value no less than the previous request. Therefore, the monotonic timer is safe to use for measuring code execution time.

```rust
use std::time::{Duration, Instant};

fn main() {
    let start = Instant::now();
    //... какая-то функциональность
    let took: Duration = start.elapsed();
}
```

## chrono

The de facto standard for working with date and time in Rust is the [chrono](https://crates.io/crates/chrono) library.

To represent date and time, `chrono` offers the following types:

* [NaiveDate](https://docs.rs/chrono/latest/chrono/struct.NaiveDate.html) — ISO 8601 date without timezone awareness.
* [NaiveTime](https://docs.rs/chrono/latest/chrono/struct.NaiveTime.html) — ISO 8601 time without timezone awareness.
* [NaiveDateTime](https://docs.rs/chrono/latest/chrono/struct.NaiveDateTime.html) — ISO 8601 date and time without timezone awareness.
* [DateTime](https://docs.rs/chrono/latest/chrono/struct.DateTime.html)<[TimeZone](https://docs.rs/chrono/latest/chrono/trait.TimeZone.html)> — ISO 8601 date and time with timezone awareness.

Let's add chrono as a dependency in `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
chrono = { version = "0.4", features = ["serde"]}
```

> [!NOTE]
> Note that data types from the chrono library support serialization via [serde](https://crates.io/crates/serde), however, this functionality must be explicitly enabled using the "serde" feature.

### NaiveDate

Let's look at the `NaiveDate` type, which allows you to:

* Create a date object.
* Convert a date to a string representation.
* Parse a date from a string.
* Add and subtract years, months, and days.
* Iterate over days, months, and years.

Consider these features in the following example:

```rust,edition2024
# use chrono;
use chrono::{Days, Months, NaiveDate};

fn main() {
    let date: NaiveDate = NaiveDate::from_ymd_opt(2025, 12, 15).unwrap();
    println!("{:?}", date); // 2025-12-15
    println!("{}", date.format("%Y-%m-%d")); // 2025-12-15
    println!("{}", date.format("%Y/%d/%m")); // 2025/15/12

    let date = NaiveDate::parse_from_str("2025-01-01", "%Y-%m-%d").unwrap();
    let three_days = date.iter_days()
        .skip(1)
        .take(3)
        .map(|d|d.to_string())
        .collect::<Vec<_>>()
        .join(",");
    println!("Three days since {date}: {three_days}");
    // Three days since 2025-01-01: 2025-01-02,2025-01-03,2025-01-04

    let new_date = date.checked_add_months(Months::new(2)).unwrap()
        .checked_add_days(Days::new(5)).unwrap();
    println!("{}", new_date); // 2025-03-06
}
```

### NaiveTime

The `NaiveTime` type provides similar functionality, but for time:

```rust,edition2024
# use chrono;
use chrono::NaiveTime;

fn main() {
    let time1: NaiveTime = NaiveTime::from_hms_opt(13, 51, 10).unwrap();
    println!("{:?}", time1); // 13:51:10
    println!("{}", time1.format("%H-%M-%S")); // 13-51-10

    let time2 = NaiveTime::parse_from_str("12:01:00", "%H:%M:%S").unwrap();
    let time_diff = time1.signed_duration_since(time2);
    println!("{} seconds", time_diff.num_seconds()); // 6610 seconds
}
```

### NaiveDateTime

`NaiveDateTime` is a combination of `NaiveDate` and `NaiveTime`.

```rust,edition2024
# use chrono;
use chrono::{NaiveDate, NaiveDateTime, NaiveTime};

fn main() {
    let dt: NaiveDateTime = NaiveDateTime::new(
        NaiveDate::from_ymd_opt(2025, 11, 15).unwrap(),
        NaiveTime::from_hms_nano_opt(13, 51, 10,123456789).unwrap()
    );
    println!("{}", dt); // 2025-11-15 13:51:10.123456789
    println!("{}", dt.format("%Y-%m-%dT%H:%M:%S.%3f")); // 2025-11-15T13:51:10.123

    let dt = NaiveDateTime::parse_from_str(
        "2025-11-15T13:51:10.123", "%Y-%m-%dT%H:%M:%S.%3f"
    ).unwrap();
    let date = dt.date();
    let time = dt.time();
}
```

### DateTime

`DateTime<Tz>` stores the date and time with timezone awareness. The timezone is specified by a generic type argument that implements the [TimeZone](https://docs.rs/chrono/latest/chrono/trait.TimeZone.html) trait.

The chrono library includes three standard implementations for the `TimeZone` trait:

* [Utc](https://docs.rs/chrono/latest/chrono/struct.Utc.html) — UTC UTC timezone (Prime Meridian).
* [Local](https://docs.rs/chrono/latest/chrono/struct.Local.html) — The current system timezone.
* [FixedOffset](https://docs.rs/chrono/latest/chrono/struct.FixedOffset.html) — Stores a fixed offset in <ins>seconds</ins> relative to UTC.

These implementations are easier to understand with an example:

```rust,edition2024
# use chrono;
use chrono::{DateTime, FixedOffset, Local, Utc};
fn main() {
    // Get current date and time in UTC and Local timezone
    let utc: DateTime<Utc> = Utc::now();
    let local: DateTime<Local> = Local::now();
    // Convert to the fixed offset form
    let utc_fixed: DateTime<FixedOffset> = utc.fixed_offset();
    let local_fixed: DateTime<FixedOffset> = local.fixed_offset();

    println!("UTC:   {utc}");
    println!("Local: {local}");
    println!("UTC fixed:   {}", utc_fixed);
    println!("Local fixed: {}", local_fixed);

    // Convert from timezone-aware form to naive form
    println!("UTC -> naive UTC:     {}", utc.naive_utc());
    println!("UTC -> naive Local:   {}", utc.naive_local());
    println!("Local -> naive Local: {}", local.naive_local());
    println!("Local -> naive UTC:   {}", local.naive_utc());
}
```

Program output (the computer running the example was in the +2 timezone):

```
UTC:   2025-12-15 14:42:07.078561733 UTC
Local: 2025-12-15 16:42:07.078569548 +02:00
UTC fixed:   2025-12-15 14:42:07.078561733 +00:00
Local fixed: 2025-12-15 16:42:07.078569548 +02:00
UTC -> naive UTC:     2025-12-15 14:42:07.078561733
UTC -> naive Local:   2025-12-15 14:42:07.078561733
Local -> naive Local: 2025-12-15 16:42:07.078569548
Local -> naive UTC:   2025-12-15 14:42:07.078569548
```

A `DateTime` object can also be created:

* From date and time components.
* From a `NaiveDateTime` object by specifying a timezone.
* By parsing a string.

```rust,edition2024
# use chrono;
use chrono::{DateTime, NaiveDate, TimeZone, Utc};

fn main() {
    let utc1: DateTime<Utc> = Utc.with_ymd_and_hms(2025, 12, 15, 13, 51, 10).unwrap();
    println!("{utc1}"); // 2025-12-15 13:51:10 UTC

    let utc2: DateTime<Utc> =
        NaiveDate::from_ymd_opt(2025, 12, 15).unwrap() // Create NaiveDate
            .and_hms_opt(13, 51, 10).unwrap() // Convert to NaiveDateTime
            .and_utc(); // Add timezone
    println!("{utc2}"); // 2025-12-15 13:51:10 UTC

    let dt = DateTime::parse_from_str(
            "2025-12-15 13:51:10 +0000",
            "%Y-%m-%d %H:%M:%S %z"
        ).unwrap();
    println!("{dt}"); // 2025-12-15 13:51:10 +00:00

    let utc3 = dt.to_utc();
    println!("{utc3}"); // 2025-12-15 13:51:10 UTC
}
```

To get a string representation of a DateTime object, just like with its Naive* counterparts, the [format](https://docs.rs/chrono/latest/chrono/struct.DateTime.html#method.format) method is used. Additionally, there are specific methods for formatting into RFC2822 and RFC3339.

```rust,edition2024
# use chrono;
use chrono::{TimeZone, Utc};

fn main() {
    let utc = Utc.with_ymd_and_hms(2025, 12, 15, 13, 51, 10).unwrap();
    println!("RFC3339: {}", utc.to_rfc3339());
    // RFC3339: 2025-12-15T13:51:10+00:00
    
    println!("RFC2822: {}", utc.to_rfc2822());
    // RFC2822: Mon, 15 Dec 2025 13:51:10 +0000
    
    println!("{}", utc.format("%Y-%m-%d %H:%M:%S.%3f %z"));
    // 2025-12-15 13:51:10.000 +0000
}
```

The last thing to cover is `DateTime<FixedOffset>`. The `FixedOffset` type allows you to define any timezone by specifying its offset in seconds relative to UTC. For example, let's create a UTC+2:30 timezone (an offset of 2 hours and 30 minutes east of Greenwich).

```rust,edition2024
# use chrono;
use chrono::{DateTime, FixedOffset, TimeZone, Utc};

fn main() {
    // Number of seconds in 2 hours and 30 minutes
    let hours_02_minutes_30 = 2 * 3600 + 30 * 60;
    // Timezone with a +02:30 offset
    let tz_02_30: FixedOffset = FixedOffset::east_opt(hours_02_minutes_30).unwrap();

    // Creating a DateTime object from time components
    let dt: DateTime<FixedOffset> =
        tz_02_30.with_ymd_and_hms(2025, 12,15, 13, 51, 10).unwrap();
    println!("{dt}"); // 2025-12-15 13:51:10 +02:30

    // Converting UTC time to UTC+2:30
    let now_utc: DateTime<Utc> = Utc::now();
    let now_02_30: DateTime<FixedOffset> = now_utc.with_timezone(&tz_02_30);
    println!("UTC:    {now_utc}");   // UTC:    2025-12-15 16:42:19.043035374 UTC
    println!("+02:30: {now_02_30}"); // +02:30: 2025-12-15 19:12:19.043036305 +02:30
}
```

