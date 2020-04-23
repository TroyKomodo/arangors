<!-- cargo-sync-readme start -->

# arangors

[![Build Status](https://travis-ci.org/fMeow/arangors.svg?branch=master)](https://travis-ci.org/fMeow/arangors)
[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![Crates.io](https://img.shields.io/crates/v/arangors.svg)](https://crates.io/crates/arangors)
[![arangors](https://docs.rs/arangors/badge.svg)](https://docs.rs/arangors)

`arangors` is an intuitive rust client for [arangoDB](https://www.arangodb.com/),
inspired by [pyArango](https://github.com/tariqdaouda/pyArango).

`arangors` enables you to connect with arangoDB server, access to database,
execute AQL query, manage arangoDB in an easy and intuitive way,
both async and plain synchrnous code with any HTTP ecosystem you love.

## Philosophy of arangors

`arangors` is targeted at ergonomic, intuitive and OOP-like API for
ArangoDB, both top level and low level API for users' choice.

Overall architecture of arangoDB:

> databases -> collections -> documents/edges

In fact, the design of `arangors` just mimic this architecture, with a
slight difference that in the top level, there is a connection object on top
of databases, containing a HTTP client with authentication information in
HTTP headers.

Hierarchy of arangors:
> connection -> databases -> collections -> documents/edges

## Features

By now, the available features of arangors are:

- make connection to arangoDB
- get list of databases and collections
- fetch database and collection info
- create and delete dababase or collections
- full featured AQL query

## TODO

- (Done) Milestone 0.1.x

    Synchronous connection based on `reqwest` and full featured AQL query.

- (X) Milestone 0.2.x

    Fill the unimplemented API in `Connection`, `Database`, `Collection` and
    `Document`.

    ~~In this stage, all operations available for database, collection and
    document should be implemented.~~

    Well, I am too lazy to fill all API, as the AQL syntax suffices in most
cases.     Let's fullill this goal in 0.4.x .

- (Done) Milestone 0.3.x

    Implement both sync and async client. Also, offers a way to use custom
HTTP client ecosystem.

- (WIP) Milestone 0.4.x

    Provides the API related to graph, index and user management.

    In this stage, all operations available for database, collection and
    document should be implemented.

## Glance

### Use Different HTTP Ecosystem, Regardless of Async or Sync

You can switch to different HTTP ecosystem with a feature gate, or implement the Client yourself(
see examples).

Currently out-of-box supported ecosystem are:
- `reqwest_async`
- `reqwest_blocking`
- `surf_async`

By default, `arangors` use `reqwest_async` as underling HTTP Client to connect with ArangoDB.
You can switch other ecosystem in feature gate:

```toml
[dependencies]
arangors = { version = "0.3", features = ["surf_async"], no-default-features = true }
```

Or if you want to stick with other ecosystem that are not listed in the feature gate,
you can get vanilla `arangors` without any HTTP client dependecies:

```toml
[dependencies]
// This one is async
arangors = { version = "0.3", no-default-features = true }
// This one is synchronous
arangors = { version = "0.3", features = ["blocking"], no-default-features = true }
```

Thanks to `maybe_async`, `arangors` can unify sync and async API and toggle with a
feature gate. Arangors adopts async first policy.

### Connection

There is three way to establish connections:

- jwt
- basic auth
- no authentication

So are the `arangors` API.

When a connection is successfully established,
`arangors` will automatically fetch the structure of arangoDB
by get the list of database, and then lists of collections per database.

Example:

- With authentication

```rust
use arangors::Connection;

// (Recommended) Handy functions
let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let conn = Connection::establish_basic_auth("http://localhost:8529", "username", "password")
    .await
    .unwrap();
```

- Without authentication, only use in evaluation setting

``` rust, ignore
let conn = Connection::establish_without_auth("http://localhost:8529").await.unwrap();
```

### Database && Collection

```rust
use arangors::Connection;

let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let db = conn.db("test_db").await.unwrap();
let collection = db.collection("test_collection").await.unwrap();
```

### AQL Query

All aql query related functions are associated with database, as AQL query
is performed at database level.

There are several way to execute AQL query, and can be categorized into two
classes:

- batch query

    - `aql_query_batch`
    - `aql_next_batch`

- query to fetch all results

    - `aql_str`
    - `aql_bind_vars`
    - `aql_query`

This later category provides a convenient high level API, whereas batch
query offers more control.

#### Typed or Not Typed

Note that results from arangoDB server, e.x. fetched documents, can be
strong typed given deserializable struct, or arbitrary JSON object with
`serde::Value`.

- Not Typed: Arbitrary JSON object

```rust
use arangors::Connection;
use serde_json::Value;

let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let db = conn.db("test_db").await.unwrap();
let resp: Vec<Value> = db
    .aql_str("FOR u IN test_collection LIMIT 3 RETURN u")
    .await
    .unwrap();
```

- Strong Typed

```rust
use arangors::Connection;
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct User {
    pub username: String,
    pub password: String,
}

let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let db = conn.db("test_db").await.unwrap();
let resp: Vec<User> = db
    .aql_str("FOR u IN test_collection RETURN u")
    .await
    .unwrap();
```

#### Batch query

`arangors` offers a way to manually handle batch query.

Use `aql_query_batch` to get a cursor, and use `aql_next_batch` to fetch
next batch and update cursor with the cursor.

```rust

let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let db = conn.db("test_db").await.unwrap();

let aql = AqlQuery::new("FOR u IN @@collection LIMIT 3 RETURN u")
    .batch_size(1)
    .count(true)
    .bind_var("@collection", "test_collection");

// fetch the first cursor
let mut cursor = db.aql_query_batch(aql).await.unwrap();
// see metadata in cursor
println!("count: {:?}", cursor.count);
println!("cached: {}", cursor.cached);
let mut results: Vec<serde_json::Value> = Vec::new();
loop {
    if cursor.more {
        let id = cursor.id.unwrap().clone();
        // save data
        results.extend(cursor.result.into_iter());
        // update cursor
        cursor = db.aql_next_batch(id.as_str()).await.unwrap();
    } else {
        break;
    }
}
println!("{:?}", results);
```

#### Fetch All Results

There are three functions for AQL query that fetch all results from
ArangoDB. These functions internally fetch batch results one after another
to get all results.

The functions for fetching all results are listed as bellow:

- `aql_str`

This function only accept a AQL query string.

Here is an example of strong typed query result with `aql_str`:

```rust
use arangors::Connection;
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct User {
    pub username: String,
    pub password: String,
}

let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let db = conn.db("test_db").await.unwrap();
let result: Vec<User> = db
    .aql_str(r#"FOR i in test_collection FILTER i.username=="test2" return i"#)
    .await
    .unwrap();
```

- `aql_bind_vars`

This function can be used to start a AQL query with bind variables.

```rust
use arangors::{Connection, Document};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    pub username: String,
    pub password: String,
}

let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let db = conn.db("test_db").await.unwrap();

let mut vars = HashMap::new();
let user = User {
    username: "test".to_string(),
    password: "test_pwd".to_string(),
};
vars.insert("user", serde_json::value::to_value(&user).unwrap());
let result: Vec<Document<User>> = db
    .aql_bind_vars(r#"FOR i in test_collection FILTER i==@user return i"#, vars)
    .await
    .unwrap();
```

- `aql_query`

This function offers all the options available to tweak a AQL query.
Users have to construct a `AqlQuery` object first. And `AqlQuery` offer all
the options needed to tweak AQL query. You can set batch size, add bind
vars, limit memory, and all others
options available.

```rust
use arangors::{AqlQuery, Connection, Cursor, Database};
use serde_json::value::Value;

let conn = Connection::establish_jwt("http://localhost:8529", "username", "password")
    .await
    .unwrap();
let database = conn.db("test_db").await.unwrap();

let aql = AqlQuery::new("FOR u IN @@collection LIMIT 3 RETURN u")
    .batch_size(1)
    .count(true)
    .bind_var("@collection", "test_collection");

let resp: Vec<Value> = database.aql_query(aql).await.unwrap();
println!("{:?}", resp);
```

### Contributing

Contributions and feed back are welcome following Github workflow.

### License

`arangors` is provided under the MIT license. See [LICENSE](./LICENSE).
An ergonomic [arangoDB](https://www.arangodb.com/) client for rust.

<!-- cargo-sync-readme end -->
