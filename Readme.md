---
title: Diesel – Type-safe SQL
author: Pascal Hertleif
date: 2017-03-01
---

## Pascal Hertleif

- Web stuff by day, Rust by night!
- Coorganizer of [Rust Cologne]
- {[twitter],[github]}.com/killercup

[Rust Cologne]: http://rust.cologne/
[twitter]: https://twitter.com/killercup
[github]: https://github.com/killercup

# First off

## [diesel.rs](http://diesel.rs)

## Requires Rust 1.15

## Supported Backends

- Postgres
- SQLite
- MySQL

# Getting started

---

```sh
$ cargo new --bin diesel-example; cd diesel-example
$ echo "DATABASE_URL=test.db" > .env
```

Add to `Cargo.toml`:

```toml
[dependencies]
diesel = { version = "0.10.1", features = ["sqlite"] }
diesel_codegen = { version = "0.10.1", features = ["sqlite"] }
dotenv = "0.8.0"
```

---

And you are good to go!

# Query Builder

---

Write queries in Rust

Invalid queries are compile-time errors

## Basic Usage

```rust
let still_todo: Vec<Todo> =
    todos
    .filter(done.eq(false))
    .limit(100)
    .load(&connection)
    .unwrap();
```

---

Not shown:

```rust
#[derive(Queryable)]
struct Todo { title: String, done: bool };

pub mod schema {
    infer_schema!("dotenv:DATABASE_URL");
}

use schema::todos::dsl::{todos, done};
```

---

