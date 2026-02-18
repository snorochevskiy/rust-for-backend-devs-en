# Backend Structure

By this point, we already know how to create an HTTP server, work with a database, and test both endpoints and database-related functionality. Now, let’s look at how all of these components are typically organized in real-world projects.

## Project Creation

We will create a multi-module (workspace) project consisting of the following crates:

* `persist` — The database abstraction layer: a library containing functions for interacting with the DB.
* `server` — The web server layer: an executable application that imports the `persist` library to handle data storage.

1\) Create a new directory named `test_backend` in a location of your choice.

2\) Inside the `test_backend/` directory, create a `Cargo.toml` file with the following content:

```toml
[workspace]
resolver = "3"
```

The root workspace is ready. Now we can add the member crates.

3\) Open a terminal in the `test_backend/` directory and create the `persist` and `server` modules:

```
cargo new persist --lib
cargo new server
```

After doing this, your root `Cargo.toml` should look like this:

```toml
[workspace]
resolver = "3"
members = ["persist", "server"]
```

4\) In the root `Cargo.toml`, declare the dependencies that we will use across our member crates:

```toml
[workspace]
members = ["server", "persist"]

[workspace.dependencies]
async-trait = "0.1"
thiserror = "1"
tokio = { version = "1", features = ["full"] }
sqlx = {version = "0.8", features = ["postgres", "chrono", "runtime-tokio", "bigdecimal"]}
chrono = "0.4"
bigdecimal = { version = "0.4", features = ["serde"]}

axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

axum-test = "18"
testcontainers = "0.26"
testcontainers-modules = { version = "0.14", features = ["postgres"] }
```

To help you stay oriented, here is what the final project structure will look like:

```
test_backend/
├── Cargo.toml
├── persist
│    ├── Cargo.toml
│    ├── src/
│    │   └── lib.rs
│    └── tests/
│        └── account_storage_tests.rs
├── server
│    ├── Cargo.toml
│    ├── src/
│    │   ├── service.rs
│    │   ├── endpoints.rs
│    │   ├── lib.rs    
│    │   └── main.rs
│    └── tests/
│        └── endpoints_tests.rs
└── migrations/
    ├── 0001_accounts.down.sql
    ├── 0001_accounts.up.sql
    ├── 0002_transactions.down.sql
    └── 0002_transactions.up.sql
```



## Крэйт persist

First, let's add all the necessary dependencies to `persist/Cargo.toml`:

```toml
[package]
name = "persist"
version = "0.1.0"
edition = "2024"

[dependencies]
async-trait = { workspace = true }
sqlx = { workspace = true }
chrono = { workspace = true }
bigdecimal = { workspace = true }
serde = { workspace = true }

[dev-dependencies]
tokio = { workspace = true }
testcontainers = { workspace = true }
testcontainers-modules = { workspace = true }
```

All the crate code will be located in the `persist/src/lib.rs` file:

```rust,noplayground
use bigdecimal::BigDecimal;
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool, postgres::PgPoolOptions};

#[derive(Debug, Serialize, Deserialize, FromRow)]
pub struct Account {
    pub id: i64,
    pub owner_name: String,
    pub balance: BigDecimal,
}

// Interface for interacting with the storage
#[async_trait::async_trait]
pub trait Storage {
    async fn fetch_accounts(&self) -> Result<Vec<Account>, sqlx::Error>;
    async fn create_accounts(
        &self, owner_name: &str, initial_balance: BigDecimal
    ) -> Result<Account, sqlx::Error>;
}

pub struct StorageImpl {
    db: PgPool,
}

impl StorageImpl {
    pub fn new(db: PgPool) -> StorageImpl {
        StorageImpl { db }
    }
    pub async fn from_connection_url(url: &str) -> Result<StorageImpl, sqlx::Error> {
        let pool = PgPoolOptions::new()
            .connect(url).await?;
        Ok(StorageImpl { db: pool })
    }
}

#[async_trait::async_trait]
impl Storage for StorageImpl {
    async fn fetch_accounts(&self) -> Result<Vec<Account>, sqlx::Error> {
        sqlx::query_as("SELECT id, owner_name, balance FROM accounts")
            .fetch_all(&self.db)
            .await
    }

    async fn create_accounts(
        &self, owner_name: &str, initial_balance: BigDecimal
    ) -> Result<Account, sqlx::Error> {
        let mut tx = self.db.begin().await?;
        sqlx::query("INSERT INTO accounts(owner_name, balance) VALUES($1, $2)")
            .bind(owner_name)
            .bind(initial_balance)
            .execute(&mut *tx)
            .await?;
        let result = sqlx::query_as(r#"
                SELECT id, owner_name, balance
                FROM accounts
                WHERE id = currval('accounts_seq')
            "#)
            .fetch_one(&mut *tx)
            .await?;
        tx.commit().await?;
        Ok(result)
    }
}
```

