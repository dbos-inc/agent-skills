---
title: Use Versioning for Blue-Green Deployments
impact: LOW
impactDescription: Safely deploy new code with version tagging
tags: versioning, blue-green, deployment, recovery
---

## Use Versioning for Blue-Green Deployments

DBOS versions workflows to prevent unsafe recovery. Use blue-green deployments to safely upgrade.

**Incorrect (deploying breaking changes without versioning):**

```python
# Deploying new code directly kills in-progress workflows
# because their checkpoints don't match the new code

# Old code
@DBOS.workflow()
def workflow():
    step_a()
    step_b()

# New code replaces old immediately - breaks recovery!
@DBOS.workflow()
def workflow():
    step_a()
    step_c()  # Changed step - old workflows can't recover
```

**Correct (using versioning with blue-green deployment):**

```python
# Set explicit version in config
config: DBOSConfig = {
    "name": "my-app",
    "application_version": "2.0.0",  # New version
}
DBOS(config=config)
```

Deploy new version alongside old version. Direct new traffic to v2.0.0, drain old workflows on v1.0.0.

### Directing Enqueued Workflows to Latest Version

Use `DBOS.get_latest_application_version` to route enqueued work to the latest version:

```python
from dbos import DBOS, SetEnqueueOptions

DBOS.register_queue("my_queue")

latest_version = DBOS.get_latest_application_version()
with SetEnqueueOptions(app_version=latest_version["version_name"]):
    DBOS.enqueue_workflow("my_queue", my_workflow, arg1, arg2)
```

Scheduled workflows are automatically enqueued to the latest version.

### Checking and Retiring Old Versions

```python
active = DBOS.list_workflows(
    app_version="1.0.0",
    status=["ENQUEUED", "PENDING"],
)
if not active:
    print("Safe to retire version 1.0.0")
```

### Reading the Current Version

```python
# Read the version this process is running as
print(DBOS.application_version)
```

### Version Management APIs

```python
# List all registered versions (newest first)
versions = DBOS.list_application_versions()

# Get the latest version
latest = DBOS.get_latest_application_version()

# Roll back: promote a previous version to latest
DBOS.set_latest_application_version("1.0.0")
```

Each `VersionInfo` is a dict with:

```python
class VersionInfo(TypedDict):
    version_id: str           # Unique ID
    version_name: str         # Unique name (matches the application_version config field)
    version_timestamp: int    # Epoch ms - determines which version is "latest"
    created_at: int           # Epoch ms when first registered
```

### Forking Workflows to a New Version

```python
new_handle = DBOS.fork_workflow(
    workflow_id="old-workflow-id",
    start_step=5,
    application_version="2.0.0"
)
```

Reference: [Versioning](https://docs.dbos.dev/python/tutorials/upgrading-workflows#versioning)
