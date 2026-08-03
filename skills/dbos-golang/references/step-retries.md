---
title: Configure Step Retries for Transient Failures
impact: HIGH
impactDescription: Automatic retries handle transient failures without manual code
tags: step, retry, exponential-backoff, resilience
---

## Configure Step Retries for Transient Failures

Steps can automatically retry on failure with exponential backoff. This handles transient failures like network issues.

**Incorrect (manual retry logic):**

```go
func fetchData(ctx context.Context) (string, error) {
	var lastErr error
	for attempt := 0; attempt < 3; attempt++ {
		resp, err := http.Get("https://api.example.com")
		if err == nil {
			defer resp.Body.Close()
			body, _ := io.ReadAll(resp.Body)
			return string(body), nil
		}
		lastErr = err
		time.Sleep(time.Duration(math.Pow(2, float64(attempt))) * time.Second)
	}
	return "", lastErr
}
```

**Correct (built-in retries with `dbos.RunAsStep`):**

```go
func fetchData(ctx context.Context) (string, error) {
	resp, err := http.Get("https://api.example.com")
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	return string(body), nil
}

func myWorkflow(ctx dbos.Context, input string) (string, error) {
	data, err := dbos.RunAsStep(ctx, fetchData,
		dbos.WithStepName("fetchData"),
		dbos.WithStepMaxRetries(10),
		dbos.WithStepBaseInterval(500*time.Millisecond),
		dbos.WithStepBackoffFactor(2.0),
		dbos.WithStepMaxInterval(5*time.Second),
	)
	return data, err
}
```

Retry parameters:
- `WithStepMaxRetries(n)`: Maximum retry attempts (default: `0` — no retries)
- `WithStepBaseInterval(d)`: Initial delay between retries (default: `100ms`)
- `WithStepBackoffFactor(f)`: Multiplier for exponential backoff (default: `2.0`)
- `WithStepMaxInterval(d)`: Maximum delay between retries (default: `5s`)
- `WithStepRetryPredicate(fn)`: Retry only when `fn(err)` returns true

With defaults, retry delays are: 100ms, 200ms, 400ms, 800ms, 1.6s, 3.2s, 5s, 5s...

If all retries are exhausted, a `dbos.Error` with code `ErrorCodeMaxStepRetriesExceeded` is returned to the calling workflow (match it with `errors.Is(err, dbos.ErrMaxStepRetriesExceeded)`).

Reference: [Configurable Retries](https://docs.dbos.dev/golang/tutorials/step-tutorial#configurable-retries)
