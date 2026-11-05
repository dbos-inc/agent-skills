---
title: Choose the Async Runtime with Cargo Features
impact: HIGH
impactDescription: A build with no runtime feature, or the wrong one, does not compile or does not run
tags: lifecycle, runtime, tokio, smol, cargo, features
---

## Choose the Async Runtime with Cargo Features

DBOS spawns its background tasks and sets timers on the runtime its Cargo features select: `runtime-tokio` (default), `runtime-smol` or `runtime-async-global-executor`. To use smol or async-global-executor, turn off the default features and keep `engine` and `macros`. Several runtime features may be enabled; at run time DBOS uses Tokio when called from a Tokio runtime, otherwise async-global-executor, then smol.

**Incorrect (turning off defaults without a runtime or the engine):**

```toml
# No runtime feature: does not build.
dbos = { version = "0.6", default-features = false }
```

**Correct (smol, with engine and macros kept):**

```rust
use std::time::Duration;

use dbos::{Config, DBOS};

async fn workflow(_: ()) -> dbos::Result<String> {
    dbos::step("step_one", || async {
        println!("Step one completed");
        Ok(())
    })
    .await?;
    dbos::sleep(Duration::from_secs(1)).await?;
    dbos::step("step_two", || async {
        println!("Step two completed");
        Ok(())
    })
    .await?;
    Ok("success".to_owned())
}

fn main() -> dbos::Result<()> {
    smol::block_on(async {
        let mut config = Config::from_env("dbos-starter");
        config.app_version = Some("0.1.0".to_owned());
        let dbos = DBOS::new(config);

        let workflow = dbos.register_workflow("workflow", workflow)?;
        dbos.launch().await?;

        let result = workflow.run(()).await?;
        println!("Workflow result: {result}");

        dbos.shutdown().await;
        Ok(())
    })
}
```

The `Cargo.toml` for the correct example:

```toml
[dependencies]
dbos = { version = "0.6", default-features = false, features = ["engine", "macros", "runtime-smol"] }
smol = "2"
```

The `macros` feature provides `select_step!`. The determinism rules (`workflow-concurrency.md`) are the same on every runtime.