- [Tell me more about this DSL](#dsl)
- [What kind of sorcery is `infer_schema!`?!](#infer_schema)

# DSL {#dsl}

## Idea

1. Create zero-sized structs for table and its columns
2. Implement traits on these structs
3. ???
4. PROFIT!

## Intuitive Methods

- `todos.select((id, title))`
- `.filter(done.eq(false))`
- `.limit(100)`

## Each method returns a new, nested type.

`todos.filter(done.eq(false)).limit(100)`

```rust
SelectStatement<
    (Integer, Text, Bool),
    (todos::id, todos::title, todos::done),
    todos::table,
    NoDistinctClause,
    WhereClause<Eq<todos::done, Bound<Bool, bool>>>,
    NoOrderClause,
    LimitClause<Bound<BigInt, i64>>
>
```

---

- [That sounds like a lot of boilerplate…](#table)
- [Is this what makes it fast?](#perf)

# Traits for Everything {#traits}

---

Diesel has `i32::MAX` traits in its codebase last time I counted

---

Add methods to methods to query builder? Write a trait like [`FilterDsl`](http://docs.diesel.rs/diesel/prelude/trait.FilterDsl.html)!

And implement generically:

```rust
impl<T, Predicate, ST> FilterDsl<Predicate> for T
```

---

Well, actually...

```rust
impl<T, Predicate, ST> FilterDsl<Predicate> for T where
  Predicate: Expression<SqlType=Bool> + NonAggregate,
  FilteredQuerySource<T, Predicate>: AsQuery<SqlType=ST>,
  T: AsQuery<SqlType=ST> + NotFiltered
```

(Those constraints are all traits as well!)

## Using rustdoc to Create Mazes

**[docs.diesel.rs](http://docs.diesel.rs)**

Protip: Just search for what you think it should be called

# A Duality of Types {#mapping-types}

---

There are

- SQL types
- Rust types

and ways to convert between them

---

- Your schema contains SQL types
- Your application's structs contain Rust types

---

- `FromSql`, `ToSql`
- Can map 1:1 like [`Float`](http://docs.diesel.rs/diesel/types/struct.Float.html)
- Or not, like [`Timestamp`](http://docs.diesel.rs/diesel/types/struct.Timestamp.html)


# The `table!` Macro {#table}

---

```rust
table! {
    todos (id) {
        id -> Integer,
        title -> Text,
        done -> Bool,
    }
}
```

---

[Let's see what we end up with](file:///Users/pascal/Projekte/diesel-demo/target/doc/diesel_demo/schema/todos/index.html)

---

- [Those don't look like Rust types](#mapping-types)
- [Do I have to type that?](#infer_schema)

# The Amazing Schema Inference {#infer_schema}

---

Never let your code and databa schema diverge!

`infer_schema!("dotenv:DATABASE_URL");`

---

It bascially generates the [`table!`](#table) macro calls for you.

---

- You need to have a database running on your dev machine
- The schema needs to be the same as in production

---

`diesel print-schema`

prints the `table!` macro calls `infer_schema!` generates

(So you can e.g. put it in version control)

---

- [How do you manage migrations](#migrations)
- [Oh, what's that `diesel` tool?](#cli)

# Derive ALL the Traits

---

Did I tell you about our Lord and Savior, Macros 1.1?

---

```rust
#[macro_use] extern crate diesel;
#[macro_use] extern crate diesel_codegen;

#[derive(Debug, Queryable)]
struct Todo {
    id: i32,
    title: String,
    done: bool,
}
```

It Just Works™

---

> Diesel Codegen provides custom derive implementations for
> [`Queryable`][queryable], [`Identifiable`][identifiable],
> [`Insertable`][insertable], [`AsChangeset`][as-changeset], and [`Associations`][associations].
>
> It also provides the macros [`infer_schema!`][infer-schema],
> [`infer_table_from_schema!`][infer-table-from-schema], and
> [`embed_migrations!`][embed-migrations].

[queryable]: http://docs.diesel.rs/diesel/query_source/trait.Queryable.html
[identifiable]: http://docs.diesel.rs/diesel/associations/trait.Identifiable.html
[insertable]: http://docs.diesel.rs/diesel/prelude/trait.Insertable.html
[as-changeset]: http://docs.diesel.rs/diesel/query_builder/trait.AsChangeset.html
[associations]: http://docs.diesel.rs/diesel/associations/index.html
[infer-schema]: http://docs.diesel.rs/diesel/macro.infer_schema!.html
[infer-table-from-schema]: http://docs.diesel.rs/diesel/macro.infer_table_from_schema!.html
[embed-migrations]: http://docs.diesel.rs/diesel/macro.embed_migrations!.html

---


# Associations

---

```rust
#[derive(Identifiable, Queryable, Associations)]
#[has_many(posts)]
pub struct User { id: i32, name: String, }

#[derive(Identifiable, Queryable, Associations)]
#[belongs_to(User)]
pub struct Post { id: i32, user_id: i32, title: String, }
```

---

---

```rust
let user = try!(users::find(1).first(&connection));
let posts = Post::belonging_to(&user).load(&connection);
```

---

Read much more about this at [docs.diesel.rs/diesel/associations/](http://docs.diesel.rs/diesel/associations/index.html)

---

# Diesel CLI Tool {#cli}

---

Install it with

```sh
$ cargo install diesel
```

---

This makes it easy to

- Setup your database
- Manage your [migrations](#migrations)
- Print your schema

---

- [Tell me more about migrations](#migrations)
- [What was that about schema printing?](#infer_schema)


# Migrations {#migrations}

---

`migrations/datetime-name/{up,down}.sql`

Simple SQL files that change your database schema (e.g. `CREATE TABLE`, `DROP TABLE`)

---

- `diesel migration generate create_todos`
- `diesel migration run`
- `diesel migration revert`

---

- [Oh, what's that `diesel` tool?](#cli)

# Performance {#perf}

---

- Almost all queries can be represented by unique types
- Each of these types returns a query (that uses bind params)
- Let's cache these queries as Prepared Statements!

---

- Is it faster than `c`? [Yes you can use Diesel in warp drives](https://hackernoon.com/comparing-diesel-and-rust-postgres-97fd8c656fdd#.kdof0mold)

# Type System Shenanigans

---

Macros to implement traits for generic tuples of up to 52 elements

---

Enable query builder features depending on the used backend

- Basically every type and trait is generic over the backend
- E.g.: Only Postgres implementes `RETURNING` clause


# Testing Diesel

## Unit tests

For helper/converter functions

- `unix_epoch_decodes_correctly_with_timezone`
- `queries_with_different_types_have_different_ids`

## Integration tests

Using diesel like a library

## Doc test

Examples in the API documentation are tests!

---

Secret sauce: `include!("src/doctest_setup.rs");`

## Quickcheck

Test roundtrips from Rust → DB → Rust

(With lots of macros, of course)

[Source](https://github.com/diesel-rs/diesel/blob/6a6f5835a40efd515dfc774d4b1d335cc87dd4da/diesel_tests/tests/types_roundtrip.rs)

## Compile-fail Tests

Invalid queries should not compile

So let's test that they return the expected errors!

---

The [compiletest](https://github.com/laumann/compiletest-rs) tool is also used by the Rust compiler and Clippy

# Thank you for listening!

- Try diesel tonight!
- Read the docs at [diesel.rs](http://diesel.rs/)
- Get help at [gitter.im/diesel-rs/diesel](https://gitter.im/diesel-rs/diesel)

# Questions?

---

Slides are available at [git.io/diesel-adventure](https://git.io/diesel-adventure)

License: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

