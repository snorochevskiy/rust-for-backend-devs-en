# Package, Crate, Module

Now that we know a single project can contain both a library and multiple executable files, we can finally give an informed answer to the question: "What is a crate?"

It is simple: a **crate** is a unit of compilation. Considering how the Rust compiler handles modules — first by including the modules and then compiling one large file — it can be said that a crate is either a library or an executable file.

To summarize:

A **package** is the project created using the `cargo new` command. The main section in the `Cargo.toml` file is even titled `[package]`.

A package contains one or more **crates**, which correspond to the `lib.rs` file and the main files for executable programs: `src/main.rs` and `src/bin/*.rs`.

We are already well-acquainted with **modules** from the [Modules](../rust-basics/modules.md) chapter. These are simply blocks of Rust code wrapped in a `mod module_name {}` section, or source code files intended to be included in other files during the crate assembly.
