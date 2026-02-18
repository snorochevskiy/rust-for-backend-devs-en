# SQLx Macros

As a language with strong static typing, Rust attempts to detect as many errors as possible during the compilation stage. However, if we make a mistake in the text of an SQL query, we typically only find out during its execution. While writing integration tests with a real DBMS for all queries is a best practice, it is not the only way to protect yourself from SQL errors.

The `cargo sqlx` utility, which we introduced in the previous chapter, has the ability to connect to a database and verify the correctness of SQL query code. The operating principle is as follows:

1. Instead of the functions `query_as`, `query_scalar`, and `query`, we must use the corresponding macros: [query_as!](https://docs.rs/sqlx/latest/sqlx/macro.query_as.html), [query_scalar!](https://docs.rs/sqlx/latest/sqlx/macro.query_scalar.html) and [query!](https://docs.rs/sqlx/latest/sqlx/macro.query.html). These macros work on the same principle as their function counterparts. Immediately after writing code with these macros, the compiler will complain and recommend running the `cargo sqlx prepare` command.
2. Therefore, we run that command:\
   `cargo sqlx prepare --database-url postgres://postgres:1111@localhost/mydb`\
   The `sqlx` utility will connect to the database and use it to validate all SQL queries used in the `query_as!`, `query_scalar!`, and `query!` macros.\
   Additionally, the utility will create a `.sqlx/` directory containing JSON files with meta-information about the verified queries.
3. When compiling the application, the `query_as!`, `query_scalar!`, and `query!` macros will validate the SQL query text using the meta-information from the `.sqlx` directory.

In this way, we achieve SQL query correctness checking at the compilation stage.

## query_as!, query_scalar! and query!

The usage format for the `query_as!` macro is practically the same as the `query_as` function, with minimal differences:

```rust,noplayground
struct ResultType {
   field1: Type1,
   field2: Type2,
   field3: Type3,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new().connect("PG_URL").await.unwrap();
    
    let query = sqlx::query_as!(
        ResultType,
        "SELECT field1, field2, field3 FROM table WHERE field1 = $1 AND field2 = $2",
        ArgumentValue$1,
        ArgumentValue$2,
    );

    let result: Vec<ResultType> = query.fetch_all(&pool).await.unwrap();
}
```

As you can see, the structure into which the result is converted no longer needs to implement the `FromRow` trait. This is logical: the macro is executed at compile time, so it already has access to the structure's fields.

Also, the `sqlx::query_as!` macro takes at least two arguments: the structure representing the query result and the SQL query itself. Recall that the `query_as` function takes only one argument — the SQL query.

If the query code contains arguments, the values for them are passed immediately after the query itself, whereas with functions, we used the `bind` method on the `Query` object.

***

Using the macro, let's rewrite our `query_as` example from the previous chapter.

```rust,noplayground
use sqlx::{postgres::PgPoolOptions, types::BigDecimal};

#[derive(Debug)]
struct Account {
    id: i64,
    owner_name: String,
    balance: BigDecimal,
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let all_accounts: Vec<Account> = sqlx::query_as!(
            Account,
            "SELECT id, owner_name, balance FROM accounts"
        ).fetch_all(&pool).await.unwrap();
    for acc in all_accounts {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
 
    let opt_acc_1: Option<Account> = sqlx::query_as!(
            Account,
            "SELECT id, owner_name, balance FROM accounts WHERE owner_name=$1",
            "John Doe" // Value for argument $1
        )
        .fetch_optional(&pool)
        .await.unwrap();
    if let Some(acc) = opt_acc_1 {
        println!("{}: {}, {}", acc.id, acc.owner_name, acc.balance.to_string());
    }
}
```

Now, run the command:

```
$ cargo sqlx prepare --database-url postgres://postgres:1111@localhost/mydb
query data written to .sqlx in the current directory;
please check this into version control
```

If the `cargo sqlx prepare` command completes successfully, you can run the program:

```
$ cargo run
1: John Doe, 1000
2: Ivan Ivanov, 2000
1: John Doe, 1000
```

***

The `query_scalar!` macro works on the same principle:

```rust,noplayground
use sqlx::{postgres::PgPoolOptions};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let account_ids: Vec<i64> = sqlx::query_scalar!("SELECT id FROM accounts")
        .fetch_all(&pool).await.unwrap();
    println!("All IDs: {account_ids:?}");

    let accounts_count: Option<i64> = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM accounts"
        )
        .fetch_one(&pool).await.unwrap();
    println!("Number of accounts: {}", accounts_count.unwrap_or_default());
}
```

After running `cargo sqlx prepare`, run the application:

```
$ cargo run
All IDs: [1, 2]
Number of accounts: 2
```

***

And the `query!` macro, as you might have guessed, also follows the same scheme:

```rust,noplayground
use sqlx::{postgres::{PgPoolOptions, PgQueryResult}, types::BigDecimal};

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .connect("postgres://postgres:1111@localhost/mydb").await.unwrap();

    let result: PgQueryResult = sqlx::query!(
            "INSERT INTO accounts(owner_name, balance) VALUES($1, $2)",
            "Some Name",
            BigDecimal::default(),
        )
        .execute(&pool).await.unwrap();
    println!("{result:?}"); // PgQueryResult { rows_affected: 1 }
}
```

After running `cargo sqlx prepare`, run the application:

```
$ cargo run
PgQueryResult { rows_affected: 1 }
```

## query* Functions vs. query*! Macros

After reading this chapter, you likely have the question: "When should I use macros, and when should I use regular functions?"

A common opinion is that functions should be used only in situations where SQL is constructed dynamically during program execution, while macros should be used in all other cases.

In practice, macros are not an absolute defense against incorrect SQL, as the database structure against which you ran `cargo sqlx prepare` might, due to various circumstances, turn out to be different from the actual production database. Therefore, it is primarily important to cover all database functionality with integration tests.

It should also be noted that macros provide no advantage in terms of execution speed, as they compile into the same structures created by their function counterparts.

