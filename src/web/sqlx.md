# SQLx

It's time to talk about working with relational databases.

To interact with relational DBMS, we will use the popular [SQLx](https://crates.io/crates/sqlx) library, which supports PostgreSQL, MySQL, SQLite, and MS SQL Server.

SQLx provides:

* DBMS drivers
* Connection pooling
* An API for executing SQL and receiving responses
* Converters that map database responses into Rust structures

## Test Database

For our examples, we will use the PostgreSQL DBMS. You can install a PostgreSQL distribution locally or use a Docker image.

First, create a new database named `mydb`. If you prefer using Docker, you can use the following commands:

Start the PostgreSQL container:

```
docker run --name my_pg_container \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=1111 \
  -d postgres
```

Enter the console of the running container:

```
docker exec -it my_pg_container bash
```

Run psql (the command-line client for PostgreSQL):

```
psql -U postgres
```

In the psql console, create a new database:

```sql
CREATE DATABASE mydb;
```

Set `mydb` as the current active database:

```
\c mydb;
```

(To view all available databases, use the `\list` command)

***

For our examples, we will need a database with two tables: "bank accounts" and "transaction history".

<pre class="ascii-diagram" >
┌────────────┐         ┌────────────────┐
│  accounts  │         │  transactions  │
├────────────┤         ├────────────────┤
│ id         │───┐     │ id             │
│ owner_name │   │     │ amount         │
│ balance    │   ├────*│ src_account_id │
└────────────┘   └────*│ dst_account_id │
                       │ tx_timestamp   │
                       └────────────────┘
</pre>

Execute the following SQL to create the tables and seed them with test data:

```sql
CREATE SEQUENCE accounts_seq START WITH 1000;

CREATE TABLE accounts ( -- mydb.public.accounts 
    id BIGINT PRIMARY KEY DEFAULT nextval('accounts_seq'),
    owner_name VARCHAR(255) NOT NULL UNIQUE,
    balance NUMERIC(10, 2)  NOT NULL DEFAULT 0.00 CHECK (balance >= 0)
);

CREATE SEQUENCE transactions_seq START WITH 1000;

CREATE TABLE transactions ( -- mydb.public.transactions 
    id BIGINT PRIMARY KEY DEFAULT nextval('transactions_seq'),
    amount NUMERIC(10, 2) DEFAULT 0.00,
    src_account_id BIGINT NOT NULL,
    dst_account_id BIGINT NOT NULL,
    tx_timestamp TIMESTAMP NOT NULL,
    FOREIGN KEY (src_account_id) REFERENCES accounts (id),
    FOREIGN KEY (dst_account_id) REFERENCES accounts (id)
);

INSERT INTO accounts(id, owner_name, balance) VALUES
(1, 'John Doe',    1000.00),
(2, 'Ivan Ivanov', 2000.00);

INSERT INTO transactions(amount, src_account_id, dst_account_id, tx_timestamp)
VALUES
(10.00, 1, 2, TO_TIMESTAMP('2025-12-11 14:00:00', 'YYYY-MM-DD HH24:MI:SS')),
(20.00, 2, 1, TO_TIMESTAMP('2025-12-12 15:00:00', 'YYYY-MM-DD HH24:MI:SS'));
```

You can use the psql command `\dt` to view the list of tables in the database and verify that `accounts` and `transactions` are present.

## Connecting to a DBMS

Now that the database has been created, we can proceed with the Rust program.

Create a new project:

```
cargo new test_sqlx
```

And add the dependencies to `Cargo.toml`:

* [sqlx](https://crates.io/crates/sqlx) — the SQLx library itself. Features:
  * `postgres` enables the implementation of SQLx interfaces for PostgreSQL.
  * `bigdecimal` enables support for the `BigDecimal` type: the SQL `NUMERIC` type is usually converted into `BigDecimal`.
* [chrono](https://crates.io/crates/chrono) — a library for working with dates and times.
* [bigdecimal](https://crates.io/crates/bigdecimal) — a library that provides the [BigDecimal](https://docs.rs/bigdecimal/latest/bigdecimal/struct.BigDecimal.html) type — a high-precision numeric type that avoids precision loss when performing operations on floating-point numbers.

```toml
[package]
name = "test_sqlx"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["postgres", "chrono", "runtime-tokio", "bigdecimal"]}
chrono = "0.4"
bigdecimal = "0.4"
```

Now we can write a program that connects to a PostgreSQL database. To create a connection pool for PostgreSQL, the [PgPoolOptions](https://docs.rs/sqlx/latest/sqlx/postgres/type.PgPoolOptions.html) builder is used. It allows you to configure a variety of connection pool parameters, such as:

* Minimum and maximum number of connections in the pool.
* Timeout for acquiring a connection from the pool, maximum connection lifetime, and maximum idle connection lifetime.
* Callbacks that can be executed: after establishing a connection with the DBMS server, before acquiring a connection from the pool, or after returning a connection to the pool.
* Various logging options.

Let's look at a simple example of connecting to our newly created database:

```rust,noplayground
use sqlx::{Pool, Postgres, postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool: Pool<Postgres> = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();
}
```

To specify the database connection settings directly, one of two methods is used: `connect` (which we used in the example above) or `connect_with`.

#### connect

The [connect](https://docs.rs/sqlx/latest/sqlx/postgres/type.PgPoolOptions.html#method.connect) method defines connection settings using a URL string.

Format: `protocol://login:password@host/database?parameters`.

For example:

- For PostgreSQL: `postgres://mylogin:mypassword@localhost/mydb`\
- For MySQL: `mysql://mylogin:mypassword@host/mydb`\
- For SQLite: `sqlite::memory:` or `sqlite://my.db`

#### connect\_with

The [connect_with](https://docs.rs/sqlx/latest/sqlx/pool/struct.PoolOptions.html#method.connect_with) method sets connection settings using the [PgConnectOptions](https://docs.rs/sqlx/latest/sqlx/postgres/struct.PgConnectOptions.html) struct, which encapsulates parameters such as host, username, password, database name, etc.

`PgConnectionOption` allows for more fine-grained configuration compared to `connect`.

Example usage:

```rust,noplayground
use sqlx::postgres::{PgConnectOptions, PgPoolOptions};

#[tokio::main]
async fn main() {
    // Connection options
    let connection_option = PgConnectOptions::new()
        .host("localhost")
        .username("postgres")
        .password("1111")
        .database("mydb");

    // Creating a connection pool
    let pool = PgPoolOptions::new()
        .connect_with(connection_option)
        .await
        .unwrap();
}
```

## Fetching Data

### Typed Queries

For fetching data, SQLx provides the [QueryAs](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html) type, which allows you to execute a SQL query and convert the database response into struct objects of a corresponding type.

A `QueryAs` object is typically created using the [sqlx::query\_as](https://docs.rs/sqlx/latest/sqlx/fn.query_as.html) function.

```rust,noplayground
let query: QueryAs<'_, Postgres, ResultType, PgArguments> = sqlx::query_as(
    "SELECT field1, field2, field3 FROM table_name"
);
```

Next, the `QueryAs` object must be passed a database connection pool object so it can acquire a connection and execute the query. One of the following methods is used for this:

* [fetch_all](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.fetch_all) — expects the query result to be a collection of records.
* [fetch_one](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.fetch_one) — expects exactly one record as a result.
* [fetch_optional](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.fetch_optional) — expects no more than one record (zero or one).

If we want to select multiple records and get the result as a vector of objects, the code would look something like this:

```rust,noplayground
#[derive(FromRow)]
struct ResultType {
   field1: Type1,
   field2: Type2,
   field3: Type3,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new().connect("PG_URL").await.unwrap();
    
    let query: QueryAs<'_, Postgres, ResultType, PgArguments> = sqlx::query_as(
        "SELECT field1, field2, field3 FROM table_name"
    );

    let result: Vec<ResultType> = query.fetch_all(&pool).await.unwrap();
}
```

As you may have noticed, the struct into which the database response is mapped must implement the [FromRow](https://docs.rs/sqlx/latest/sqlx/trait.FromRow.html) trait. Additionally, the field names of the struct must match the names of the corresponding columns in the SQL query result.

Consider an example of fetching data from the `accounts` table in our database:

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, prelude::FromRow, types::BigDecimal};

#[derive(Debug, FromRow)]
struct Account {
    id: i64,
    owner_name: String,
    balance: BigDecimal,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    // Fetching a list of records
    let all_accounts: Vec<Account> = sqlx::query_as(
            "SELECT id, owner_name, balance FROM accounts"
        ).fetch_all(&pool).await.unwrap();
    for acc in all_accounts {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
    // 1: John Doe, 1000
    // 2: Ivan Ivanov, 2000

    // Fetching a single record
    let opt_acc_1: Option<Account> = sqlx::query_as(r#"
            SELECT id, owner_name, balance FROM accounts WHERE owner_name=$1
        "#)
        .bind("John Doe") // Binding the value to the $1 placeholder
        .fetch_optional(&pool)
        .await.unwrap();
    if let Some(acc) = opt_acc_1 {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
    // 1: John Doe, 1000
}
```

As you can see, the principle is simple—you need to:

1. Write the SQL query.
2. Create a struct with the same set of fields as the columns in the SQL query result.
3. Use the `query_as` function.

***

`query_as` is used only when selecting two or more columns. If you need to select only a single column, use the [query\_scalar](https://docs.rs/sqlx-core/latest/sqlx_core/query_scalar/fn.query_scalar.html) function instead. It behaves exactly the same way but converts the database result into simple types (such as strings and numbers) rather than struct objects.

For example:

```rust,noplayground
use sqlx::{postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    // Selecting a single column
    let account_ids: Vec<i64> = sqlx::query_scalar("SELECT id FROM accounts")
        .fetch_all(&pool).await.unwrap();
    println!("All IDs: {account_ids:?}"); // All IDs: [1, 2]

    // Selecting a single value
    let accounts_count: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM accounts")
        .fetch_one(&pool).await.unwrap();
    println!("Number of accounts: {accounts_count}"); // Number of accounts: 2
}
```

***

Now, let's look at an example using a JOIN query: we will retrieve the entire transaction history along with the names of the sender and the recipient. To get the names associated with the account IDs, we will JOIN the `transactions` table with the `accounts` table.

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, prelude::FromRow, types::BigDecimal};
use chrono::NaiveDateTime;

#[derive(Debug, FromRow)]
struct TransactionFullInfo {
    id: i64,
    amount: BigDecimal,
    src_account_owner_name: String,
    dst_account_owner_name: String,
    tx_timestamp: NaiveDateTime,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();

    let transaction: Vec<TransactionFullInfo> = sqlx::query_as(r#"
            SELECT
                tx.id as id, tx.amount, tx.tx_timestamp,
                src_acc.owner_name as src_account_owner_name,
                dst_acc.owner_name as dst_account_owner_name
            FROM
                transactions tx
                JOIN accounts src_acc ON tx.src_account_id = src_acc.id
                JOIN accounts dst_acc ON tx.dst_account_id = dst_acc.id
    "#)
    .fetch_all(&pool).await.unwrap();

    for tx in transaction {
        println!(
            "TXID:{} amount={}, timestamp={}, src: {}, dst: {}",
            tx.id, tx.amount, tx.tx_timestamp,
            tx.src_account_owner_name, tx.dst_account_owner_name
        );
    }
// TXID:1 amount=10, timestamp=2025-12-11 14:00:00, src: John Doe, dst: Ivan Ivanov
// TXID:2 amount=20, timestamp=2025-12-12 15:00:00, src: Ivan Ivanov, dst: John Doe
}
```

***

If you need to provide arguments within the query text, you can pass values to them using the [bind](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.bind) method.

As an example, let's write a query that calculates the total sum of transactions for a specific sender and recipient, but only for transactions where the amount exceeds a certain threshold.

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, types::BigDecimal};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let total_amount: BigDecimal = sqlx::query_scalar(r#"
            SELECT
                SUM(amount)
            FROM transactions tx
                JOIN accounts src_acc ON tx.src_account_id = src_acc.id
                JOIN accounts dst_acc ON tx.dst_account_id = dst_acc.id
            WHERE
                amount >= $1
                AND src_acc.owner_name = $2
                AND dst_acc.owner_name = $3
        "#)
        .bind(20.0)
        .bind("Ivan Ivanov")
        .bind("John Doe")
        .fetch_one(&pool).await.unwrap();

    println!("{total_amount}"); // 20
}
```

It is important to note that SQLx uses [prepared statement](https://www.postgresql.org/docs/current/sql-prepare.html) to pass query arguments. This means that the database server parses the SQL code and generates an execution plan only during the first execution. All subsequent calls to the same query with different argument values will reuse the execution plan from the previous calls, improving performance and security.

### Untyped Queries

The `QueryAs` type allows you to perform a selection and immediately convert the result into struct objects. However, there is another query type — [Query](https://docs.rs/sqlx/latest/sqlx/query/struct.Query.html), which does not handle such conversion. Instead, it returns the result as a collection of untyped `Row` records, which are similar to hash maps.

A `Query` object is created using the [sqlx::query](https://docs.rs/sqlx/latest/sqlx/fn.query.html) function, which is very similar to `sqlx::query_as`. To execute a `Query` object, just like with `QueryAs`, you need to use one of the `fetch_*` methods:

```rust,noplayground
let query: Query<'_, Postgres, PgArguments> = sqlx::query(
    "SELECT field1, field2, field3 FROM my_table"
);
let rows: Vec<PgRow> = query.fetch_all(&pool).await.unwrap();
```

The query result is represented by objects of the [PgRow](https://docs.rs/sqlx/latest/sqlx/postgres/struct.PgRow.html) type (it would be [MySqlRow](https://docs.rs/sqlx-mysql/latest/sqlx_mysql/struct.MySqlRow.html) for MySQL or [SqliteRow](https://docs.rs/sqlx-sqlite/latest/sqlx_sqlite/struct.SqliteRow.html) for SQLite).

To extract column values from a `PgRow`, the [try\_get](https://docs.rs/sqlx/latest/sqlx/trait.Row.html#method.try_get) method is used.

As an example, let's rewrite the previous JOIN query example using `sqlx::query`:

```rust,noplayground
use chrono::NaiveDateTime;
use sqlx::postgres::PgRow;
use sqlx::{postgres::PgPoolOptions, types::BigDecimal};
use sqlx::Row;

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();

    let rows: Vec<PgRow> = sqlx::query(r#"
            SELECT
                tx.id as id, tx.amount, tx.tx_timestamp,
                src_acc.owner_name as src_account_owner_name,
                dst_acc.owner_name as dst_account_owner_name
            FROM
                transactions tx
                JOIN accounts src_acc ON tx.src_account_id = src_acc.id
                JOIN accounts dst_acc ON tx.dst_account_id = dst_acc.id
        "#)
        .fetch_all(&pool)
        .await
        .unwrap();
 
    for r in rows {
        let id: i64 = r.try_get("id").unwrap();
        let amount: BigDecimal = r.try_get("amount").unwrap();
        let ts: NaiveDateTime = r.try_get("tx_timestamp").unwrap();
        let src: String = r.try_get("src_account_owner_name").unwrap();
        let dst: String = r.try_get("dst_account_owner_name").unwrap();

        println!("TXID:{id} amount={amount}, timestamp={ts}, src: {src}, dst: {dst}");
    }
// TXID:1 amount=10, timestamp=2025-12-11 14:00:00, src: John Doe, dst: Ivan Ivanov
// TXID:2 amount=20, timestamp=2025-12-12 15:00:00, src: Ivan Ivanov, dst: John Doe
}
```

## Inserting Data

Now let's look at how to insert data into tables.

To perform INSERT, UPDATE, and DELETE queries, we use the `Query` type we are already familiar with. However, to execute the query, the [execute](https://docs.rs/sqlx/latest/sqlx/query/struct.Query.html#method.execute) method is used instead of the `fetch_*` methods.

Consider an example:

```rust,noplayground
use sqlx::{postgres::{PgPoolOptions, PgQueryResult}, types::BigDecimal};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let result: PgQueryResult = sqlx::query(
            "INSERT INTO accounts(owner_name, balance) VALUES($1, $2)"
        )
        .bind("Some Name")
        .bind(BigDecimal::default())
        .execute(&pool).await.unwrap();
    println!("{result:?}"); // PgQueryResult { rows_affected: 1 }
}
```

The execute method returns a [PgQueryResult](https://docs.rs/sqlx-postgres/latest/sqlx_postgres/struct.PgQueryResult.html) struct object, which stores the number of rows actually inserted or modified.

***

If we have a whole collection of entities that we want to insert into a table, it is convenient to use the [QueryBuilder](https://docs.rs/sqlx/latest/sqlx/struct.QueryBuilder.html) type. Its usage looks like this:

```rust,noplayground
struct Entity { // The entity type associated with the database table
    field1: Type1,
    field2: Type2,
    field3: Type3,
}

// Create a QueryBuilder object with the INSERT query header
let mut qb = QueryBuilder::new(r#"INSERT INTO my_table(field1, field2, field3)"#);

let entity_vector: Vec<Entity> = ...; // collection of entities for insertion

qb.push_values(&entity_vector, |mut builder, entity| {
    // bind entity fields to the columns from the INSERT query header
    builder
        .push_bind(&entity.field1)
        .push_bind(&entity.field2)
        .push_bind(&entity.field3);
});

qb.build().execute(&pool).await.unwrap(); // execute the query
```

As an example, let's look at a program that inserts several new records into the `accounts` table:

```rust,noplayground
use sqlx::{
    Postgres, QueryBuilder, types::BigDecimal,
    postgres::{PgPoolOptions, PgQueryResult}
};

struct NewAcc {
    owner_name: String,
    balance: BigDecimal,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb")
        .await
        .unwrap();

    let new_accounts = vec![
        NewAcc {owner_name: "Name 1".to_string(), balance: BigDecimal::default()},
        NewAcc {owner_name: "Name 2".to_string(), balance: BigDecimal::default()},
        NewAcc {owner_name: "Name 3".to_string(), balance: BigDecimal::default()},
    ];

    let mut qb: QueryBuilder<'_, Postgres> =
        QueryBuilder::new(r#"INSERT INTO accounts(owner_name, balance)"#);

    qb.push_values(&new_accounts, |mut builder, acc| {
        builder
            .push_bind(&acc.owner_name)
            .push_bind(&acc.balance);
    });

    let result: PgQueryResult = qb.build().execute(&pool).await.unwrap();
    println!("{result:?}"); // PgQueryResult { rows_affected: 3 }
}
```

## Transactions

Now let's explore how to work with transactions in SQLx.

A transaction can be created by calling the [begin](https://docs.rs/sqlx/latest/sqlx/struct.Pool.html#method.begin) method on the database connection pool object. This call returns a [Transaction](https://docs.rs/sqlx/latest/sqlx/struct.Transaction.html) object, which encapsulates the transaction. This transaction object can then be used instead of the connection pool (`Pool`) to execute queries in methods like `fetch_all`, `fetch_one`, `execute`, and so on. All queries executed on the transaction object will be performed within the scope of that transaction.

```rust,noplayground
#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new().connect("PG_URL").await.unwrap();
    
    let mut tx = pool.begin().await.expect("Cannot start transaction");
    
    sqlx::query("UPDATE ...").execute(&mut *tx).await?;
    sqlx::query("INSERT ...").execute(&mut *tx).await?;
    sqlx::query("DELETE ...").execute(&mut *tx).await?;

    tx.commit().await.expect("Cannot commit");
}
```

How does it work?

First, if we look at the signature of the [Query::execute](https://docs.rs/sqlx/latest/sqlx/query/struct.Query.html#method.execute) method, we see that it doesn't take a connection pool ([Pool](https://docs.rs/sqlx/latest/sqlx/struct.Pool.html)) as an argument, but rather something called an [Executor](https://docs.rs/sqlx/latest/sqlx/trait.Executor.html). Without going into too much detail, simply put, both `Pool` and `Transaction` implement this `Executor` trait. This is precisely why queries can be executed on either the connection pool object or the transaction object.

Now let's look at the `Transaction` type itself. It works as follows:

* When we call the `begin()` method on the connection pool, a connection is pulled from the pool, and a `BEGIN TRANSACTION` call is immediately sent to it.
* Calling the `commit()` method causes a `COMMIT` to be sent to the connection, which in turn finalizes the transaction.
* If the `Transaction` object is dropped (destroyed) before the `commit()` method is called on it, its destructor will send a `ROLLBACK` command to the database connection.

As an example, let's consider a function that performs a money transfer from one account to another. This function must perform three operations transactionally:

1. Deduct money from the sender's account.
2. Add money to the recipient's account.
3. Create a record of the transaction.

```rust,noplayground
use bigdecimal::FromPrimitive;
use sqlx::{PgPool, postgres::PgPoolOptions, types::BigDecimal};

struct Transfer {
    src_account_id: i64,
    dst_account_id: i64,
    amount: BigDecimal,
}

async fn make_transfer(transfer: &Transfer, pool: &PgPool) -> Result<(),sqlx::Error> {
    // Start the transaction
    let mut tx = pool.begin().await?;

    // Add the transfer amount to the recipient's balance
    let _ = sqlx::query("UPDATE accounts SET balance = balance + $1 WHERE id = $2")
        .bind(&transfer.amount)
        .bind(&transfer.dst_account_id)
        .execute(&mut *tx)
        .await?;

    // Decrease the sender's balance by the transfer amount
    let _ = sqlx::query("UPDATE accounts SET balance = balance - $1 WHERE id = $2")
        .bind(&transfer.amount)
        .bind(&transfer.src_account_id)
        .execute(&mut *tx)
        .await?;

    // Create a new record for the money transfer transaction
    let _ = sqlx::query(r#"
            INSERT INTO transactions(
                amount, src_account_id, dst_account_id, tx_timestamp
            ) VALUES ($1, $2, $3, NOW())
        "#)
        .bind(&transfer.amount)
        .bind(&transfer.src_account_id)
        .bind(&transfer.dst_account_id)
        .execute(&mut *tx)
        .await?;

    // Commit the transaction
    tx.commit().await?;

    Ok(())
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let transfer = Transfer {
        src_account_id: 1,
        dst_account_id: 2,
        amount: BigDecimal::from_f64(50.0).unwrap()
    };

    let _ = make_transfer(&transfer, &pool).await.unwrap();
}
```

Now, if we look at the `accounts` and `transactions` tables, we can see the changes made by the program:

```
mydb=# select * from accounts;
 id | owner_name  | balance
----+-------------+---------
  1 | John Doe    |  950.00
  2 | Ivan Ivanov | 2050.00
```

```
mydb=# select * from transactions;
 id | amount | src_account_id | dst_account_id |        tx_timestamp
----+--------+----------------+----------------+----------------------------
  1 |  10.00 |              1 |              2 | 2025-12-11 14:00:00
  2 |  20.00 |              2 |              1 | 2025-12-12 15:00:00
  3 |  50.00 |              1 |              2 | 2025-12-13 02:00:22.788004
```

To see how a transaction rollback occurs, let's try to make a transfer of an amount that exceeds the current balance in the sender's account. This will trigger a `CHECK (balance > 0)` constraint in the `accounts` table.

```rust,noplayground
#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let transfer = Transfer {
        src_account_id: 1,
        dst_account_id: 2,
        amount: BigDecimal::from_f64(5000.0).unwrap() // The account doesn't have this much money
    };

    let _ = make_transfer(&transfer, &pool).await.unwrap();
}
```

As you can see, we tried to transfer 5000, which caused the second `UPDATE` operation (the one subtracting the amount from the sender's account) in the `make_transfer` function to fail with an error.

```
Database(PgDatabaseError {
    severity: Error, code: "23514",
    message: "new row for relation 'accounts' violates check constraint 'accounts_balance_check'",
    detail: Some("Failing row contains (1, John Doe, -4050.00)."),
    schema: Some("public"),
    table: Some("accounts"),
    constraint: Some("accounts_balance_check"),
    routine: Some("ExecConstraints")
})
```

The transaction was completely rolled back as a result.