Now, let's add an integration test for the database functionality in `persist/tests/account_storage_tests.rs`:

```rust,noplayground
use bigdecimal::{BigDecimal, FromPrimitive};
use persist::{Storage, StorageImpl};
use sqlx::{migrate::Migrator, postgres::{PgConnectOptions, PgPoolOptions}};
use testcontainers::runners::AsyncRunner;

#[tokio::test]
async fn test_create_and_fetch_account() {
    let container = testcontainers_modules::postgres::Postgres::default()
        .with_password("1111")
        .start().await.unwrap();

    let connection_options = PgConnectOptions::new()
        .host(&container.get_host().await.unwrap().to_string())
        .port(container.get_host_port_ipv4(5432).await.unwrap())
        .database("postgres")
        .username("postgres")
        .password("1111");
    let pool = PgPoolOptions::new()
        .connect_with(connection_options).await.unwrap();

    Migrator::new(std::path::Path::new("../migrations")).await.unwrap()
        .run(&pool).await.unwrap();

    // The storage object under test
    let sut = StorageImpl::new(pool);

    // Initially, the accounts table is empty
    let accounts = sut.fetch_accounts().await.unwrap();
    assert!(accounts.is_empty());

    // Create a new account
    let created_acc = sut.create_accounts(
            "Test-Account-1", BigDecimal::from_f64(1000.0).unwrap()
        ).await
        .unwrap();

    // Fetch all accounts to ensure the newly created account is present
    let accounts = sut.fetch_accounts().await.unwrap();
    assert_eq!(accounts.len(), 1);
    assert_eq!(accounts[0].id, created_acc.id);
    assert_eq!(accounts[0].owner_name, created_acc.owner_name);
    assert_eq!(accounts[0].balance, created_acc.balance);
}
```

> [!NOTE]
> As you might have noticed, the variable containing the `StorageImpl` object under test is named sut. The abbreviation **SUT** stands for System Under Test. Using the name `sut` for the entity being tested makes the test code easier to read.

## The server Crate

Now, let's look at the `server` crate, which uses the `persist` crate as a dependency. The interaction scheme between the modules can be illustrated as follows:

<pre class="ascii-diagram" >
┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐   ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐
┆ server crate               ┆   ┆ persist crate ┆
┆┌───────────┐   ┌──────────┐┆   ┆┌──────────┐   ┆
┆│ endpoints ├──>│ services ├────>│  lib.rs  │   ┆
┆└───────────┘   └──────────┘┆   ┆└──────────┘   ┆
└╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘   └╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘
</pre>


First, let's add all the necessary dependencies to `server/Cargo.toml`:

```toml
[package]
name = "server"
version = "0.1.0"
edition = "2024"

[dependencies]
persist = { path = "../persist" }
thiserror = { workspace = true }
async-trait = { workspace = true }
tokio = { workspace = true }
axum = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
sqlx = { workspace = true }

[dev-dependencies]
bigdecimal = { workspace = true }
axum-test = { workspace = true }
testcontainers = { workspace = true }
testcontainers-modules = { workspace = true }
```

