title = "SQLite Storage"
template = "spin_main"
date = "2023-04-14T00:22:56Z"
enable_shortcodes = true
[extra]
url = "https://github.com/fermyon/developer/blob/main/content/spin/sqlite-api-guide.md"

---
- [Using SQLite Storage From Applications](#using-sqlite-storage-from-applications)
- [Preparing an SQLite Database](#preparing-an-sqlite-database)
- [Custom SQLite Databases](#custom-sqlite-databases)
- [Granting SQLite Database Permissions to Components](#granting-sqlite-database-permissions-to-components)

Spin provides an interface for you to persist data in an SQLite database managed by Spin. This database allows Spin developers to persist relational data across application invocations.

{{ details "Why do I need a Spin interface? Why can't I just use my own external database?" "You can absolutely still use your own external database either with the [MySQL or Postgres APIs](/spin/rdbms-storage). However, if you're interested in quick, local relational storage without any infrastructure set-up then Spin's SQLite database is a great option." }}

## Using SQLite Storage From Applications

The Spin SDK surfaces the Spin SQLite database interface to your language.

The set of operations is common across all SDKs:

| Operation  | Parameters | Returns | Behavior |
|------------|------------|---------|----------|
| `open`  | name | connection  | Open the database with the specified name. If `name` is the string "default", the default database is opened, provided that the component that was granted access in the component manifest from `spin.toml`. Otherwise, `name` must refer to a store defined and configured in a [runtime configuration file](/spin/dynamic-configuration.md#sqlite-storage-runtime-configuration) supplied with the application.|
| `execute` | connection, sql, parameters | database records | Executes the SQL statement and returns the results of the statement. SELECT statements typically return records or scalars. INSERT, UPDATE, and DELETE statements typically return empty result sets, but may return values in some cases. |
| `close` | connection | - | Close the specified `connection`. |

The exact detail of calling these operations from your application depends on your language:

{{ tabs "sdk-type" }}

{{ startTab "Rust"}}

SQLite functions are available in the `spin_sdk::sqlite` module. The function names match the operations above. For example:

```rust
use anyhow::Result;
use serde::Serialize;
use spin_sdk::{
    http::{Request, Response},
    http_component,
    sqlite::{DataTypeParam, Error, Connection},
};

#[http_component]
fn handle_request(req: Request) -> Result<Response> {
    let connection = Connection::open_default()?;

    let execute_params = [
        ValueParam::Text("Try out Spin SQLite"),
        ValueParam::Text("Friday"),
    ];
    connection.execute(
        "INSERT INTO todos (description, due) VALUES (?, ?)",
        execute_params.as_slice(),
    )?;

    let rowset = connection.execute(
        "SELECT id, description, due FROM todos",
        &[]
    )?;

    let todos = rowset.rows().map(|row|
        ToDo {
            id: row.get::<u32>("id").unwrap(),
            description: row.get::<&str>("description").unwrap().to_owned(),
            due: row.get::<&str>("due").unwrap().to_owned(),
        }
    );

    let body = serde_json::to_vec(&todos)?;
    Ok(http::Response::builder().status(200).body(Some(body.into()))?)
}

// Helper for returning the query results as JSON
#[derive(Serialize)]
struct ToDo {
    id: u32,
    description: String,
    due: String,
}
```

**General Notes** 
* All functions are on the `spin_sdk::sqlite::Connection` type.
* Parameters are instances of the `ValueParam` enum; you must wrap raw values in this type.
* The `execute` function returns a `QueryResult`. To iterate over the rows use the `rows()` function. This returns an iterator; use `collect()` if you want to load it all into a collection.
* The values in rows are instances of the `ValueResult` enum.  However, you can use `row.get(column_name)` to extract a specific column from a row.  `get` casts the database value to the target Rust type. If the compiler can't infer the target type, write `row.get::<&str>(column_name)` (or whatever the desired type is).
* All functions wrap the return in `Result`, with the error type being `spin_sdk::sqlite::Error`.

{{ blockEnd }}

{{ startTab "Typescript"}}

To use SQLite functions, use the `Sqlite.open` or `Sqlite.openDefault` function to obtain a `Connection` object. `Connection` provides the `execute` method as described above. For example:

```javascript
import {Sqlite} from "@fermyon/spin-sdk"

const conn = Sqlite.openDefault();
const result = conn.execute("SELECT * FROM todos WHERE id > (?);", [1]);
const json = JSON.stringify(result.rows);
```

**General Notes**
* The `spinSdk` object is always available at runtime. Code checking and completion are available in TypeScript at design time if the module imports anything from the `@fermyon/spin-sdk` package.
* Parameters are JavaScript values (numbers, strings, byte arrays, or nulls). Spin infers the underlying SQL type.
* The `execute` function returns an object with `rows` and `columns` properties. `columns` is an array of strings representing column names. `rows` is an array of rows, each of which is an array of JavaScript values (as above) in the same order as `columns`.
* The `Connection` object doesn't surface the `close` function.
* Errors are surfaced as exceptions.

{{ blockEnd }}

{{ startTab "Python"}}

To use SQLite functions, use the `spin_sqlite` module in the Python SDK. The `sqlite_open` and `sqlite_open_default` functions return a connection object. The connection object provides the `execute` method as described above. For example:

```python
from spin_http import Response
from spin_sqlite import sqlite_open_default

def handle_request(request):
    conn = sqlite_open_default()
    result = conn.execute("SELECT * FROM todos WHERE id > (?);", [1])
    rows = result.rows()
    return Response(200,
                    {"content-type": "application/json"},
                    bytes(str(rows), "utf-8"))
```

**General Notes**
* Parameters are Python values (numbers, strings, and lists). Spin infers the underlying SQL type.
* The `execute` method returns an object with `rows` and `columns` methods. `columns` returns a list of strings representing column names. `rows` is an array of rows, each of which is an array of Python values (as above) in the same order as `columns`.
* The connection object doesn't surface the `close` function.
* Errors are surfaced as exceptions.

{{ blockEnd }}

{{ startTab "TinyGo"}}

The Go SDK doesn't currently surface the SQLite API.

{{ blockEnd }}

{{ blockEnd }}

## Preparing an SQLite Database

Although Spin provides SQLite as a built-in database, SQLite still needs you to create its tables.  In most cases, the most convenient way to do this is to use the `spin up --sqlite` option to run whatever SQL statements you need before your application starts.  This is typically used to create or alter tables, but can be used for whatever other maintenance or troubleshooting tasks you need.

You can run a SQL script from a file using the `@filename` syntax:

<!-- @nocpy -->

```bash
spin up --sqlite @migration.sql
```

Or you can pass SQL statements directly on the command line as a (quoted) string:

<!-- @nocpy -->

```bash
spin up --sqlite "CREATE TABLE IF NOT EXISTS todos (id INTEGER PRIMARY KEY AUTOINCREMENT, description TEXT NOT NULL, due TEXT NOT NULL)"
```

You can provide the `--sqlite` flag more than once; Spin runs the statements (or files) in the order you provide them, and waits for each to complete before running the next.

> It's also possible to create tables from your Wasm components using the usual `execute` function. That can end up mingling your "hot path" application logic with database maintenance code; decide which approach is best based on your application's needs.

## Custom SQLite Databases

Spin defines a database named `"default"` and provides automatic backing storage.  If you need to customize Spin with additional databases, or to change the backing storage for the default database, you can do so via the `--runtime-config-file` flag and the `runtime-config.toml` file.  See [SQLite Database Runtime Configuration](/spin/dynamic-configuration#sqlite-storage-runtime-configuration) for details.

## Granting SQLite Database Permissions to Components

By default, a given component of an app will not have access to any SQLite databases. Access must be granted specifically to each component via the component manifest.  For example, a component could be given access to the default store using:

```toml
[component]
sqlite_databases = ["default"]
```

Components can be given access to different databases:

```toml
# c1 has no access to any databases
[component]
name = "c1"

# c2 can use the default database, but no custom databases
[component]
name = "c2"
sqlite_databases = ["default"]

# c3 can use the custom databases "marketing" and "sales", which must be
# defined in the runtime config file, but cannot use the default database
[component]
name = "c3"
sqlite_databases = ["marketing", "sales"]
```
