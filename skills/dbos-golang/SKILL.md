---
name: dbos-golang
description: DBOS Go SDK for building reliable, fault-tolerant applications with durable workflows. Use this skill when writing Go code with DBOS, creating workflows and steps, using queues, using the DBOS Client from external applications, or building Go applications that need to be resilient to failures.
license: MIT
metadata:
  author: dbos
  version: "2.2.0"
  organization: DBOS
  date: October 2026
  abstract: Comprehensive guide for building fault-tolerant Go applications with DBOS. Covers workflows, steps, queues, communication patterns, and best practices for durable execution.
---

# DBOS Go Best Practices

Guide for building reliable, fault-tolerant Go applications with DBOS durable workflows.

## When to Apply

Reference these guidelines when:
- Adding DBOS to existing Go code
- Creating workflows and steps
- Running database transactions durably through datasources
- Using queues for concurrency control
- Implementing workflow communication (events, messages, streams)
- Configuring and launching DBOS applications
- Using the DBOS Client from external applications
- Testing DBOS applications

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Lifecycle | CRITICAL | `lifecycle-` |
| 2 | Workflow | CRITICAL | `workflow-` |
| 3 | Step | HIGH | `step-` |
| 4 | Queue | HIGH | `queue-` |
| 5 | Communication | MEDIUM | `comm-` |
| 6 | Pattern | MEDIUM | `pattern-` |
| 7 | Testing | LOW-MEDIUM | `test-` |
| 8 | Client | MEDIUM | `client-` |
| 9 | Advanced | LOW | `advanced-` |

## Critical Rules

### Installation

Install the DBOS Go module (v1):

```bash
go get github.com/dbos-inc/dbos-transact-golang/dbos@latest
```

Connecting to DBOS Conductor (setting `ConductorAPIKey`, or running in DBOS Cloud) requires importing `go.dbos.dev/dbos-enterprise-go` (`go get go.dbos.dev/dbos-enterprise-go`), with a blank import anywhere in the program; otherwise `dbos.NewContext` fails with an initialization error:

```go
import _ "go.dbos.dev/dbos-enterprise-go"
```

### DBOS Configuration and Launch

A DBOS application MUST create a context, register workflows, and launch before running any workflows:

```go
package main

import (
	"context"
	"log"
	"os"
	"time"

	"github.com/dbos-inc/dbos-transact-golang/dbos"
)

func main() {
	ctx, err := dbos.NewContext(context.Background(), dbos.Config{
		AppName:            "my-app",
		ApplicationVersion: "0.1.0",
		DatabaseURL:        os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	})
	if err != nil {
		log.Fatal(err)
	}
	defer dbos.Shutdown(ctx, 30*time.Second)

	dbos.RegisterWorkflow(ctx, myWorkflow)

	if err := dbos.Launch(ctx); err != nil {
		log.Fatal(err)
	}
}
```

When creating a new application, set `ApplicationVersion` to `"0.1.0"`. If omitted, DBOS derives an opaque hash from the binary. When editing an existing application, leave its configured version alone — changing it is a deployment decision (see `references/advanced-application-versions.md`).

`DatabaseURL` accepts Postgres/CockroachDB URLs or SQLite URLs (`sqlite:/path/to.db`, `sqlite::memory:`; SQLite requires a blank import of `github.com/dbos-inc/dbos-transact-golang/dbos/driver/sqlite`). See `references/lifecycle-config.md` for all configuration options, including Conductor settings and running schema migrations separately from the application.

`dbos.Context` extends `dbos.Client`: management functions (`Enqueue`, `ListWorkflows`, `CancelWorkflow`, queue and schedule management, ...) take a `dbos.Client` and accept either a `Context` or a standalone client from `dbos.NewClient` (see `references/client-setup.md`).

`AppName` identifies the application and owns everything it creates. Multiple applications (in any language) can share one system database, isolated by application name (v1.2.0+, see `references/advanced-shared-database.md`).

### Workflow and Step Structure

Workflows are comprised of steps. Any function performing complex operations or accessing external services must be run as a step using `dbos.RunAsStep`:

```go
func fetchData(ctx context.Context) (string, error) {
	resp, err := http.Get("https://api.example.com/data")
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	return string(body), nil
}

func myWorkflow(ctx dbos.Context, input string) (string, error) {
	result, err := dbos.RunAsStep(ctx, fetchData, dbos.WithStepName("fetchData"))
	if err != nil {
		return "", err
	}
	return result, nil
}
```

Database writes that must happen exactly once belong in a datasource transaction (`dbos.RunAsTransaction`), not a step (see `references/step-transactions.md`).

### Key Constraints

- Do NOT start or enqueue workflows from within steps
- Do NOT call `dbos.RunAsTransaction`, `handle.GetResult`, or `dbos.CloseStream` from within steps (reads, `dbos.SetEvent`, and `dbos.WriteStream` are allowed)
- Do NOT use uncontrolled goroutines to start workflows - use `dbos.RunWorkflow` with queues or `dbos.Go`/`dbos.Select` for concurrent steps
- Workflows MUST be deterministic - non-deterministic operations go in steps
- Do NOT modify global variables from workflows or steps
- All workflows MUST be registered, and all datasources created with `dbos.NewDataSource`, before calling `Launch()`; queues are registered with `dbos.RegisterQueue` and persisted in the database
- Starting a workflow with an ID already in use returns the existing workflow by default; pass `dbos.WithWorkflowIDReusePolicy(dbos.WorkflowIDReusePolicyReject)` to get `dbos.ErrWorkflowIDInUse` instead (see `references/pattern-idempotency.md`)
- A panic in a workflow does not crash the process: DBOS records it as the workflow's error (`dbos.ErrWorkflowPanic`)

## How to Use

Read individual rule files for detailed explanations and examples:

```
references/lifecycle-config.md
references/workflow-determinism.md
references/step-transactions.md
references/queue-concurrency.md
references/workflow-control.md
```

## References

- https://docs.dbos.dev/
- https://github.com/dbos-inc/dbos-transact-golang
