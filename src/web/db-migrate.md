# Database Schema Versioning

SQLx provides a command-line utility called [sqlx-cli](https://crates.io/crates/sqlx-cli), which allows you to:

* Read metadata from the database and validate SQL queries at compile time.
* Quickly create and drop test databases.
* Version database structures, a process often referred to as **migration**.

In this chapter, we will focus specifically on database schema versioning.

First, you need to install the utility using the following command:

`cargo install sqlx-cli`

Now, let's move our SQL script for creating tables (from the previous chapter) into migration files.

Create the first migration file using this command:

```
cargo sqlx migrate add accounts -r --sequential
```

This command generates two empty files:

* `migrations/0001_accounts.up.sql` — the file for applying changes to the database structure.
* `migrations/0001_accounts.down.sql` — the file for rolling back those changes.

Open `0001_accounts.up.sql` and add the following:

```sql
CREATE SEQUENCE accounts_seq START WITH 1000;

CREATE TABLE accounts ( -- mydb.public.accounts 
    id BIGINT PRIMARY KEY DEFAULT nextval('accounts_seq'),
    owner_name VARCHAR(255) NOT NULL UNIQUE,
    balance NUMERIC(10, 2)  NOT NULL DEFAULT 0.00 CHECK (balance >= 0)
);
```

Then, fill `0001_accounts.down.sql` with:

```sql
DROP TABLE accounts;
DROP SEQUENCE accounts_seq;
```

Next, we’ll create a second pair of files for the `transactions` table.

You can do this using the command `cargo sqlx migrate add transactions -r --sequential`, or you can simply create the files `0002_transactions.up.sql` and `0002_transactions.down.sql` manually.

Fill `0002_transactions.up.sql` with:

```sql
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
```

And `0002_transactions.down.sql`:

```sql
DROP TABLE transactions;
DROP SEQUENCE transactions_seq;
```

To test our migrations, let's create a new database called my_migration.

```sql
CREATE DATABASE my_migration;
```

Once the database is created, run the migration:

```
$ cargo sqlx migrate run \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 1/migrate accounts (3.99736ms)
Applied 2/migrate transactions (4.774571ms)
```

Three tables should now appear in your database:

* `accounts`
* `transactions`
* `_sqlx_migrations`

The `_sqlx_migrations` table is an internal SQLx table that stores metadata about applied migrations. The migration mechanism uses this table to track which scripts have already been executed. It stores the version numbers, the names of the corresponding migration files, execution timestamps, and a checksum of the file (to ensure that an already applied migration file hasn't been modified "after the fact").

```
> select * from _sqlx_migrations;
 version | description  |  installed_on       | success | checksum | execution_time
---------+------------- +---------------------+---------+---------------------------
       1 | accounts     | 2025-12-12 02:19:32 | t       | \x3124df | 3997360
       2 | transactions | 2025-12-12 02:19:32 | t       | \xaded4e | 4774571
```

As we can see, both migrations were successfully applied.

The `cargo sqlx migrate` utility allows you not only to apply migrations but also to roll them back. For example, let's revert the database structure to version 1 — the state where the `accounts` table exists, but the `transactions` table has not yet been created.

```
$ cargo sqlx migrate revert \
    --target-version 1 \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 2/revert transactions (3.658186ms)
Skipped 1/revert accounts (0ns)
```

If you check the tables in the `my_migration` database now, you will see that the `transactions` table is gone. If you run the migration again, the `transactions` table will be recreated.

```
$ cargo sqlx migrate run \
    --database-url postgres://postgres:1111@localhost/my_migration

Applied 2/migrate transactions
```

