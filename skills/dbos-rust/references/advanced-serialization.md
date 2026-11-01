---
title: Customize Payload Encoding Safely with CustomSerializer
impact: LOW
impactDescription: Supports payload encryption, compression, or self-describing binary codecs without breaking internal DBOS checkpoints
tags: advanced, serialization, Serializer, CustomSerializer, Payload, PayloadSlot, encryption
---

## Customize Payload Encoding Safely with CustomSerializer

By default (`Serializer::RustSerde`), DBOS Rust encodes workflow inputs, step outputs, events, messages, and stream values as JSON under the `"rust_serde"` tag (preserving `f32`/`f64` `NaN` and `Infinity` as string tokens). Implement `dbos::CustomSerializer` and pass `Serializer::custom(codec)` on `Config::serializer` (and `ClientConfig::serializer`) when payloads require envelope encryption, compression, or another **self-describing** serde format encoded to UTF-8 text.

**Incorrect (using a non-self-describing format like bincode/postcard or reserved serializer name):**

```rust
use dbos::{CustomSerializer, Payload, PayloadSlot};

#[derive(Debug)]
struct BrokenCodec;

impl CustomSerializer for BrokenCodec {
    fn name(&self) -> &str {
        // Fails validation at `launch()`! `""`, `"rust_serde"`, and `"portable_json"` are reserved
        "rust_serde"
    }

    fn serialize(
        &self,
        _value: Payload<'_>,
    ) -> Result<String, Box<dyn std::error::Error + Send + Sync>> {
        // Note: Non-self-describing codecs (`bincode`, `postcard`) fail on `deserialize_any`,
        // which DBOS requires for `serde_json::Value` schedule contexts and internal checkpoints!
        Ok(String::new())
    }

    fn deserialize(
        &self,
        _text: &str,
        _into: PayloadSlot<'_>,
    ) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
        Ok(())
    }
}
```

**Correct (implementing a self-describing CustomSerializer with a versioned tag name):**

```rust
use dbos::{Config, CustomSerializer, DBOS, Payload, PayloadSlot, Serializer};

#[derive(Debug)]
struct PrefixedJsonSerializer;

impl CustomSerializer for PrefixedJsonSerializer {
    fn name(&self) -> &str {
        // Stored in each row's `serialization` column; keep stable so historical rows stay readable!
        "app_json_v1"
    }

    fn serialize(
        &self,
        value: Payload<'_>,
    ) -> Result<String, Box<dyn std::error::Error + Send + Sync>> {
        // `Payload<'_>` implements `serde::Serialize`; binary/encrypted bytes should be base64-encoded
        let json = serde_json::to_string(&value)?;
        Ok(format!("v1:{json}"))
    }

    fn deserialize(
        &self,
        text: &str,
        into: PayloadSlot<'_>,
    ) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
        let raw = text
            .strip_prefix("v1:")
            .ok_or("missing v1: payload prefix")?;
        let mut de = serde_json::Deserializer::from_str(raw);
        into.fill(&mut de)
    }
}

fn build_dbos_with_custom_serializer() -> DBOS {
    DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        serializer: Serializer::custom(PrefixedJsonSerializer),
        ..Config::from_env("vault-app")
    })
}
```

### Three Hard Rules for `CustomSerializer`

1. **Self-describing format (`deserialize_any` support)**: DBOS serializes `serde_json::Value` inside schedule contexts and internal checkpoints, so formats that omit field/type tags (`bincode`, `postcard`) fail at runtime. Use `serde_json` (plus optional AEAD encryption or zstd compression + base64), CBOR (`ciborium` + base64), or MessagePack (`rmp-serde` named mode + base64).
2. **Tag-driven reads**: Every row stores its `serialization` tag. Built-in `"rust_serde"` and `"portable_json"` rows always remain readable even when `Serializer::Custom` is configured, while an unrecognized custom tag fails cleanly with `Error::UnknownSerialization { name }`.
3. **Rewrite existing schedules after switching `Serializer`**: `workflow_schedules.context` has no `serialization` column, so after switching `Config::serializer`, re-run `apply_schedules` (or `update_schedule`) so stored schedule contexts are re-encoded under the new serializer.

Reference: [dbos::CustomSerializer](https://docs.rs/dbos/latest/dbos/trait.CustomSerializer.html), [dbos::Serializer](https://docs.rs/dbos/latest/dbos/enum.Serializer.html), [dbos::PayloadSlot::fill](https://docs.rs/dbos/latest/dbos/struct.PayloadSlot.html#method.fill)
