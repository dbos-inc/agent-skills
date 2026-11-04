---
title: Customize Workflow Serialization
impact: MEDIUM
impactDescription: Payloads that must be encrypted or compressed are stored as plain JSON
tags: advanced, serialization, serde, custom
---

## Customize Workflow Serialization

By default payloads are `serde_json` text (`rust_serde`). Implement `CustomSerializer` (a name, `serialize`, `deserialize`) and set `config.serializer = Serializer::custom(...)` to encode them yourself. The format must be self-describing, its output can't contain NUL, the name must stay fixed, and it must stay configured while rows written with it exist.

**Incorrect (bincode, which is not self-describing):**

```rust
// WRONG: bincode isn't self-describing, so DBOS can't read back the
// serde_json::Value payloads it records.
fn serialize(&self, value: Payload<'_>) -> Result<String, Box<dyn Error + Send + Sync>> {
    Ok(base64(bincode::serialize(&value)?))
}
```

**Correct (a custom serializer):**

```rust
/// Wraps JSON in an envelope; a real serializer might compress or encrypt it.
#[derive(Debug)]
struct EnvelopeJson;

impl CustomSerializer for EnvelopeJson {
    fn name(&self) -> &str {
        // Recorded on every row this serializer writes.
        "envelope_json"
    }

    fn serialize(
        &self,
        value: Payload<'_>,
    ) -> Result<String, Box<dyn std::error::Error + Send + Sync>> {
        Ok(format!("v1:{}", serde_json::to_string(&value)?))
    }

    fn deserialize(
        &self,
        text: &str,
        into: PayloadSlot<'_>,
    ) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
        let json = text.strip_prefix("v1:").ok_or("not an envelope")?;
        let mut deserializer = serde_json::Deserializer::from_str(json);
        into.fill(&mut deserializer)?;
        Ok(deserializer.end()?)
    }
}

fn configure() -> Config {
    let mut config = Config::from_env("dbos-starter");
    config.serializer = Serializer::custom(EnvelopeJson);
    config
}
```
