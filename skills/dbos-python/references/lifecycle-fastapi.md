---
title: Integrate DBOS with FastAPI
impact: CRITICAL
impactDescription: Proper integration ensures workflows survive server restarts
tags: fastapi, http, server, integration
---

## Integrate DBOS with FastAPI

When using DBOS with FastAPI, configure and launch DBOS inside the main function before starting uvicorn, or launch it from a FastAPI lifespan function (see below).

**Incorrect (launching at import time):**

```python
from fastapi import FastAPI
from dbos import DBOS, DBOSConfig

app = FastAPI()

config: DBOSConfig = {"name": "my-app", "application_version": "0.1.0"}
DBOS(config=config)
DBOS.launch()  # Don't launch at import time - launch in main or a lifespan

@app.get("/")
@DBOS.workflow()
def endpoint():
    return {"status": "ok"}
```

**Correct (configuration in main):**

```python
import os
from fastapi import FastAPI
from dbos import DBOS, DBOSConfig
import uvicorn

app = FastAPI()

@DBOS.step()
def process_data():
    return "processed"

@app.get("/")
@DBOS.workflow()
def endpoint():
    result = process_data()
    return {"result": result}

if __name__ == "__main__":
    config: DBOSConfig = {
        "name": "my-app",
        "application_version": "0.1.0",
        "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    }
    DBOS(config=config)
    DBOS.launch()
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

The workflow decorator can be combined with FastAPI route decorators. The FastAPI decorator should come first (outermost).

The `DBOS(fastapi=app)` / `DBOS(flask=app)` constructor parameters were removed in 3.0. For HTTP request spans, use your framework's OpenTelemetry instrumentation; DBOS workflow spans join them automatically.

### Alternative: Launch From a Lifespan

If the server is started by an external runner (e.g. `uvicorn main:app`), launch and destroy DBOS in a FastAPI lifespan:

```python
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI
from dbos import DBOS, DBOSConfig

@asynccontextmanager
async def lifespan(app: FastAPI):
    DBOS.launch()
    await DBOS.register_queue_async("tasks")  # async context: use _async variants
    try:
        yield
    finally:
        DBOS.destroy()

app = FastAPI(lifespan=lifespan)
config: DBOSConfig = {
    "name": "my-app",
    "application_version": "0.1.0",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
}
DBOS(config=config)
```

### Async Handlers Must Use `_async` Methods

Many synchronous DBOS methods (such as `DBOS.sleep`, `DBOS.recv`, `DBOS.send`, `DBOS.set_event`, `DBOS.get_event`, and `DBOS.register_queue`) raise `RuntimeError` when called while an event loop is running, which includes every `async def` FastAPI handler.

**Incorrect (sync DBOS call in an async handler):**

```python
@app.get("/status/{workflow_id}")
async def status(workflow_id: str):
    return DBOS.get_event(workflow_id, "status")  # RuntimeError
```

**Correct:**

```python
@app.get("/status/{workflow_id}")
async def status(workflow_id: str):
    return await DBOS.get_event_async(workflow_id, "status")

# Start coroutine workflows with start_workflow_async
@app.post("/start")
async def start():
    handle = await DBOS.start_workflow_async(my_async_workflow)
    return {"id": handle.get_workflow_id()}

# Or keep the handler synchronous (FastAPI runs it in a threadpool)
@app.post("/start-sync")
def start_sync():
    handle = DBOS.start_workflow(my_sync_workflow)
    return {"id": handle.get_workflow_id()}
```

Reference: [DBOS with FastAPI](https://docs.dbos.dev/python/tutorials/workflow-tutorial)
