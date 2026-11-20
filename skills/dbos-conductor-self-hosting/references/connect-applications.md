---
title: Connect Applications to Self-Hosted Conductor
impact: HIGH
impactDescription: Without conductor_url, applications connect to DBOS-hosted Conductor instead
tags: connect, conductor_url, conductorURL, ConductorURL, withConductorDomain, conductor_key, api key, register, enterprise, websocket
---

## Connect Applications to Self-Hosted Conductor

An application connects to Conductor over a WebSocket, with a Conductor API key. For a self-hosted Conductor, register the application and create the API key on your self-hosted Conductor (its Console at http://localhost, or dbosctl), then give the application both the key and your Conductor's WebSocket URL. The application's name in its DBOS configuration must match the name it is registered under.

**Incorrect (no Conductor URL, or a key from the wrong place):**

```python
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    # No conductor_url: connects to DBOS-hosted Conductor, which rejects this key.
    # A key from https://console.dbos.dev or the license key fails the same way.
    "conductor_key": os.environ.get("DBOS_CONDUCTOR_KEY"),
}
```

**Correct (API key from the self-hosted Conductor, plus its WebSocket URL):**

```bash
export DBOS_CONDUCTOR_KEY="<API key from your self-hosted Conductor>"
export DBOS_CONDUCTOR_URL="ws://localhost:8090/"
```

```python
# pip install "dbos[enterprise]"
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    "conductor_key": os.environ.get("DBOS_CONDUCTOR_KEY"),
    "conductor_url": os.environ.get("DBOS_CONDUCTOR_URL"),
}
DBOS(config=config)
```

TypeScript passes both to `DBOS.launch`, not `DBOS.setConfig`, and needs `@dbos-inc/dbos-enterprise` at the same minor version as `@dbos-inc/dbos-sdk`:

```typescript
// npm install @dbos-inc/dbos-enterprise
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
});
await DBOS.launch({
  conductorKey: process.env.DBOS_CONDUCTOR_KEY,
  conductorURL: process.env.DBOS_CONDUCTOR_URL,
});
```

Go needs `go.dbos.dev/dbos-enterprise-go` at the same minor version as `github.com/dbos-inc/dbos-transact-golang`, imported once anywhere in the program:

```go
import _ "go.dbos.dev/dbos-enterprise-go"

dbosContext, err := dbos.NewContext(context.Background(), dbos.Config{
    AppName:         "my-app",
    DatabaseURL:     os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
    ConductorAPIKey: os.Getenv("DBOS_CONDUCTOR_KEY"),
    ConductorURL:    os.Getenv("DBOS_CONDUCTOR_URL"),
})
```

Java needs `dev.dbos:transact-enterprise` at the same version as `dev.dbos:transact`, and takes the URL with `withConductorDomain`:

```java
DBOSConfig config = DBOSConfig.defaults("my-app")
    .withDatabaseUrl(System.getenv("DBOS_SYSTEM_JDBC_URL"))
    .withConductorKey(System.getenv("DBOS_CONDUCTOR_KEY"))
    .withConductorDomain(System.getenv("DBOS_CONDUCTOR_URL"));
```

DBOS for Rust does not support Conductor yet.

Registering and creating the key:
- In your self-hosted Console (http://localhost for the local trial, your Console's address in production), register the application, then generate an API key. Not at https://console.dbos.dev
- Or with dbosctl: `dbosctl app register my-app --profile local`, then `dbosctl api-key create my-app-key --profile local`, which prints the key once (see [connect-dbosctl](connect-dbosctl.md))
- The API key is a secret: keep it in the application's environment or secret store

The URL:
- Use the full WebSocket URL: `ws://localhost:8090/` for the local trial, and in production the `wss://` URL of your TLS proxy, including any path prefix routed to Conductor, such as `wss://conductor.example.com/conductor-api` (see [production-tls-proxy](production-tls-proxy.md))
- Over `wss://`, the application must trust the proxy's certificate
- An application running in a container can't reach Conductor at `localhost`. Run it as a service in the same Compose project and use `ws://conductor:8090/`
- This differs from the Console container's `DBOS_CONDUCTOR_URL`, which is a bare `host:port` such as `conductor:8090`

Once connected, the application shows as available in the self-hosted Console and in `dbosctl app list`.

Reference: [Self-Hosting Guide: Connecting Applications](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#connecting-applications)