In the `server` crate, we have the following source files:

* `service.rs` — a module containing business logic built around calling functionality from the `persist` crate.
* `endpoints.rs` — a module containing endpoint handler functions that invoke the logic defined in the `service.rs` module.
* `lib.rs` — contains the implementation for creating and configuring the Axum server.
* `main.rs` — contains the `main` function, which launches the server creation function from `lib.rs`.

File `server/src/service.rs`:

```rust,noplayground
use std::sync::Arc;
use persist::{Account, Storage};
use crate::endpoints::NewAcc;

#[derive(Debug, thiserror::Error)]
pub enum AccountServiceError {
    #[error("Database error: (0)")]
    StorageError(#[from] sqlx::Error),
}

// Interface for business logic operations
#[async_trait::async_trait]
pub trait AccountService {
    async fn get_all_accounts(&self) -> Result<Vec<Account>, AccountServiceError>;
    async fn create_new_account(&self, new_acc: NewAcc) -> Result<Account, String>;
}

pub struct AccountServiceImpl {
    storage: Arc<dyn Storage + Send + Sync>,
}

impl AccountServiceImpl {
    pub fn new(storage: Arc<dyn Storage + Send + Sync>) -> AccountServiceImpl {
        AccountServiceImpl { storage }
    }
}

#[async_trait::async_trait]
impl AccountService for AccountServiceImpl {
    async fn get_all_accounts(&self) -> Result<Vec<Account>, AccountServiceError> {
        self.storage.fetch_accounts().await
            .map_err(AccountServiceError::from)
    }
    async fn create_new_account(&self, new_acc: NewAcc) -> Result<Account, String> {
        self.storage.create_accounts(&new_acc.owner_name, new_acc.init_balance).await
            .map_err(|e|e.to_string())
    }
}
```

Note that the `AccountServiceImpl` struct encapsulates the storage via `Arc<dyn Storage + Send + Sync>`. This allows us to inject either the real storage (`StorageImpl`) or a mock for testing. The `Send` and `Sync` bounds are necessary because the trait methods are asynchronous and may run across thread boundaries.

Now, the file where the Axum server is initialized — `server/src/lib.rs`. Notice that we store our business logic object, `AccountServiceImpl`, in the application state, also wrapped in an `Arc<dyn AccountService + Send + Sync>`.

```rust,noplayground
use std::sync::Arc;
use axum::{Router, routing::{get, post}};
use persist::StorageImpl;
use crate::service::{AccountService, AccountServiceImpl};

pub mod service;
pub mod endpoints;

pub struct AppState {
    pub account_service: Arc<dyn AccountService + Send + Sync>,
}

pub async fn run_server() {
    let storage = StorageImpl::from_connection_url(
        "postgres://postgres:1111@localhost/mydb"
    ).await.unwrap();
    let account_service = AccountServiceImpl::new(Arc::new(storage));

    let app = build_app(Arc::new(account_service));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

pub fn build_app(account_service: Arc<dyn AccountService + Send + Sync>) -> Router {
    let state = AppState { account_service };
    let app = Router::new()
        .route("/accounts", get(endpoints::list_accounts))
        .route("/accounts", post(endpoints::create_new_account))
        .with_state(Arc::new(state));
    app
}
```

And finally, the endpoints themselves — `server/src/endpoints.rs`. They are quite simple: they extract the service from the state, call the corresponding methods, and repackage the results into HTTP responses.

```rust,noplayground
use std::sync::Arc;
use axum::{Json, extract::State, http::StatusCode};
use serde::{Deserialize, Serialize};
use sqlx::types::BigDecimal;
use persist::Account;
use crate::AppState;

#[derive(Serialize, Deserialize)]
pub struct NewAcc {
    pub owner_name: String,
    pub init_balance: BigDecimal,
}

pub async fn list_accounts(
    State(state): State<Arc<AppState>>
) -> Result<Json<Vec<Account>>, (StatusCode, String)> {
    match state.account_service.get_all_accounts().await {
        Ok(accounts) => Ok(Json(accounts)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}

pub async fn create_new_account(
    State(state): State<Arc<AppState>>, Json(acc): Json<NewAcc>
) -> Result<(StatusCode, Json<Account>), (StatusCode, String)> {
    match state.account_service.create_new_account(acc).await {
        Ok(account) => Ok((StatusCode::CREATED, Json(account))),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}
```

