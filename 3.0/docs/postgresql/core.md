# PostgreSQL Core

PostgreSQL ([vapor/postgresql](https://github.com/vapor/postgresql)) is a pure-Swift (no `libpq` dependency), event-driven, non-blocking driver for PostgreSQL. It's built on top of the [SwiftNIO](http://github.com/apple/swift-nio) networking library.

!!! seealso
    The higher-level, Fluent PostgreSQL ORM guide is located at [PostgreSQL &rarr; Fluent](fluent.md)

Using just the PostgreSQL package for your project may be a good idea if any of the following are true.

- You have an existing DB with non-standard structure.
- You rely heavily on custom or complex SQL queries.
- You just plain don't like ORMs.

PostgreSQL core is built on top of DatabaseKit which provides some conveniences like connection pooling and integrations with Vapor's [Services](../getting-started/services.md) architecture.

!!! tip
    Even if you do choose to use [Fluent PostgreSQL](fluent.md), all of the features of PostgreSQL core will be available to you.

## Getting Started

Let's take a look at how you can get started using PostgreSQL core.

### Package

The first step to using PostgreSQL core is adding it as a dependency to your project in your SPM package manifest file.

```swift
// swift-tools-version:4.0
import PackageDescription

let package = Package(
    name: "MyApp",
    dependencies: [
        /// Any other dependencies ...
        
        // 🐘 Non-blocking, event-driven Swift client for PostgreSQL.
        .package(url: "https://github.com/vapor/postgresql.git", from: "1.0.0-rc"),
    ],
    targets: [
        .target(name: "App", dependencies: ["PostgreSQL", ...]),
        .target(name: "Run", dependencies: ["App"]),
        .testTarget(name: "AppTests", dependencies: ["App"]),
    ]
)
```

Don't forget to add the module as a dependency in the `targets` array. Once you have added the dependency, regenerate your Xcode project with the following command:

```sh
vapor xcode
```


### Config

The next step is to configure the database in your [`configure.swift`](../getting-started/structure.md#configureswift) file.

```swift
import PostgreSQL

/// ...

/// Register providers first
try services.register(PostgreSQLProvider())
```

Registering the provider will add all of the services required for PostgreSQL to work properly. It also includes a default database config struct that uses typical development environment credentials. 

You can of course override this config struct if you have non-standard credentials.

```swift
/// Register custom PostgreSQL Config
let psqlConfig = PostgreSQLDatabaseConfig(hostname: "localhost", port: 5432, username: "vapor")
services.register(psqlConfig)
```

### Query

Now that the database is configured, you can make your first query.

```swift
router.get("psql-version") { req -> Future<String> in
    return req.withPooledConnection(to: .psql) { conn in
        return try conn.query("select version() as v;").map(to: String.self) { rows in
            return try rows[0].firstValue(forColumn: "v")?.decode(String.self) ?? "n/a"
        }
    }
}
```

Visiting this route should display your PostgreSQL version.

## Connection

A `PostgreSQLConnection` is normally created using the `Request` container and can perform two different types of queries.

### Create

There are two methods for creating a `PostgreSQLConnection`.

```swift
return req.withPooledConnection(to: .psql) { conn in
    /// ...
}
return req.withConnection(to: .psql) { conn in
    /// ...
}
```

As the names imply,  `withPooledConnection(to:)` utilizes a connection pool. `withConnection(to:)` does not. Connection pooling is a great way to ensure your application does not exceed the limits of your database, even under peak load.

### Simply Query

Use `.simpleQuery(_:)` to perform a query on your PostgreSQL database that does not bind any parameters. Some queries you send to PostgreSQL may actually require that you use the `simpleQuery(_:)` method instead of the parameterized method. 

!!! note
    This method sends and receives data as text-encoded, meaning it is not optimal for transmitting things like integers.
    
```swift
let rows = req.withPooledConnection(to: .psql) { conn in
    return conn.simpleQuery("SELECT * FROM users;")
}
print(rows) // Future<[[PostgreSQLColumn: PostgreSQLData]]>
```

You can also choose to receive each row in a callback, which is great for conserving memory for large queries.

```swift
let done = req.withPooledConnection(to: .psql) { conn in
    return conn.simpleQuery("SELECT * FROM users;") { row in
        print(row) // [PostgreSQLColumn: PostgreSQLData]
    }
}
print(done) // Future<Void>
```

### Parameterized Query

PostgreSQL also supports sending parameterized queries (sometimes called prepared statements). This method allows you to insert data placeholders into the SQL string and send the values separately.

Data sent via parameterized queries is binary encoded, making it more efficient for sending some data types. In general, you should use parameterized queries where ever possible.

```swift
let users = req.withPooledConnection(to: .psql) { conn in
    return try conn.query("SELECT *  users WHERE name = $1;", ["Vapor"])
}
print(users) // Future<[[PostgreSQLColumn: PostgreSQLData]]>
```

You can also provide a callback, similar to simple queries, for handling each row individually.




