---
title: Use Datasources for Database Operations
impact: HIGH
impactDescription: Datasource transactions provide exactly-once database execution within workflows
tags: datasource, transaction, database, postgres, sqlite, sqlalchemy, async
---

## Use Datasources for Database Operations

Datasources wrap a SQLAlchemy engine with DBOS transaction tracking so each database operation inside a workflow runs exactly once, even if the workflow is interrupted and retried. Use `SQLAlchemyDatasource` for synchronous code and `AsyncSQLAlchemyDatasource` for `async def` code. Datasources connect to any PostgreSQL or SQLite database.

**Incorrect (raw database access in a workflow — not checkpointed):**

```python
@DBOS.workflow()
def add_greeting(name: str, note: str):
    # Direct DB access isn't tracked; on replay it runs again
    engine.execute("INSERT INTO greetings (name, note) VALUES (?, ?)", name, note)
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
from dbos import AsyncSQLAlchemyDatasource

# create() is a coroutine for the async datasource — await it
ads = await AsyncSQLAlchemyDatasource.create(os.environ["APP_DATABASE_URL"])

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

- `@ds.transaction(name=..., isolation_level=...)`: `isolation_level` is one of `"SERIALIZABLE"` (default), `"REPEATABLE READ"`, or `"READ COMMITTED"`. `name` is the step name recorded in the workflow log.
- `SQLAlchemyDatasource` only supports `def` functions; `AsyncSQLAlchemyDatasource` only supports `async def`. Decorating the wrong kind raises `DBOSException` at decoration time.
- Call `ds.sql_session()` / `ads.sql_session()` only inside a datasource transaction; it raises otherwise.
- `create(database_url, engine_kwargs=..., engine=..., schema=..., serializer=...)`: pass an existing engine via `engine`, or set `schema` for the `datasource_outputs` tracking table (defaults to `"dbos"`; Postgres only).
- Outside a workflow, datasource transactions run as ordinary SQLAlchemy transactions with no tracking overhead.

Reference: [Transactions & Datasources](https://docs.dbos.dev/python/tutorials/transaction-tutorial)
