# Axum and SQLx

Now that we have figured out how to work with SQLx, let's look at how to use it together with Axum.

We will expand our test_sqlx project (which we created in the [SQLx](sqlx.md) chapter) by adding an Axum server to it. For this simple example, our server will provide only two endpoints:

* `GET /accounts` — get a list of all accounts
* `POST /account` — create a new account

First, add the axum dependency to `Cargo.toml`:

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

axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Now, let's create empty files as boilerplates for our modules:

* `persist.rs` — for database interaction code
* `server.rs` — for the HTTP server code

As a result, we should have the following file tree:

```
test_sqlx/
├── Cargo.toml
├── src/
│   ├── persist.rs
│   ├── server.rs
│   └── main.rs
└── migrations/
    ├── 0001_accounts.down.sql
    ├── 0001_accounts.up.sql
    ├── 0002_transactions.down.sql
    └── 0002_transactions.up.sql
```

First, let's write the functionality for working with the database. We will have two functions:

* One for fetching all accounts from the `accounts` table.
* One for inserting a new account.

Here is our `src/persist.rs` file:

```rust,noplayground
use bigdecimal::BigDecimal;
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool};

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

pub async fn create_accounts(
    db: &PgPool, owner_name: &str, initial_balance: BigDecimal
) -> Result<Account, sqlx::Error> {
    let mut tx = db.begin().await?;
    // Insert a new account
    sqlx::query("INSERT INTO accounts(owner_name, balance) VALUES($1, $2)")
        .bind(owner_name)
        .bind(initial_balance)
        .execute(&mut *tx)
        .await?;
    // Read the newly inserted record.
    // The currval(sequence_name) function, which returns the
    // last sequence value obtained, must be called in the same session
    // as the INSERT query that used that sequence. 
    // This is why we use a transaction.
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
```

Now for our server code. In order for the endpoints to access the database, we will simply place the DB connection pool object into a state object.

Thus, the `src/server.rs` file will look like this:

```rust,noplayground
use std::sync::Arc;

use axum::{Json, Router, extract::State, http::StatusCode, routing::{get, post}};
use serde::Deserialize;
use sqlx::{PgPool, postgres::PgPoolOptions, types::BigDecimal};
use crate::persist::{self, Account};

struct AppState {
    db: PgPool,
}

/// Creates and starts the Axum server
pub async fn run_server() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let state = AppState { db: pool };

    let app = Router::new()
        .route("/accounts", get(list_accounts))
        .route("/accounts", post(create_new_account))
        .with_state(Arc::new(state)); // Place the DB connection pool in the state

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

#[derive(Deserialize)]
struct NewAcc {
    owner_name: String,
    init_balance: BigDecimal,
}

async fn list_accounts(
    state: State<Arc<AppState>>
) -> Result<Json<Vec<Account>>, (StatusCode, String)> {
    // Extract the connection pool object from the state and use it to
    // call the function that fetches all accounts from the DB.
    match persist::fetch_accounts(&state.db).await {
        Ok(accounts) => Ok(Json(accounts)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}

async fn create_new_account(
    state: State<Arc<AppState>>, Json(acc): Json<NewAcc>
) -> Result<Json<Account>, (StatusCode, String)> {
    match persist::create_accounts(&state.db, &acc.owner_name, acc.init_balance).await {
        Ok(account) => Ok(Json(account)),
        Err(e) => Err((StatusCode::INTERNAL_SERVER_ERROR, e.to_string())),
    }
}
```

In the main file `src/main.rs`, we simply call the server startup function:

```rust,noplayground
mod persist;
mod server;

#[tokio::main]
async fn main() {
    server::run_server().await
}
```

For the sake of a clean experiment, let's delete all records from the `transactions` and `accounts` tables that were left over from previous chapters.

If you used Docker to run the Postgres server, you can use these commands:

* `docker start my_pg_container` — start the container if it's inactive
* `docker exec -it my_pg_container bash` — open a console in the container
* `psql -U postgres` — connect to the DB itself
* `\c mydb;` — select _mydb_ as the current database
* `delete from transactions; delete from accounts;` — clear the tables

***

Now, run our application:

```
cargo run
```

First, let's call the endpoint that returns all accounts. Since we just cleared the tables, the response should be an empty list:

```
$ curl -i http://localhost:8080/accounts
HTTP/1.1 200 OK
content-type: application/json
content-length: 2
date: Sat, 03 Jan 2026 02:02:18 GMT

[]
```

Now, let's add a new account:

```
$ curl -i -X POST \
    -H "Content-Type: application/json" \
    -d '{"owner_name": "Acc-1", "init_balance": 1000.0}' \
    http://localhost:8080/accounts
</strong>HTTP/1.1 200 OK
content-type: application/json
content-length: 49
date: Sat, 03 Jan 2026 02:02:20 GMT

{"id":1000,"owner_name":"Acc-1","balance":"1000"}
```

Call the endpoint that returns all accounts again:

```
$ curl -i http://localhost:8080/accounts
HTTP/1.1 200 OK
content-type: application/json
content-length: 51
date: Sat, 03 Jan 2026 02:02:22 GMT

[{"id":1000,"owner_name":"Acc-1","balance":"1000"}]
```

Everything works as expected.

***

As you can see, the state object is the preferred place to store long-lived entities that need to be shared across different endpoints.

