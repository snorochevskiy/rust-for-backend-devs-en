# Application Configuration

In this chapter, we will look at how to read application configurations from text files using the [config](https://crates.io/crates/config) library.

> [!NOTE]
> You may come across many configuration examples online using the [dotenv](https://crates.io/crates/dotenv) library, which was popular at one time but has long been abandoned. There was a slightly more recent fork called [dotenvy](https://crates.io/crates/dotenvy), but it is also abandoned.

## Reading a Configuration File

First, let's add the _config_ crate to `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
config = "0.15"
serde = { version = "1", features = ["derive"] }
```

Suppose we are creating a backend application that works with a relational database. In the configuration file, we will define:

* Database connection settings
* The port our backend application will listen on

We will use TOML as the format for our configuration file, as it is the most popular in the Rust ecosystem. However, the _config_ library also supports JSON, YAML, INI, RON, and Corn.

Let's create the file `config/application.toml` and add the configurations:

```toml
[db]
host = "localhost"
port = 5432
login = "postgres"
password = "1111"

[server]
listen_port = 3000
```

Now, let's write a program that reads this configuration file into a struct object.

```rust
use config::{Config, File};
use serde::Deserialize;

// Root type for configuration
#[derive(Debug, Deserialize, Clone)]
struct AppConfig {
    db: DbConfig,
    server: ServerConfig,
}

// Type for fields in the [db] section
#[derive(Debug, Deserialize, Clone)]
struct  DbConfig {
    host: String,
    port: u32,
    login: String,
    password: String,
}

// Type for fields in the [server] section
#[derive(Debug, Deserialize, Clone)]
struct ServerConfig {
    listen_port: u32,
}

fn main() {
    // Read the config file
    let cfg  = Config::builder()
        .add_source(File::with_name("config/application.toml"))
        .build()
        .unwrap();

    // Convert the configuration into a struct object
    let app_config: AppConfig = cfg.try_deserialize().unwrap();

    println!("{app_config:?}");
}
```

Program output:

```
AppConfig {
  db: DbConfig { host: "localhost", port: 5432, login: "postgres", password: "1111" },
  server: ServerConfig { listen_port: 3000 }
}
```

As you can see, the _config_ library relies on the _serde_ crate to create struct objects from the configuration, so all _serde_ features are available for configuration deserialization.

## Multi-layered Configuration

If you are developing a backend application, you will likely have several configurations for running in different environments, such as:

* A developer's local machine
* Production
* Pre-production
* etc.

In these cases, part of the configuration remains the same across all environments, while other parts are environment-specific. For example, the server might listen on port 8080 in all environments, but the database connection settings will differ everywhere.

For such situations, it is convenient to split the configuration into at least two layers:

* The first layer is a single file (let's call it `default.toml`) containing default parameter values.
* The second layer is environment-specific (`prod.toml`, `dev.toml`, etc.) and stores only the values that differ from those set in `default.toml`.

Let's expand our previous example. Instead of a single `application.toml` file, we will have:

* A file with default configurations — `default.toml`
* A configuration for the developer's environment — `dev.toml`
* A configuration for production — `prod.toml`.

File `config/default.toml`:

```toml
[db]
host = "localhost"
port = 5432
login = "postgres"
password = "1111"

[server]
listen_port = 3000
```

File `config/dev.toml`:

```toml
[db]
host = "dev-db.my.com"
port = 5432
login = "postgres"
password = "1111"
```

File `config/prod.toml`:

```toml
[db]
host = "prod-db.my.com"
port = 5432
login = "prod_user"
password = "prod_passwd"

[server]
listen_port = 8080
```

Now for `main.rs`. In our example, we expect the environment name to be passed via the `PROFILE` environment variable.

```rust
use config::{Config, File};
use serde::Deserialize;

#[derive(Debug, Deserialize, Clone)]
struct AppConfig { db: DbConfig, server: ServerConfig }

#[derive(Debug, Deserialize, Clone)]
struct  DbConfig { host: String, port: u32, login: String, password: String }

#[derive(Debug, Deserialize, Clone)]
struct ServerConfig { listen_port: u32 }

fn main() {
    // Get the environment name
    let profile = std::env::var("PROFILE").unwrap_or_else(|_| "dev".into());

    // Load values from default.toml first, and then from the file
    // corresponding to the environment. Values with matching names will be overwritten.
    let cfg  = Config::builder()
        .add_source(File::with_name("config/default.toml"))
        .add_source(File::with_name(&format!("config/{profile}.toml")))
        .build()
        .unwrap();

    let app_config: AppConfig = cfg.try_deserialize().unwrap();

    println!("{app_config:?}");
}
```

Program output:

```
$ export PROFILE=dev
$ cargo run
AppConfig {
  db: DbConfig { host: "dev-db.my.com", port: 5432, login: "postgres", password: "1111" },
  server: ServerConfig { listen_port: 3000 }
}
```

## Overriding with Environment Variables

Sometimes it is useful to be able to override a parameter value without changing the configuration file, but rather by using an environment variable.

In the example below, we add the ability to override configuration parameter values using environment variables with the prefix "APP__".

```rust
fn main() {
    let profile = std::env::var("PROFILE").unwrap_or_else(|_| "dev".into());

    let cfg  = Config::builder()
        .add_source(File::with_name("config/default.toml"))
        .add_source(File::with_name(&format!("config/{profile}.toml")))
        .add_source( // перезапись
             config::Environment::with_prefix("app").separator("__")
         )
        .build()
        .unwrap();

    let app_config: AppConfig = cfg.try_deserialize().unwrap();

    println!("{app_config:?}");
}
```

For example, if we want to override the value of the `host` parameter in the `[db]` section, we can do so using an environment variable named `APP__db__host`.

Running the program:

```
$ export APP__db__host=XXX
$ cargo run
AppConfig {
  db: DbConfig { host: "XXX", port: 5432, login: "postgres", password: "1111" },
  server: ServerConfig { listen_port: 3000 }
}
```