As you can see, the handlers just take `AccountService` from the state and call its methods.

Lastly, `server/src/main.rs`. In the main function, we simply call the server runner from lib.rs.

```rust,noplayground
#[tokio::main]
async fn main() {
    server::run_server().await
}
```

Also, add an integraton test for endpoints from `server/tests/endpoints_tests.rs`. The test will call the endpoint to create an account, and then check the newlycreated account in the storage.

```rust,noplayground
use std::sync::Arc;
use axum::http::StatusCode;
use bigdecimal::{BigDecimal, FromPrimitive};
use persist::{Account, StorageImpl};
use server::{build_app, endpoints::NewAcc, service::AccountServiceImpl};
use sqlx::{migrate::Migrator, postgres::{PgConnectOptions, PgPoolOptions}};
use testcontainers::runners::AsyncRunner;
use axum_test::TestServer;

#[tokio::test]
async fn test_account_endpoints() {
    let container = testcontainers_modules::postgres::Postgres::default()
        .with_password("1111")
        .start().await.unwrap();

    let connection_options = PgConnectOptions::new()
        .host(&container.get_host().await.unwrap().to_string())
        .port(container.get_host_port_ipv4(5432).await.unwrap())
        .database("postgres")
        .username("postgres")
        .password("1111");
    let pool = PgPoolOptions::new()
        .connect_with(connection_options).await.unwrap();

    Migrator::new(std::path::Path::new("../migrations")).await.unwrap()
        .run(&pool).await.unwrap();

    let storage = StorageImpl::new(pool);
    let account_service = AccountServiceImpl::new(Arc::new(storage));
    let app = build_app(Arc::new(account_service));

    let sut = TestServer::new(app).unwrap();

    // Test account data
    let new_acc = NewAcc {
        owner_name: "John Doe".to_string(),
        init_balance: BigDecimal::from_f64(1000.0).unwrap()
    };

    // Verify account creation
    let create_acc_resp = sut.post("/accounts")
        .json(&new_acc)
        .await;
    create_acc_resp.assert_status(StatusCode::CREATED);
    let created_acc = create_acc_resp.json::<Account>();
    assert_eq!(created_acc.owner_name, new_acc.owner_name);
    assert_eq!(created_acc.balance, new_acc.init_balance);

    // Verify fetching all accounts
    let list_accs_resp = sut.get("/accounts").await;
    list_accs_resp.assert_status_ok();
    let fetched_accs = list_accs_resp.json::<Vec<Account>>();
    assert_eq!(fetched_accs.len(), 1);
    assert_eq!(fetched_accs[0].id, created_acc.id);
    assert_eq!(fetched_accs[0].owner_name, created_acc.owner_name);
    assert_eq!(fetched_accs[0].balance, created_acc.balance);

}
```

## Migrations

Copy the DB schema migration files from the [DB Schema Versioning](db-migrate.md) chapter and place them in the `migrations` directory.

## Running

Now you can run the tests:

```
cargo test
```

Or run the server:

```
cargo run
```

and call the endpoints to ensure they are working (don't forget to make sure the database server is running).

Create an account:

```
curl -i -X POST \
    -H "Content-Type: application/json" \
    -d '{"owner_name": "John Doe", "init_balance": 1000.0}' \
    http://localhost:8080/accounts
```

Get all accounts:

```
curl -i http://localhost:8080/accounts
```

***

That’s it: we have explored how to write backends using Axum and SQLx with a classic layered architecture, featuring a data layer, a business logic layer, and a presentation layer.

