Getting Started with GRDB
=========================

<img align="right" src="https://github.com/groue/GRDB.swift/raw/master/Documentation/DemoApps/GRDBDemoiOS/Screenshot.png" width="50%">

**This tutorial describes the building of the [UIKit demo application], step by step, applying best SQLite and GRDB practices along the way.** In the end, you will have a database access layer that fulfills the application needs, and can be tested.

The target audience is already fluent with the creation of an Xcode project, storyboards, or view controllers: these topics are not covered.

Basic database knowledge is assumed, such as what tables and columns are. The tutorial covers high-level GRDB features (record types, query interface, database observation). For all topics, a "raw SQL" version is also provided, so that you can choose your favorite way to deal with SQLite.

The tutorial explains its design choices: when you want an explanation about some particular piece of code, expand the notes marked with an ℹ️.

As you can see in the [screenshot](https://github.com/groue/GRDB.swift/raw/master/Documentation/DemoApps/GRDBDemoiOS/Screenshot.png), the demo application displays the list of players stored in the database. The application user can sort players by name or by score. She can add, edit, and delete players. The list of players can be "refreshed". For demo purpose, refreshing players performs random modifications to the players.

Let's start!

- [The Database Service](#the-database-service)
- [The Shared Application Database](#the-shared-application-database)
- [The Database Schema](#the-database-schema)
- [Inserting Players in the Database, and the Player Struct](#inserting-players-in-the-database-and-the-player-struct)
- [Deleting Players](#deleting-players)
- [Testing the Database](#testing-the-database)

## The Database Service

In this chapter, we introduce the `AppDatabase` service. It is the class that grants access to the player database, in a controlled fashion.

We'll make it possible to fetch the list of players, insert new players, as well as other application needs. But not all database operations will be possible. For example, setting up the database schema is the strict privilege of `AppDatabase`, not of the rest of the application.

The `AppDatabase` service accesses the SQLite database through a GRDB [database connection]. We'd like the application to use a `DatabasePool`, because this connection leverages the advantages of the SQLite [WAL mode]. On the other side, we'd prefer application tests to run as fast as possible, with an in-memory database provided by a `DatabaseQueue`.

Pools and queues share a common protocol, `DatabaseWriter`, and this is what our `AppDatabase` service needs:

```swift
// File: AppDatabase.swift
import GRDB

final class AppDatabase {
    /// Creates an `AppDatabase`.
    init(_ dbWriter: DatabaseWriter) throws {
        self.dbWriter = dbWriter
    }
    
    /// Provides access to the database.
    private let dbWriter: DatabaseWriter
}
```

<details>
    <summary>ℹ️ Design Notes</summary>

> The `dbWriter` property is private: this allows `AppDatabase` to restrict the operations that can be performed on the database.
> 
> The initializer is not private: we can freely create `AppDatabase` instances, for the application, and for tests.
> 
> The initializer is declared with the `throws` qualifier, because it will be extended, below in this guide, in order to prepare the database for application use.

</details>

> ✅ At this stage, we have a `AppDatabase` class which encapsulates access to the database. It supports both WAL databases, and in-memory databases, so that it can feed both the application, and tests.

## The Shared Application Database

Our app uses a single database file, so we want a "shared" database.

Inspired by `UIApplication.shared`, `UserDefaults.standard`, or `FileManager.default`, we will define `AppDatabase.shared`.

<details>
    <summary>ℹ️ Design Notes</summary>

> Some applications will prefer to manage the shared `AppDatabase` instance differently, for example with some dependency injection technique. In this case, you will not define `AppDatabase.shared`.
> 
> Just make sure that there exists a single instance of `DatabaseQueue` or `DatabasePool` for any given database file. This is because multiple instances would compete for database access, and sometimes throw errors. [Sharing a database] is hard. Get inspiration from `AppDatabase.makeShared()`, below, in order to create the single instance of your database service.

</details>

The shared `AppDatabase` instance opens the database on the file system, in a file named "db.sqlite" stored in a "database" folder, and creates an empty database if the file does not already exist:

```swift
// File: Persistence.swift
import GRDB

extension AppDatabase {
    /// The database for the application
    static let shared = makeShared()
    
    private static func makeShared() -> AppDatabase {
        do {
            // Create a folder for storing the SQLite database
            let fileManager = FileManager()
            let folderURL = try fileManager
                .url(for: .applicationSupportDirectory, in: .userDomainMask, appropriateFor: nil, create: true)
                .appendingPathComponent("database", isDirectory: true)
            try fileManager.createDirectory(at: folderURL, withIntermediateDirectories: true)
            
            // Connect to a database on disk
            let dbURL = folderURL.appendingPathComponent("db.sqlite")
            let dbPool = try DatabasePool(path: dbURL.path)
            
            // Create the AppDatabase
            let appDatabase = try AppDatabase(dbPool)
            return appDatabase
        } catch {
            fatalError("Unresolved error \(error)")
        }
    }
}
```

<details>
    <summary>ℹ️ Design Notes</summary>

> The database is stored in its own directory, so that you can easily:
> 
> - Set up [data protection].
> - Remove the database as well as its companion [temporary files](https://sqlite.org/tempfiles.html) from disk in a single stroke.
> 
> The shared `AppDatabase` uses a `DatabasePool` in order to profit from the SQLite [WAL mode].
> 
> Any error which prevents the application from opening the database has the application crash. You will have to adapt this sample code if you intend to build an app that is able to run without a working database. For example, you could modify `AppDatabase` so that it owns a `Result<DatabaseWriter, Error>` instead of a plain `DatabaseWriter` - but your mileage may vary.

</details>

> ✅ At this stage, we have a `AppDatabase.shared` object which vends an empty database. We'll add methods and properties to `AppDatabase`, as we discover the needs of our application.


## The Database Schema

Now that we have an empty database, let's define its schema: the database table(s) that will store our application data. A good database schema will have SQLite manage the database integrity for you, and make sure it is impossible to store invalid data in the database: this is an important step!

<details>
    <summary>ℹ️ Design Notes</summary>

> Some database libraries derive the database schema and relational constraints right from application code. For example, the fact that the name of a player can't be nil would be expressed in Swift, and the database library would prevent nil names from entering the database. With such libraries, you may not be free to define the database schema as you would want it to be, and you do not have much guarantee about the quality of your data.
> 
> With GRDB, it is just the other way around: you freely define the database schema so that it fulfills your application needs, and you access the database data with Swift code that matches this schema. You can't build a safer haven for your precious users' data than a robust SQLite schema. Bring your database skills with you!

</details>

Our database has one table, `player`, where each row contains the attributes of a player: a unique identifier (aka *primary key*), a name, and a score. The identifier makes it possible to instruct the database to perform operations on a specific player. We'll make sure all players have a name and a score (we'll prevent *NULL* values from entering those columns).

In order to define the schema and create the `player` table, it is recommended to use [migrations]. All applications around us evolve as time passes, and ship several versions: it is likely our app will do the same. The database schema evolves as well, as we add features and fix bugs in our app. That's exactly what migrations are for: they represent the building steps of our database schema, as our app goes through its versions.

The migrations are defined in the `AppDatabase.migrator` property, in which we register the *initial migration*:

```swift
    private var migrator: DatabaseMigrator {
        var migrator = DatabaseMigrator()
        
        migrator.registerMigration("initial") { db in
            try db.create(table: "player") { t in
                t.autoIncrementedPrimaryKey("id")
                t.column("name", .text).notNull()
                t.column("score", .integer).notNull()
            }
        }
        
        // Migrations for future application versions will be inserted here:
        // migrator.registerMigration(...) { db in
        //     ...
        // }
        
        return migrator
    }
```

<details>
    <summary>Raw SQL version</summary>

Some readers like to write SQL. Please be welcome:

```swift
    private var migrator: DatabaseMigrator {
        var migrator = DatabaseMigrator()
        
        migrator.registerMigration("initial") { db in
            try db.execute(sql: """
                CREATE TABLE player (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    name TEXT NOT NULL,
                    score INTEGER NOT NULL
                )
                """)
        }
        
        return migrator
    }
```

</details>

<details>
    <summary>ℹ️ Design Notes</summary>

> The database table for players is named `player`, because GRDB recommends that table names are English, singular, and camel-cased (`player`, `country`, `postalAddress`, etc.) Such names will help you using [Associations] when you need them. Database table names that follow another naming convention are totally OK, but you will have to perform extra configuration.
> 
> The primary key for players is an auto-incremented column named `id`. It also could have been a UUID column named `uuid`. GRDB generally accepts all primary keys, even if they are not named `id`, even if they span several columns, without any extra setup. Yet `id` is a frequent convention.
> 
> The `id` column is [autoincremented](https://sqlite.org/autoinc.html), in order to avoid id reuse. Reused ids can trip up [database observation] tools: a deletion followed by an insertion with the same id may be interpreted as an update, with unintended consequences.

</details>

The migrations are now defined, but they are not applied yet. Let's modify the `AppDatabase` initializer, and *migrate* the database. It means that unapplied migrations are run. In the end, we are sure that the database schema is exactly what we want it to be:

```swift
    /// Creates an `AppDatabase` from a database connection,
    /// and make sure the database schema is ready.
    init(_ dbWriter: DatabaseWriter) throws {
        self.dbWriter = dbWriter
        try migrator.migrate(dbWriter)
    }
```

> ✅ At this stage, we have a `AppDatabase.shared` object which vends a database that contains a `player` table.

## Inserting Players in the Database, and the Player Struct

The `player` table can't remain empty, or the application will never display anything!

The application will insert players through the `AppDatabase` service. We could define an `AppDatabase.insertPlayer(name:score:)` method. But this does not scale well with the number of player columns: who wants to call a method with a dozen arguments?

Instead, let's define a `Player` struct that groups all player attributes:

```swift
// File: Player.swift

/// A Player (take one)
struct Player {
    var name: String
    var score: Int
}
```

This is enough to define the `AppDatabase.insertPlayer(_:)` method. But let's go further: the app will, eventually, deal with player identifiers. So let's add a `Player.id` property right away:

```swift
/// A Player
struct Player {
    var id: Int64?
    var name: String
    var score: Int
}
```

The type of the `id` property is `Int64?` because it matches the `id` column in the `player` table (SQLite integer primary keys are 64-bit integers, even on 32-bit platforms). When the id is nil, the player is not yet saved in the database. When the id is not nil, it is the identifier of a player in the database.

The `name` and `score` properties are regular `String` and `Int`. They are not optional (`String?` or `Int?`), because we added "not null" constraints on those database columns when we defined the `player` table.

> ✅ Now we have a `Player` type that fits the columns of the `player` database table, and is able to deal with all application needs. This is what GRDB calls a [record type]. At this stage, `Player` has no database power yet, but hold on.

The `Player` type allows `AppDatabase` to insert a player. We can do it with raw SQL:

<details>
    <summary>Raw SQL version</summary>

```swift
// File: AppDatabase.swift

extension AppDatabase {
    /// Inserts a player. When the method returns, the
    /// player id is set to the newly inserted id. 
    func insertPlayer(_ player: inout Player) throws {
        try dbWriter.write { db in
            try db.execute(literal: """
                INSERT INTO player (name, score) 
                VALUES (\(player.name), \(player.score))
                """)
            player.id = db.lastInsertedRowID
        }
    }
}
```

Note that the `execute(literal:)` method above avoids [SQL injection], thanks to [SQL Interpolation].

</details>

SQL is just fine, but we can also make `Player` a [persistable record]. With persistable records, you do not have to write the SQL queries that perform common persistence operations.

The `Player` struct adopts the `MutablePersistableRecord` protocol, because the `player` database table has an autoincremented id. "Persistable": players can be inserted, updated and deleted. "Mutable": inserting a player modifies it by setting its id.

Conformance to `MutablePersistableRecord` is almost free for types that adopt the standard [Codable] protocol:

```swift
// File: Player.swift

// Give Player database powers
extension Player: Encodable, MutablePersistableRecord {
    /// Updates a player id after it has been inserted in the database.
    mutating func didInsert(with rowID: Int64, for column: String?) {
        id = rowID
    }
}
```

In this tutorial, the name of the database table matches the name of the type, and columns matches properties, so we do not have to configure anything.

<details>
    <summary>Avoiding Encodable</summary>

The `Encodable` protocol is handy, but you may prefer not to use it. In this case, you have to fulfill the persistable record requirements:

```swift
// File: Player.swift

// Give Player database powers
extension Player: MutablePersistableRecord {
    /// Defines the values persisted in the database
    func encode(to container: inout PersistenceContainer) {
        container["id"] = id
        container["name"] = name
        container["score"] = score
    }
    
    /// Updates a player id after it has been inserted in the database.
    mutating func didInsert(with rowID: Int64, for column: String?) {
        id = rowID
    }
}
```

</details>

That's it. And thanks to persistable records, we now support both inserts and updates:

```swift
// File: AppDatabase.swift

extension AppDatabase {
    /// Saves (inserts or updates) a player. When the method returns, the
    /// player id is not nil.
    func savePlayer(_ player: inout Player) throws {
        try dbWriter.write { db in
            try player.save(db)
        }
    }
}
```

Now, we can save players from the app:

```swift
// Inserts a player
var player = Player(id: nil, name: "Arthur", score: 0)
try AppDatabase.shared.savePlayer(player)
print("Player id is \(player.id)") // Prints "Player id is 1"

// Updates a player
player.score = 1000
try AppDatabase.shared.savePlayer(player)
```

> 👆 **Note**: make sure you define the `Encodable` extension to `Player` in the same file where the `Player` struct is defined: this is how you will profit from the synthesized conformance to this protocol.
>
> ✅ At this stage, we have a `AppDatabase.shared` object which is able to insert and update players in the database.

## Deleting Players

The application can delete players:

- The "Trash" icon at the bottom left of the screen deletes all players.
- The "Edit" button at the top left of the screen lets the user delete individual players.
- The player list supports the swipe to delete gesture.

The `AppDatabase` service supports these use cases with two new methods:

```swift
// File: AppDatabase.swift

extension AppDatabase {
    /// Delete the specified players
    func deletePlayers(ids: [Int64]) throws {
        try dbWriter.write { db in
            _ = try Player.deleteAll(db, keys: ids)
        }
    }
    
    /// Delete all players
    func deleteAllPlayers() throws {
        try dbWriter.write { db in
            _ = try Player.deleteAll(db)
        }
    }
}
```

Both `deleteAll(_:)` and `deleteAll(_:keys:)` methods are available for all persistable records.

<details>
    <summary>Raw SQL version</summary>

If you do not want to make `Player` a persistable record, you can fallback to raw SQL:

```swift
// File: AppDatabase.swift

extension AppDatabase {
    /// Delete the specified players
    func deletePlayers(ids: [Int64]) throws {
        try dbWriter.write { db in
            try db.execute(literal: """
                DELETE FROM player WHERE id IN \(ids)
                """)
        }
    }
    
    /// Delete all players
    func deleteAllPlayers() throws {
        try dbWriter.write { db in
            try db.execute(sql: "DELETE FROM player")
        }
    }
}
```

The `deletePlayers(ids:)` method above uses [SQL Interpolation] so that you can embed an array of ids right inside your SQL query (`WHERE id IN \(ids)`). The "really raw" SQL version below is a little more involved:

```swift
    func deletePlayers(ids: [Int64]) throws {
        try dbWriter.write { db in
            if ids.isEmpty {
                // Avoid SQL syntax error
                return
            }
            // DELETE FROM player WHERE id IN (?, ?, ...)
            //                                 ~~~~~~~~~
            //   as many question marks as there are ids
            let placeholders = databaseQuestionMarks(count: ids.count)
            let query = "DELETE FROM player WHERE id IN (\(placeholders))"
            try db.execute(sql: query, arguments: StatementArguments(ids))
        }
    }
```

All the techniques we have seen avoid [SQL injection].

</details>

## Testing the Database

[UIKit demo application]: DemoApps/GRDBDemoiOS
[database connection]: https://github.com/groue/GRDB.swift/blob/master/README.md#database-connections
[DatabaseQueue]: ../README.md#database-queues
[DatabasePool]: ../README.md#database-pools
[migrations]: Migrations.md
[Codable]: https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types
[record type]: ../README.md#records
[persistable record]: ../README.md#persistablerecord-protocol
[WAL mode]: https://sqlite.org/wal.html
[data protection]: ../README.md#data-protection
[Associations]: AssociationsBasics.md
[Sharing a database]: SharingADatabase.md
[database observation]: ../README.md#database-changes-observation
[SQL Interpolation]: SQLInterpolation.md
[SQL injection]: ../README.md#avoiding-sql-injection