---
title: Use Datasources for Database Operations
impact: HIGH
impactDescription: Datasource transactions provide exactly-once database execution within workflows
tags: datasource, transaction, database, postgres, sqlite, sqlalchemy, async, migrate, sessionmaker
---

## Use Datasources for Database Operations

Datasources wrap a SQLAlchemy engine with DBOS transaction tracking so each database operation inside a workflow runs exactly once, even if the workflow is interrupted and retried. Use `SQLAlchemyDatasource` for synchronous code and `AsyncSQLAlchemyDatasource` for `async def` code. Datasources connect to any PostgreSQL or SQLite database.

**Incorrect (raw database access in a workflow — not checkpointed):**

```python
@DBOS.workflow()
def add_greeting(name: str, note: str):
    # Direct DB access isn't tracked; on replay it runs again
    with engine.begin() as conn:
        conn.execute(
            text("INSERT INTO greetings (name, note) VALUES (:name, :note)"),
            {"name": name, "note": note},
        )
```

**Correct (synchronous datasource):**

```python
import os
from dbos import DBOS, SQLAlchemyDatasource
from sqlalchemy import text

ds = SQLAlchemyDatasource.create(os.environ["APP_DATABASE_URL"])

@ds.transaction()
def insert_greeting(name: str, note: str) -> None:
    session = ds.sql_session()  # sqlalchemy.orm.Session
    session.execute(
        text("INSERT INTO greetings (name, note) VALUES (:name, :note)"),
        {"name": name, "note": note},
    )

@DBOS.workflow()
def greeting_workflow(name: str, note: str) -> None:
    insert_greeting(name, note)
```

**Async datasource (native `async def` transactions):**

```python
import asyncio
import os
from dbos import DBOS, AsyncSQLAlchemyDatasource
from sqlalchemy import text

# create() is a coroutine, and `await` is not allowed at module scope.
# Use asyncio.run so the datasource is a module-level global the decorators can use.
# (The datasource is not tied to the event loop that created it.)
ads = asyncio.run(AsyncSQLAlchemyDatasource.create(os.environ["APP_DATABASE_URL"]))

@ads.transaction()
async def insert_greeting(name: str, note: str) -> None:
    session = ads.sql_session()  # sqlalchemy.ext.asyncio.AsyncSession
    await session.execute(
        text("INSERT INTO greetings (name, note) VALUES (:name, :note)"),
        {"name": name, "note": note},
    )

@DBOS.workflow()
async def greeting_workflow(name: str, note: str) -> None:
    await insert_greeting(name, note)
```

**Incorrect (module-scope `await`, or creating a datasource after launch):**

```python
ads = await AsyncSQLAlchemyDatasource.create(url)  # SyntaxError at module scope

DBOS.launch()
ds = SQLAlchemyDatasource.create(url)  # Raises DBOSException
```

Create all datasources **before** `DBOS.launch()` (DBOS tracks them so `rewind_workflow` can delete their transaction checkpoints).

For `AsyncSQLAlchemyDatasource` with SQLite, use an async driver URL such as `sqlite+aiosqlite:///app.sqlite` (install with `pip install "dbos[aiosqlite]"`); a plain `sqlite:///` URL raises an error.

### Running Datasource Migrations Separately

By default, `create()` creates or migrates the `datasource_outputs` table (and its schema), which requires DDL privileges. If the application's database role can't run DDL, migrate with a privileged role and create the datasource with `run_migrations=False`, so it only verifies the table is migrated (raising `DBOSInitializationError` if not):

```python
# In your migration script, run with a privileged role.
# application_role (Postgres only) is granted USAGE on the schema, SELECT/INSERT/DELETE
# on datasource_outputs, and SELECT on the migration version table.
SQLAlchemyDatasource.migrate(os.environ["ADMIN_DATABASE_URL"], application_role="my_app_role")
# Async: await AsyncSQLAlchemyDatasource.migrate(...)

# In your application, run as my_app_role:
ds = SQLAlchemyDatasource.create(os.environ["APP_DATABASE_URL"], run_migrations=False)
```

`dbos migrate` and `DBOS.migrate` migrate only the system database, not datasource tables (see [lifecycle-config](lifecycle-config.md)).

### Running Inline Without a Decorator

Use `run_tx_step` (sync) or `run_tx_step_async` (async) to run an undecorated function as a datasource transaction:

```python
def insert_greeting(name: str, note: str) -> None:
    ds.sql_session().execute(
        text("INSERT INTO greetings (name, note) VALUES (:name, :note)"),
        {"name": name, "note": note},
    )

@DBOS.workflow()
def greeting_workflow(name: str, note: str) -> None:
    # First arg is a DatasourceOptions dict ({"name", "isolation_level"}) or None
    ds.run_tx_step({"name": "insert_greeting"}, insert_greeting, name, note)
```

For async code, use `await ads.run_tx_step_async({...}, async_fn, *args)`.

### Options and Notes

- `@ds.transaction(name=..., isolation_level=...)`: `isolation_level` is one of `"SERIALIZABLE"` (default), `"REPEATABLE READ"`, or `"READ COMMITTED"` (SQLite supports only `"SERIALIZABLE"`). `name` is the step name recorded in the workflow log (defaults to the function's `__qualname__`).
- `SQLAlchemyDatasource` only supports `def` functions; `AsyncSQLAlchemyDatasource` only supports `async def`. Decorating the wrong kind raises `DBOSException` at decoration time.
- Call `ds.sql_session()` / `ads.sql_session()` only inside a datasource transaction; it raises otherwise.
- `create(database_url, engine_kwargs=..., engine=..., schema=..., serializer=..., sessionmaker=..., run_migrations=True)`: pass an existing engine via `engine`, or set `schema` for the `datasource_outputs` tracking table (defaults to `"dbos"`; Postgres only). The default `serializer` is pickle, not the `serializer` in `DBOSConfig`. DBOS connects to Postgres with the psycopg driver.
- `sessionmaker`: a custom `sessionmaker` (or `async_sessionmaker` for `AsyncSQLAlchemyDatasource`) for transaction sessions, e.g. a custom `Session` subclass, session options, or event hooks. Sessions are always bound to the datasource's engine: any `bind` is ignored, and setting `binds` raises `DBOSException`. Defaults to `sessionmaker(expire_on_commit=False)`.
- `@DBOS.transaction`, `DBOS.sql_session`, and `application_database_url` were removed in 3.0; datasources replace them (see [advanced-upgrading-v3](advanced-upgrading-v3.md)).
- Outside a workflow, datasource transactions run as ordinary SQLAlchemy transactions with no tracking overhead.
- Once a workflow completes, DBOS deletes its rows from `datasource_outputs` (the outcome is recorded in the system database).

Reference: [Transactions & Datasources](https://docs.dbos.dev/python/tutorials/transaction-tutorial)
