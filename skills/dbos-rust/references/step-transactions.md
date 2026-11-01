---
title: Make Database Writes in Steps Idempotent
impact: HIGH
impactDescription: Prevents duplicate rows or double decrements when a crash occurs between an application SQL commit and the DBOS step checkpoint
tags: step, database, sqlx, idempotency, at-least-once, postgres
---

## Make Database Writes in Steps Idempotent

`dbos` for Rust does **not** have datasources or transactional steps yet (`#[dbos::transaction]` and `run_as_transaction` do not exist). Application database writes executed inside `dbos::step` commit on your own `sqlx::PgPool` in a **separate transaction** from DBOS's step checkpoint in `dbos.operation_outputs`. If the process crashes in the narrow window after your SQL transaction commits and before DBOS records the step checkpoint, recovery will run that step again (**at-least-once** execution). Make SQL writes inside steps idempotent using natural domain keys or a `(workflow_id, step_id)` key.

**Incorrect (non-idempotent INSERT and decrement that double-apply if re-run before checkpointing):**

```rust
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
#[error("db error: {0}")]
struct DbError(String);

async fn place_order(pool: PgPool, sku: String, cents: i64) -> dbos::Result<i64, DbError> {
    dbos::step("insert_order_and_decrement", || {
        let pool = pool.clone();
        let sku = sku.clone();
        async move {
            // If the process dies right after `commit()` and before DBOS checkpoints the step,
            // recovery runs this step a second time, inserting a second order and decrementing twice!
            let mut tx = pool.begin().await.map_err(|e| DbError(e.to_string()))?;
            sqlx::query("UPDATE inventory SET stock = stock - 1 WHERE sku = $1")
                .bind(&sku)
                .execute(&mut *tx)
                .await
                .map_err(|e| DbError(e.to_string()))?;
            let order_id: i64 = sqlx::query_scalar(
                "INSERT INTO orders (sku, cents) VALUES ($1, $2) RETURNING id",
            )
            .bind(&sku)
            .bind(cents)
            .fetch_one(&mut *tx)
            .await
            .map_err(|e| DbError(e.to_string()))?;
            tx.commit().await.map_err(|e| DbError(e.to_string()))?;
            Ok(order_id)
        }
    })
    .await
}
```

**Correct (keying the application transaction on workflow_id + step_id with ON CONFLICT):**

```rust
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
#[error("db error: {0}")]
struct DbError(String);

async fn place_order(pool: PgPool, sku: String, cents: i64) -> dbos::Result<i64, DbError> {
    dbos::step("insert_order_and_decrement", || {
        let pool = pool.clone();
        let sku = sku.clone();
        async move {
            // Stable across retries and crash recovery for this exact workflow step
            let wf_id = dbos::workflow_id().unwrap_or_default();
            let step_no = dbos::step_id().unwrap_or(0);
            let op_key = format!("{wf_id}:{step_no}");

            let mut tx = pool.begin().await.map_err(|e| DbError(e.to_string()))?;

            // Upsert keyed on `op_key` (UNIQUE column on `orders`): only inserts once!
            let inserted: Option<i64> = sqlx::query_scalar(
                "INSERT INTO orders (op_key, sku, cents) VALUES ($1, $2, $3) \
                 ON CONFLICT (op_key) DO NOTHING RETURNING id",
            )
            .bind(&op_key)
            .bind(&sku)
            .bind(cents)
            .fetch_optional(&mut *tx)
            .await
            .map_err(|e| DbError(e.to_string()))?;

            let order_id = match inserted {
                Some(id) => {
                    sqlx::query("UPDATE inventory SET stock = stock - 1 WHERE sku = $1")
                        .bind(&sku)
                        .execute(&mut *tx)
                        .await
                        .map_err(|e| DbError(e.to_string()))?;
                    id
                }
                None => sqlx::query_scalar("SELECT id FROM orders WHERE op_key = $1")
                    .bind(&op_key)
                    .fetch_one(&mut *tx)
                    .await
                    .map_err(|e| DbError(e.to_string()))?,
            };

            tx.commit().await.map_err(|e| DbError(e.to_string()))?;
            Ok(order_id)
        }
    })
    .await
}
```

### Guidelines for Application SQL in DBOS Rust

- **Never put SQL queries directly in the workflow body**: Reads and writes must still run inside `dbos::step` so completed steps replay from `dbos.operation_outputs` without hitting your application tables again during recovery.
- **Derive deterministic idempotency keys from DBOS context**: Inside a step body, `dbos::workflow_id()` returns `Some(String)` and `dbos::step_id()` returns `Some(i32)` (the 0-based ordinal step position), which stay identical across step retries and process restarts.
- **Convert `sqlx::Error` to a serializable `DurableError`**: `sqlx::Error` does not implement `Serialize` or `Deserialize`. Map it into a domain error enum or struct (preserving retryability signals such as PostgreSQL SQLSTATE codes `"40001"` / `"40P01"` if using `StepOptions::should_retry`).

Reference: [DBOS Rust Widget Store - Note on Transactional Steps](https://github.com/dbos-inc/dbos-transact-rust/blob/main/demo-apps/dbos-rust-widget-store/README.md#a-note-on-transactional-steps), [dbos::workflow_id](https://docs.rs/dbos/latest/dbos/fn.workflow_id.html), [dbos::step_id](https://docs.rs/dbos/latest/dbos/fn.step_id.html)
