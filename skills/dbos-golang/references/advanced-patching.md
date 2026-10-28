---
title: Use Patching for Safe Workflow Upgrades
impact: LOW
impactDescription: Safely deploy breaking workflow changes without disrupting in-progress workflows
tags: advanced, patching, upgrade, breaking-change, auto-upgrade, WithAutoUpgrade, enterprise
---

## Use Patching for Safe Workflow Upgrades

Use `dbos.Patch` to safely deploy breaking changes to workflow code. Breaking changes alter which steps run or their order, which can cause recovery failures.

**Incorrect (breaking change without patching):**

```go
// BEFORE: original workflow
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	result, _ := dbos.RunAsStep(ctx, foo, dbos.WithStepName("foo"))
	_, _ = dbos.RunAsStep(ctx, bar, dbos.WithStepName("bar"))
	return result, nil
}

// AFTER: breaking change - recovery will fail for in-progress workflows!
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	result, _ := dbos.RunAsStep(ctx, baz, dbos.WithStepName("baz")) // Changed step
	_, _ = dbos.RunAsStep(ctx, bar, dbos.WithStepName("bar"))
	return result, nil
}
```

**Correct (using patch):**

```go
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	useBaz, err := dbos.Patch(ctx, "use-baz")
	if err != nil {
		return "", err
	}
	var result string
	if useBaz {
		result, _ = dbos.RunAsStep(ctx, baz, dbos.WithStepName("baz")) // New workflows
	} else {
		result, _ = dbos.RunAsStep(ctx, foo, dbos.WithStepName("foo")) // Old workflows
	}
	_, _ = dbos.RunAsStep(ctx, bar, dbos.WithStepName("bar"))
	return result, nil
}
```

`dbos.Patch` returns `true` for new workflows and `false` for workflows that started before the patch.

**Deprecating patches (after all old workflows complete):**

```go
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	dbos.DeprecatePatch(ctx, "use-baz") // Always takes the new path
	result, _ := dbos.RunAsStep(ctx, baz, dbos.WithStepName("baz"))
	_, _ = dbos.RunAsStep(ctx, bar, dbos.WithStepName("bar"))
	return result, nil
}
```

**Removing patches (after all workflows using DeprecatePatch complete):**

```go
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	result, _ := dbos.RunAsStep(ctx, baz, dbos.WithStepName("baz"))
	_, _ = dbos.RunAsStep(ctx, bar, dbos.WithStepName("bar"))
	return result, nil
}
```

Lifecycle: `Patch()` → deploy → wait for old workflows → `DeprecatePatch()` → deploy → wait → remove patch entirely.

### Recovering old workflows on patched code

`dbos.Patch` and `dbos.DeprecatePatch` need no configuration (v1.8.0+). `EnablePatching` is deprecated: it only pins the application version to `"PATCHING_ENABLED"` and logs a warning.

Recovery only runs a workflow on the application version that started it ([advanced-versioning.md](advanced-versioning.md)), so patched code must be able to pick up workflows started on old code. Either:

1. **Pin the application version**: keep the same `ApplicationVersion` across every patched deployment.
2. **Auto-upgrade**: give each deployment its own version and let DBOS move workflows left on older versions to the latest one.

**Incorrect (deprecated flag):**

```go
ctx, _ := dbos.NewContext(context.Background(), dbos.Config{
	AppName:        "my-app",
	DatabaseURL:    os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	EnablePatching: true, // Deprecated: Patch no longer needs it
})
```

**Correct (pinned version):**

```go
ctx, _ := dbos.NewContext(context.Background(), dbos.Config{
	AppName:            "my-app",
	DatabaseURL:        os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	ApplicationVersion: "1.0.0", // unchanged across patched deployments
})
```

### Auto-upgrade

Auto-upgrade (v1.8.0+) is a DBOS Transact Enterprise feature: it requires the `go.dbos.dev/dbos-enterprise-go` import (same minor version as the core module) and a key whose organization is entitled to it, `ConductorAPIKey` or, in development, `TransactEnterpriseKey`. **Only enroll a workflow if every breaking change to it is patched.**

```go
import (
	"github.com/dbos-inc/dbos-transact-golang/dbos"
	_ "go.dbos.dev/dbos-enterprise-go" // required for auto-upgrade
)

ctx, err := dbos.NewContext(context.Background(), dbos.Config{
	AppName:            "my-app",
	ApplicationVersion: os.Getenv("APP_VERSION"), // a new version per deployment
	DatabaseURL:        os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	ConductorAPIKey:    os.Getenv("DBOS_CONDUCTOR_KEY"),
	AutoUpgrade:        true,
	// AutoUpgradeGracePeriod: 5 * time.Minute, // let old executors finish PENDING workflows first
})
if err != nil {
	log.Fatal(err)
}
dbos.RegisterWorkflow(ctx, myWorkflow)                                    // enrolled
dbos.RegisterWorkflow(ctx, unpatchedWorkflow, dbos.WithAutoUpgrade(false)) // opted out
```

- `dbos.WithAutoUpgrade(true)` enrolls a single workflow even when `AutoUpgrade` is unset; `WithAutoUpgrade(false)` opts it out
- Every `AutoUpgradeInterval` (default 30s), one executor running the latest application version moves the enrolled workflows left on older versions: `ENQUEUED` and `DELAYED` ones immediately, so latest-version executors dequeue them; `PENDING` ones, once the latest version has been the latest for `AutoUpgradeGracePeriod` (default 0), are re-enqueued and recovered from their last completed step. If an old executor is still running one, only one execution records its outcome
- Without the import, `NewContext` (or `Launch`, for a workflow enrolled only through `WithAutoUpgrade`) fails with an initialization error, as it does when the key is missing or not entitled

Reference: [Patching](https://docs.dbos.dev/golang/tutorials/upgrading-workflows#patching), [DBOS Transact Enterprise](https://docs.dbos.dev/explanations/transact-enterprise)
