# Testcontainers

It is generally accepted that full-scale integration testing of functionality interacting with a database should only be performed using a real DBMS server. However, setting up and preparing a PostgreSQL server separately for tests is very inconvenient, especially when it comes to testing on a CI server. This is why Docker containers with the required DBMS server are typically used for testing database interactions.

There is a popular library called [Testcontainers](https://testcontainers.com/) that significantly simplifies integration testing with real systems by handling all the routine tasks of:

* Starting the test container at the beginning of the test
* Interacting with the container
* Shutting down the container after the test finishes

> [!TIP]
> If you have written backend applications in Java, C#, Python, Ruby, etc., you have likely already encountered a variation of the Testcontainers library for your language.

***

Based on the database structure created in previous chapters, let's write a simple program that first adds a new account to the database, then reads all accounts from the database and prints them to the console.

To do this, we will write two functions:

* `fetch_accounts()` — reads all accounts from the DB
* `insert_accounts(owner_name, initial_balance)` — inserts a new account with a given name and initial balance

However, this time we are more interested in testing the functions that interact with the database rather than the program itself. The test we will write using Testcontainers will:

1. Spin up a new PostgreSQL container
2. Use SQLx migrations to create tables in the database
3. Call the `insert_accounts` function to insert a new account
4. Call the `fetch_accounts` function and verify that the result contains the account we just created

First, let's add the [testcontainers](https://crates.io/crates/testcontainers) dependency to `Cargo.toml`:

```toml
[package]
name = "test_sqlx"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"] }
sqlx = {version = "0.8", features = ["postgres", "chrono", "runtime-tokio", "bigdecimal"]}
chrono = "0.4"
bigdecimal = { version = "0.4", features = ["serde"]}

[dev-dependencies]
testcontainers = "0.26"
```

Copy the migration files from the project in the previous chapter, [DB Structure Versioning](db-migrate.md). After this, your project file tree should look like this::

```
/
├── Cargo.toml
├── src/
│   └── main.rs
└── migrations/
    ├── 0001_accounts.down.sql
    ├── 0001_accounts.up.sql
    ├── 0002_transactions.down.sql
    └── 0002_transactions.up.sql
```

Now, here is our `src/main.rs`:

```rust,noplayground
use bigdecimal::{BigDecimal, FromPrimitive};
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool, postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    insert_accounts(&pool, "John Doe", BigDecimal::from_i32(1000).unwrap())
        .await.unwrap();

    let accounts = fetch_accounts(&pool).await.unwrap();
    println!("{accounts:?}");
}

#[derive(Debug, Serialize, Deserialize, FromRow)]
pub struct Account {
    id: i64,
    owner_name: String,
    balance: BigDecimal,
}

pub async fn fetch_accounts(db: &PgPool) -> Result<Vec<Account>, sqlx::Error> {
    sqlx::query_as("SELECT id, owner_name, balance FROM accounts")
        .fetch_all(db)
        .await
}

pub async fn insert_accounts(
    db: &PgPool, owner_name: &str, initial_balance: BigDecimal
) -> Result<(), sqlx::Error> {
    sqlx::query("INSERT INTO accounts(owner_name, balance) VALUES($1, $2)")
        .bind(owner_name)
        .bind(initial_balance)
        .execute(db)
        .await?;
    Ok(())
}

#[cfg(test)]
mod test {
    use super::*;
    use sqlx::{migrate::Migrator, postgres::{PgConnectOptions, PgPoolOptions}};
    use testcontainers::{
        core::{IntoContainerPort, WaitFor},
        runners::AsyncRunner, GenericImage, ImageExt
    };

    #[tokio::test]
    async fn test_create_account() {
        // Start the PostgreSQL container
        let container = GenericImage::new("postgres", "18")
            .with_wait_for(WaitFor::message_on_stderr(
                "database system is ready to accept connections"
            ))
            .with_exposed_port(5432.tcp())
            .with_env_var("POSTGRES_PASSWORD", "1111")
            .start()
            .await
            .expect("Postgres started");

        // Connect to the DB in the container
        let connection_options = PgConnectOptions::new()
            .host(&container.get_host().await.unwrap().to_string())
            .port(container.get_host_port_ipv4(5432).await.unwrap())
            .database("postgres")
            .username("postgres")
            .password("1111");

        let pool = PgPoolOptions::new()
            .connect_with(connection_options).await.unwrap();

        // Create tables in the database using SQLx migration scripts
        Migrator::new(std::path::Path::new("./migrations")).await.unwrap()
            .run(&pool).await.unwrap();

        // Insert a new account
        insert_accounts(&pool, "Test-Account-1", BigDecimal::from_i32(1000).unwrap())
            .await.unwrap();

        // Fetch all accounts
        let accounts = fetch_accounts(&pool).await.unwrap();

        // Verify the correctness of the DB response
        assert_eq!(accounts.len(), 1);
        assert_eq!(accounts[0].owner_name, "Test-Account-1".to_string());
        assert_eq!(accounts[0].balance, BigDecimal::from_i32(1000).unwrap());
    }
}
```

Run the test using the `cargo test` command:

```
running 1 test
test test::test_create_account ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 2.58s
```

***

Now, let's break down the section of code where the container is created:

```rust,noplayground
let container = GenericImage::new("postgres", "18")
    .with_wait_for(WaitFor::message_on_stderr(
        "database system is ready to accept connections"
    ))
    .with_exposed_port(5432.tcp())
    .with_env_var("POSTGRES_PASSWORD", "1111")
    .start()
    .await
    .expect("Postgres started");
```

Here:

* `GenericImage::new("postgres", "18")` — indicates that we want to create a container based on the [postgres](https://hub.docker.com/_/postgres) image from [https://hub.docker.com/](https://hub.docker.com/). The "18" argument is the image tag.
* `.with_wait_for` — sets a condition that must be met before we can start working with the container. In our case, we wait for the moment when the PostgreSQL server inside the container prints the string "database system is ready to accept connections" to the console.
* `.with_exposed_port(5432.tcp())` — indicates that we want to map port 5432 from the container to a random free port on the host system. The port number on the host system can be retrieved from the container object.
* `.with_env_var("POSTGRES_PASSWORD", "1111")` — pushes the `POSTGRES_PASSWORD` environment variable, set to `1111`, into the container. According to the image documentation on Docker Hub, this is how we set the database password in the container. The default username is "postgres".

Any other image available on Docker Hub can be spun up in a similar manner. Upon completion of the test, all containers will be automatically shut down, even if the test finished with an error.

## Testcontainers Modules

Alongside the _testcontainers_ crate, there is also the [testcontainers-modules](https://crates.io/crates/testcontainers-modules), который содержит удобные обёртки для популярных Docker образов. Разумеется, обёртка для crate, which contains convenient wrappers for popular Docker images. Naturally, a wrapper for the PostgreSQL image is included among them.

Add the _testcontainers-modules_ dependency to `Cargo.toml`:

```toml
[dev-dependencies]
testcontainers = "0.26"
testcontainers-modules = { version = "0.14", features = ["postgres"] }
```

After this, you can start the container like this:

```rust,noplayground
// Start the PostgreSQL container
let container = testcontainers_modules::postgres::Postgres::default()
    .with_password("1111")
    .start().await.unwrap();

// Connect to the DB in the container
let connection_options = PgConnectOptions::new()
    .host(&container.get_host().await.unwrap().to_string())
    .port(container.get_host_port_ipv4(5432).await.unwrap())
    .database("postgres")
    .username("postgres")
    .password("1111");
```

As you can see, the code for starting the PostgreSQL container has become significantly shorter and simpler.

Additionally, _testcontainers-modules_ offers similar wrappers for Anvil, Azurite, CockroachDB, Clickhouse, Consul, DynamoDB, ElasticSearch, Kafka, Localstack, Minio, MongoDB, MS SQL Server, MySQL, Nats, Neo4J, OpenLDAP, Oracle, OrientDB, RabbitMQ, Redis, RQLitem, Scylladb, Solr, SurrealDB, and Zookeeper.

