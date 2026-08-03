---
name: dbos-golang
description: DBOS Go SDK for building reliable, fault-tolerant applications with durable workflows. Use this skill when writing Go code with DBOS, creating workflows and steps, using queues, using the DBOS Client from external applications, or building Go applications that need to be resilient to failures.
license: MIT
metadata:
  author: dbos
  version: "2.0.0"
  organization: DBOS
  date: August 2026
  abstract: Comprehensive guide for building fault-tolerant Go applications with DBOS. Covers workflows, steps, queues, communication patterns, and best practices for durable execution.
---

# DBOS Go Best Practices

Guide for building reliable, fault-tolerant Go applications with DBOS durable workflows.

## When to Apply

Reference these guidelines when:
- Adding DBOS to existing Go code
- Creating workflows and steps
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

`DatabaseURL` accepts Postgres/CockroachDB URLs or SQLite URLs (`sqlite:/path/to.db`, `sqlite::memory:`; SQLite requires a blank import of `github.com/dbos-inc/dbos-transact-golang/dbos/driver/sqlite`). See `references/lifecycle-config.md` for all configuration options.

`dbos.Context` extends `dbos.Client`: management functions (`Enqueue`, `ListWorkflows`, `CancelWorkflow`, queue and schedule management, ...) take a `dbos.Client` and accept either a `Context` or a standalone client from `dbos.NewClient` (see `references/client-setup.md`).

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

### Key Constraints

- Do NOT start or enqueue workflows from within steps
- Do NOT call `dbos.RunAsTransaction`, `handle.GetResult`, or `dbos.CloseStream` from within steps (reads, `dbos.SetEvent`, and `dbos.WriteStream` are allowed)
- Do NOT use uncontrolled goroutines to start workflows - use `dbos.RunWorkflow` with queues or `dbos.Go`/`dbos.Select` for concurrent steps
- Workflows MUST be deterministic - non-deterministic operations go in steps
- Do NOT modify global variables from workflows or steps
- All workflows MUST be registered before calling `Launch()`; queues are registered with `dbos.RegisterQueue` and persisted in the database

## How to Use

Read individual rule files for detailed explanations and examples:

```
references/lifecycle-config.md
references/workflow-determinism.md
references/queue-concurrency.md
```

## References

- https://docs.dbos.dev/
- https://github.com/dbos-inc/dbos-transact-golang
