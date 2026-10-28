---
title: Use Patching for Safe Workflow Upgrades
impact: LOW
impactDescription: Deploy breaking changes without disrupting in-progress workflows
tags: patching, upgrade, versioning, migration, auto-upgrade, auto_upgrade, enterprise
---

## Use Patching for Safe Workflow Upgrades

Use `DBOS.patch()` to safely deploy breaking workflow changes. Breaking changes alter what steps run or their order.

**Incorrect (breaking change without patch):**

```python
# Original
@DBOS.workflow()
def workflow():
    foo()
    bar()

# Updated - breaks in-progress workflows!
@DBOS.workflow()
def workflow():
    baz()  # Replaced foo() - checkpoints don't match
    bar()
```

**Correct (using patch):**

```python
@DBOS.workflow()
def workflow():
    if DBOS.patch("use-baz"):
        baz()  # New workflows use baz
    else:
        foo()  # Old workflows continue with foo
    bar()
```

Deprecating patches after all old workflows complete:

```python
# Step 1: Deprecate (runs all workflows, stops inserting marker)
@DBOS.workflow()
def workflow():
    DBOS.deprecate_patch("use-baz")
    baz()
    bar()

# Step 2: Remove entirely (after all deprecated workflows complete)
@DBOS.workflow()
def workflow():
    baz()
    bar()
```

`DBOS.patch(name)` returns:
- `True` for new workflows (started after patch deployed, or not yet past this point)
- `False` for old workflows (a different checkpoint already exists at this point)

`DBOS.patch` and `DBOS.deprecate_patch` need no configuration (3.4+). `enable_patching` is deprecated: it only pins `application_version` to `"PATCHING_ENABLED"` and logs a warning.

In coroutine (`async def`) workflows, use the async variants; the sync versions raise an error when called from a running event loop:

```python
@DBOS.workflow()
async def workflow():
    if await DBOS.patch_async("use-baz"):
        await baz()
    else:
        await foo()
    await bar()

# Later, when deprecating:
#     await DBOS.deprecate_patch_async("use-baz")
```

If a patch is missing, or deprecated/removed too early, the workflow raises `DBOSUnexpectedStepError` pointing at the mismatched step.

### Recovering Old Workflows on Patched Code

DBOS only recovers a workflow on the application version that started it (see [advanced-versioning](advanced-versioning.md)), so patched code must be able to pick up workflows started on old code. Either:

1. **Pin the application version**: keep the same `application_version` in every patched deployment:

```python
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    "application_version": "1.0.0",  # unchanged across patched deployments
}
```

2. **Auto-upgrade** (3.4+): give each deployment its own version and let DBOS move workflows left on older versions to the latest one.

### Auto-Upgrade

Auto-upgrade is a DBOS Transact Enterprise feature: it requires `pip install "dbos[enterprise]"` and a key whose organization is entitled to it (`conductor_key`, or `transact_enterprise_key` in development; see [lifecycle-config](lifecycle-config.md)). **Only auto-upgrade a workflow if every breaking change to it is patched.**

```python
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    "conductor_key": os.environ.get("DBOS_CONDUCTOR_KEY"),
    "auto_upgrade": True,
    "auto_upgrade_grace_sec": 300,  # Optional: let old processes finish PENDING workflows first
}
DBOS(config=config)

@DBOS.workflow(auto_upgrade=False)  # opt a workflow with unpatched changes out
def workflow_without_patches():
    ...
```

- `@DBOS.workflow(auto_upgrade=True)` enrolls a single workflow even when `auto_upgrade` is not set in the config; `auto_upgrade=False` opts it out
- Every `auto_upgrade_interval_sec` (default 30), one process running the latest application version moves the enrolled workflows left on older versions: `ENQUEUED` and `DELAYED` ones immediately, so latest-version processes dequeue them; `PENDING` ones, once the latest version has been the latest for `auto_upgrade_grace_sec` (default 0), are re-enqueued and recovered from their last completed step. If an old process is still running one, only one execution records its outcome

**Upgrading to DBOS 3.0 with patching:** 3.0 changes the storage format of workflow inputs/outputs, and 2.x processes cannot process workflows created by 3.0. If you use patching, shut down all DBOS 2.x processes before launching 3.0 processes (see [advanced-upgrading-v3](advanced-upgrading-v3.md)).

Reference: [Patching](https://docs.dbos.dev/python/tutorials/upgrading-workflows#patching), [Auto-Upgrading Workflows](https://docs.dbos.dev/python/tutorials/upgrading-workflows#auto-upgrading-workflows)
