---
title: Diesel – Type-safe SQL
author: Pascal Hertleif
date: 2017-03-01
---

## Pascal Hertleif

- Web stuff by day, Rust enthusiast by night!
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

# Query Builder

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

- `todos.select(title)`
- `todos.filter(done.eq(false))`
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

# Traits Everywhere {#traits}

Expression & AsExpression

## Using rustdoc to Create Mazes

**[docs.diesel.rs](http://docs.diesel.rs)**

Protip: Just search for what you think it should be called

# A Duality of Types {#mapping-types}

- SQL types
- Rust types

and how to convert between them

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

# The Amazing `infer_schema!` {#infer_schema}

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

# Associations

# Diesel CLI Tool {#cli}

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

# Conclusion

- Try diesel tonight!

# Thank you!

# Questions?

---

Slides will be available at [](http://killercup.github.io/)
