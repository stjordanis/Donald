# Donald

[![NuGet Version](https://img.shields.io/nuget/v/Donald.svg)](https://www.nuget.org/packages/Donald)
[![Build Status](https://travis-ci.org/pimbrouwers/Donald.svg?branch=master)](https://travis-ci.org/pimbrouwers/Donald)

Meet [Donald](https://en.wikipedia.org/wiki/Donald_D._Chamberlin). 

If you're a programmer and have used a database, he's impacted your life in a big way. 

This library is named after him.

> Honorable mention goes to [@dysme](https://github.com/dsyme) another important Donald and F#'s [BDFL](https://en.wikipedia.org/wiki/Benevolent_dictator_for_life).

## Key Features

Donald is a well-tested library, with pleasant ergonomics that aims to make working with [ADO.NET](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/ado-net-overview) *a lot more* succinct. It is an entirely generic abstraction, and will work with all ADO implementations.

The library is delivered as a [computation expression](#command-builder) responsible for building `IDbCommand` instances, which is executed using the `Db` module.

Two sets of type [extensions](#reading-values) for `IDataReader` are included to make manual object mapping a lot easier.

> If you came looking for an ORM, this is not your light saber. And may the force be with you.

## Design Goals 

- Support all ADO implementations.
- Provide a [natural DSL](#quick-start) for interacting with databases.
- Enable asynchronuos workflows.
- Provide explicit error flow control.
- Make object mapping easier.

## Getting Started

Install the [Donald](https://www.nuget.org/packages/Donald/) NuGet package:

```
PM>  Install-Package Donald
```

Or using the dotnet CLI
```cmd
dotnet add package Donald
```

### Quick Start

```fsharp
open Donald

type Author = 
    { FullName : string }

    static member Create fullName =
        { FullName = fullName }

module Author =
  let fromDataReader (rd : IDataReader) =
      let fullName = rd.ReadString "full_name" 
      Author.Create fullName

use conn = new SQLiteConnection("{your connection string}")

let authors : DbResult<Author list> =    
    dbCommand conn {
        cmdText  "SELECT  author_id
                        , full_name 
                  FROM    author 
                  WHERE   author_id = @author_id"
        cmdParam  [ "author_id", SqlType.Int 1]
    }
    |> Db.query Author.fromDataReader
```

## An Example using SQLite

For this example, assume we have an `IDbConnection` named `conn`:

> Reminder: Donald will work with __any__ ADO implementation (SQL Server, SQLite, MySQL, Postgresql etc.).

Consider the following model:

```fsharp
type Author = 
    { AuthorId : int
      FullName : string }

    static member Create authorId fullName =
        { AuthorId = authorId
          FullName = fullName }
    
module Author
    let fromDataReader (rd : IDataReader) : Author = 
        // IDataReader extension method (see below)
        let authorId = rd.ReadInt32 "author_id"
        let fullName = rd.ReadString "full_name"
          
        Author.Create authorId fullName
```

### Query for multiple strongly-typed results

```fsharp
dbCommand conn {
    cmdText "SELECT  author_id
                   , full_name 
             FROM    author"
}
|> Db.query Author.fromDataReader // DbResult<Author list>

// Async
dbCommand conn {
    cmdText "SELECT  author_id
                   , full_name 
             FROM    author"
}
|> Db.Async.query Author.fromDataReader // Task<DbResult<Author list>>
```

### Query for a single strongly-typed result

```fsharp
dbCommand conn {
    cmdText "SELECT  author_id
                   , full_name 
             FROM    author 
             WHERE   author_id = @author_id"
    cmdParam [ "author_id", SqlType.Int 1]
} 
|> Db.querySingle Author.fromDataReader // DbResult<Author option>

// Async
dbCommand conn {
    cmdText "SELECT  author_id
                   , full_name 
             FROM    author 
             WHERE   author_id = @author_id"
    cmdParam [ "author_id", SqlType.Int 1]
} 
|> Db.Async.querySingle Author.fromDataReader // Task<DbResult<Author option>>
```

### Execute a statement

```fsharp
dbCommand conn {
    cmdText "INSERT INTO author (full_name)"
    cmdParam [ "full_name", SqlType.String "John Doe" ]
}
|> Db.exec // DbResult<unit>

// Async
dbCommand conn {
    cmdText "INSERT INTO author (full_name)"
    cmdParam [ "full_name", SqlType.String "John Doe" ]
}
|> Db.Async.exec // Task<DbResult<unit>>
```

### Execute a statement many times

```fsharp
dbCommand conn {
   cmdText "INSERT INTO author (full_name)" 
}
|> Db.execMany [ "full_name", SqlType.String "John Doe"
                 "full_name", SqlType.String "Jane Doe" ]

// Async
dbCommand conn {
   cmdText "INSERT INTO author (full_name)" 
}
|> Db.Async.execMany [ "full_name", SqlType.String "John Doe"
                       "full_name", SqlType.String "Jane Doe" ]                           
```

### Execute statements within an explicit transaction

Donald exposes most of it's functionality through `dbCommand { ... }` and the `Db` module. But three type extension methods are exposed to make dealing with transactions safer.

> Donald contains a computation expression `dbResult { ... }` for dealing with `DbResult<'a>' instances, which is especially useful when you are working with dependent commands, which is common during transactional work.

```fsharp
// Safely begin transaction or throw CouldNotBeginTransactionError on failure
use tran = conn.TryBeginTransaction()

// Build our IDbCommand's
let param = [ "full_name", SqlType.String "John Doe" ]

let insertCmd = dbCommand conn {
    cmdText "INSERT INTO author (full_name)"
    cmdParam param
    cmdTran  tran
}

let selectCmd = dbCommand conn {
    cmdText "SELECT  author_id
                   , full_name 
             FROM    author 
             WHERE   full_name = @author_id"
    cmdParam param
    cmdTran  tran
} 

// Execute IDbCommand's
let result = dbResult {
  do! insertCmd |> Db.exec 
  return! selectCmd |> Db.querySingle Author.fromDataReader
}

// Attempt to commit, rollback on failure and throw CouldNotCommitTransactionError
tran.TryCommit() // or, safely rollback tran.TryRollback()
```

## Command Builder

At the core of Donald is a computation expression for building `IDbCommand` instances. It exposes five modification points:

1. `cmdText` - SQL statement you intend to execute (default: `String.empty`).
2. `cmdParam` - Input parameters for your statement (default: `[]`). 
3. `cmdType` - Type of command you want to execute (default: `CommandType.Text`) 
4. `cmdTran` - Transaction to assign to command.
5. `cmdTimeout` - The maximum time a command can run for (default: underlying DbCommand default, usually 30 seconds)

## Reading Values

To make obtaining values from reader more straight-forward, 2 sets of extension methods are available for:
1. Get value, automatically defaulted
2. Get value as `option<'a>`

> If you need an explicit `Nullable<'a>` you can use `Option.asNullable`.

Assuming we have an active `IDataReader` called `rd` and are currently reading a row, the following extension methods are available to simplify reading values:

```fsharp
rd.ReadString "some_field"           // string -> string
rd.ReadBoolean "some_field"          // string -> bool
rd.ReadByte "some_field"             // string -> byte
rd.ReadChar "some_field"             // string -> char
rd.ReadDateTime "some_field"         // string -> DateTime
rd.ReadDateTimeOffset "some_field"   // string -> DateTimeOffset
rd.ReadDecimal "some_field"          // string -> Decimal
rd.ReadDouble "some_field"           // string -> Double
rd.ReadFloat "some_field"            // string -> float32
rd.ReadGuid "some_field"             // string -> Guid
rd.ReadInt16 "some_field"            // string -> int16
rd.ReadInt32 "some_field"            // string -> int32
rd.ReadInt64 "some_field"            // string -> int64
rd.ReadBytes "some_field"            // string -> byte[]

rd.ReadStringOption "some_field"         // string -> string option
rd.ReadBooleanOption "some_field"        // string -> bool option
rd.ReadByteOption "some_field"           // string -> byte option
rd.ReadCharOption "some_field"           // string -> char option
rd.ReadDateTimeOption "some_field"       // string -> DateTime option
rd.ReadDateTimeOffsetOption "some_field" // string -> DateTimeOffset option
rd.ReadDecimalOption "some_field"        // string -> Decimal option
rd.ReadDoubleOption "some_field"         // string -> Double option
rd.ReadFloatOption "some_field"          // string -> float32 option
rd.ReadGuidOption "some_field"           // string -> Guid option
rd.ReadInt16Option "some_field"          // string -> int16 option
rd.ReadInt32Option "some_field"          // string -> int32 option
rd.ReadInt64Option "some_field"          // string -> int64 option
rd.ReadBytesOption "some_field"          // string -> byte[] option
```

## Exceptions

Donald exposes six custom exception types to represent failure at different points in the lifecycle:

```fsharp
exception ConnectionBusyError
exception CouldNotOpenConnectionError of exn
exception CouldNotBeginTransactionError of exn
exception CouldNotCommitTransactionError of exn
exception CouldNotRollbackTransactionError of exn
```

During command execution failures the `Error` case of `DbResult<'a>` is used, that encapsulates a `DbExecutionError` record. These are produced internally as a `FailedExecutionError` and transformed by the `Db` module.

```fsharp
type DbExecutionError = 
    { Statement : string
      Error     : DbException }

type DbResult<'a> = Result<'a, DbExecutionError>

exception FailedExecutionError of DbExecutionError
```

> It's important to note that Donald will only raise these exceptions in _exceptional_ situations. 

## Find a bug?

There's an [issue](https://github.com/pimbrouwers/Donald/issues) for that.

## License

Built with ♥ by [Pim Brouwers](https://github.com/pimbrouwers) in Toronto, ON. Licensed under [Apache License 2.0](https://github.com/pimbrouwers/Donald/blob/master/LICENSE).
