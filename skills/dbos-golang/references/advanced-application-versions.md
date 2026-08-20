---
title: Manage Application Versions for Safe Code Upgrades
impact: MEDIUM
impactDescription: Pin recovery to a specific binary and roll forward without losing in-flight work
tags: advanced, versioning, recovery, deploy
---

## Manage Application Versions for Safe Code Upgrades

DBOS tracks an application version for every workflow. Recovery only re-runs workflows whose stored `application_version` matches the executor's current version, so changing workflow logic without bumping the version can deadlock recovery. See [advanced-versioning.md](advanced-versioning.md) for the underlying constraint; this reference covers the management API.

### Setting the version

**Incorrect (no version control, every deploy gets a random hash):**

```go
// Binary hash changes on every recompile, so recovery thinks every workflow
// belongs to a different version and refuses to re-run them.
ctx, _ := dbos.NewContext(context.Background(), dbos.Config{
    AppName:     "my-app",
    DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
})
```

**Correct (explicit version):**

`Config.ApplicationVersion` overrides the auto-computed binary hash. The env var `DBOS__APPVERSION` overrides the config field:

```go
ctx, _ := dbos.NewContext(context.Background(), dbos.Config{
    AppName:            "my-app",
    DatabaseURL:        os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
    ApplicationVersion: "v1.2.3",
})
```

Read the resolved values from the context:

```go
version := dbos.GetApplicationVersion(ctx)
executorID := dbos.GetExecutorID(ctx)
appID := dbos.GetApplicationID(ctx)
```

### Inspecting and promoting versions

DBOS records every version that runs against the database. These functions take any `Client`, so they also work from a `dbos.Client` outside the app. Promote a previously-seen version to "latest" without redeploying:

```go
versions, _ := dbos.ListApplicationVersions(ctx)
for _, v := range versions {
    log.Printf("%s registered at %d (epoch ms)", v.Name, v.CreatedAt)
}

latest, _ := dbos.GetLatestApplicationVersion(ctx)
log.Printf("current latest: %s", latest.Name)

// Promote a previously-registered version. Workflows enqueued without an
// explicit ApplicationVersion will now run on this version.
dbos.SetLatestApplicationVersion(ctx, "v1.2.3")
```

`VersionInfo` fields:

- `ID string` — internal UUID
- `Name string` — the version name (binary hash by default)
- `Timestamp int64` — epoch ms, bumped by `SetLatestApplicationVersion`
- `CreatedAt int64` — epoch ms when the version was first registered
- `ApplicationName string` — owning application; empty if owned by no application

Versions are tracked per application: when multiple applications [share a system database](advanced-shared-database.md), these functions only see the calling application's versions (plus unowned ones), and promoting a version registered by a different application returns an error.

### Pinning a single workflow to a version

Override the version for one workflow at enqueue/start time:

```go
handle, _ := dbos.RunWorkflow(ctx, processTask, input,
    dbos.WithApplicationVersion("v1.2.3"))
```

The same option exists on the client side via `dbos.WithEnqueueApplicationVersion`.

Reference: [Application versions](https://docs.dbos.dev/golang/tutorials/workflow-management)
