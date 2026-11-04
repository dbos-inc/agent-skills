---
title: Use Patching for Safe Workflow Upgrades
impact: LOW
impactDescription: An unpatched change to a workflow's steps breaks recovery of in-flight executions
tags: advanced, patch, deprecate_patch, upgrade
---

## Use Patching for Safe Workflow Upgrades

With a pinned app version, branch new code on `dbos::patch(name)`: `true` for executions that take the new code, `false` for those already past this point. When no old execution can run, replace it with `dbos::deprecate_patch(name)` and keep only the new path; later remove it. Call both from the workflow body only, and await each on its own: inside a `join!`, or built before another call that is awaited first, a patch fails with `Error::PatchNondeterminism`.

**Incorrect (changing steps in place):**

```rust
// Deployed while executions of the old code may still recover:
async fn workflow(_: ()) -> dbos::Result<String> {
    // WRONG: foo was replaced by baz in place. A recovered execution that recorded
    // foo at this position is refused with UnexpectedStep and never completes.
    let a = step("baz", baz).await?;
    let b = step("bar", bar).await?;
    Ok(a + &b)
}
```

**Correct (a patch):**

```rust
async fn workflow(_: ()) -> dbos::Result<String> {
    let a = if dbos::patch("use-baz").await? {
        // New workflows take the new code.
        step("baz", baz).await?
    } else {
        // Workflows that already ran past this point take the old code.
        step("foo", foo).await?
    };
    let b = step("bar", bar).await?;
    Ok(a + &b)
}
```

Then:

```rust
async fn workflow(_: ()) -> dbos::Result<String> {
    dbos::deprecate_patch("use-baz").await?;
    let a = step("baz", baz).await?;
    let b = step("bar", bar).await?;
    Ok(a + &b)
}
```
