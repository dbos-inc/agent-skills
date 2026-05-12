---
title: Configure Step Retries for Transient Failures
impact: HIGH
impactDescription: Automatic retries handle transient failures without manual code
tags: step, retry, exponential-backoff, resilience, should_retry
---

## Configure Step Retries for Transient Failures

Steps can automatically retry on failure with exponential backoff. This handles transient failures like network issues.

**Incorrect (manual retry logic):**

```python
@DBOS.step()
def fetch_data():
    # Manual retry logic is error-prone
    for attempt in range(3):
        try:
            return requests.get("https://api.example.com").json()
        except Exception:
            if attempt == 2:
                raise
            time.sleep(2 ** attempt)
```

**Correct (built-in retries):**

```python
@DBOS.step(retries_allowed=True, max_attempts=10, interval_seconds=1.0, backoff_rate=2.0)
def fetch_data():
    # Retries handled automatically
    return requests.get("https://api.example.com").json()
```

Retry parameters:
- `retries_allowed`: Enable automatic retries (default: `False`)
- `max_attempts`: Maximum retry attempts (default: `3`)
- `interval_seconds`: Initial delay between retries (default: `1.0`)
- `backoff_rate`: Multiplier for exponential backoff (default: `2.0`)
- `should_retry`: Optional predicate for selective retries (see below)

With defaults, retry delays are: 1s, 2s, 4s, 8s, 16s...

If a step exhausts all `max_attempts`, it raises `DBOSMaxStepRetriesExceeded` to the calling workflow.

### Filtering Retries With `should_retry`

By default every exception is retried. Use `should_retry` to skip retries for non-transient errors like validation failures or 4xx responses. The predicate receives the raised exception; returning `False` re-raises it immediately without further retries.

**Incorrect (retrying non-transient errors wastes attempts):**

```python
@DBOS.step(retries_allowed=True, max_attempts=10)
def fetch_user(user_id: str):
    response = requests.get(f"https://api.example.com/users/{user_id}")
    response.raise_for_status()  # 404 will retry 10 times before giving up
    return response.json()
```

**Correct (only retry transient failures):**

```python
@DBOS.step(
    retries_allowed=True,
    max_attempts=10,
    should_retry=lambda e: not (
        isinstance(e, requests.HTTPError)
        and 400 <= e.response.status_code < 500
    ),
)
def fetch_user(user_id: str):
    response = requests.get(f"https://api.example.com/users/{user_id}")
    response.raise_for_status()
    return response.json()
```

For async steps, `should_retry` may itself be an `async` function:

```python
async def is_retryable(e: BaseException) -> bool:
    return not isinstance(e, FatalError)

@DBOS.step(retries_allowed=True, max_attempts=10, should_retry=is_retryable)
async def example_step():
    ...
```

Async predicates are only supported for async steps; pairing an async `should_retry` with a sync step raises an exception.

Reference: [Configurable Retries](https://docs.dbos.dev/python/tutorials/step-tutorial#configurable-retries)
