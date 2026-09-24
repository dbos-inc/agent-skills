---
title: Use Patching for Safe Workflow Upgrades
impact: LOW
impactDescription: Deploy breaking changes without disrupting in-progress workflows
tags: patching, upgrade, versioning, migration
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
# Enable patching in config
config: DBOSConfig = {
    "name": "my-app",
    "enable_patching": True,
}
DBOS(config=config)

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

`enable_patching: True` is required: without it, `DBOS.patch` and `DBOS.deprecate_patch` raise `DBOSException`.

In coroutine (`async def`) workflows, use the async variants; the sync versions raise `RuntimeError` inside a running event loop:

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

If a patch is missing, or deprecated/removed too early, recovery raises `DBOSUnexpectedStepError` pointing at the mismatched step.

**Upgrading to DBOS 3.0 with patching:** 3.0 changes the storage format of workflow inputs/outputs, and 2.x processes cannot process workflows created by 3.0. Because patching keeps the same `application_version`, shut down all DBOS 2.x processes before launching 3.0 processes (see [advanced-upgrading-v3](advanced-upgrading-v3.md)).

Reference: [Patching](https://docs.dbos.dev/python/tutorials/upgrading-workflows#patching)
