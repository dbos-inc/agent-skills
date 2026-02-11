---
title: Integrate DBOS with HTTP Servers
impact: CRITICAL
impactDescription: Proper integration ensures workflows survive server restarts
tags: http, integration, server, gin, echo
---

## Integrate DBOS with HTTP Servers

Create the DBOS context, register workflows, and launch DBOS before starting your HTTP server.

**Incorrect (DBOS not launched before server starts):**

```go
func main() {
	http.HandleFunc("/process", func(w http.ResponseWriter, r *http.Request) {
		// DBOS not initialized - this will fail
		handle, _ := dbos.RunWorkflow(nil, processTask, r.URL.Query().Get("data"))
		result, _ := handle.GetResult()
		fmt.Fprint(w, result)
	})
	http.ListenAndServe(":8080", nil)
}
```

**Correct (launch DBOS first, then start server):**

```go
func processTask(ctx dbos.DBOSContext, data string) (string, error) {
	return dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
		return "processed: " + data, nil
	}, dbos.WithStepName("process"))
}

func main() {
	ctx, err := dbos.NewDBOSContext(context.Background(), dbos.Config{
		AppName:     "my-app",
		DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	})
	if err != nil {
		log.Fatal(err)
	}
	defer dbos.Shutdown(ctx, 30*time.Second)

	dbos.RegisterWorkflow(ctx, processTask)

	if err := dbos.Launch(ctx); err != nil {
		log.Fatal(err)
	}

	http.HandleFunc("/process", func(w http.ResponseWriter, r *http.Request) {
		handle, err := dbos.RunWorkflow(ctx, processTask, r.URL.Query().Get("data"))
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		result, err := handle.GetResult()
		if err != nil {
			http.Error(w, err.Error(), 500)
			return
		}
		fmt.Fprint(w, result)
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

The DBOS context is safe to share across goroutines — Go's `net/http` server handles each request in its own goroutine, and `dbos.RunWorkflow` is concurrency-safe.

Reference: [DBOS Go Programming Guide](https://docs.dbos.dev/golang/programming-guide)
