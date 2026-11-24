---
title: Diagnose Applications That Don't Connect
impact: MEDIUM
impactDescription: An application that never connects runs normally but gets no recovery, observability or management
tags: troubleshooting, connection, unavailable, executors, conductor_url, docker network, enterprise, name mismatch
---

## Diagnose Applications That Don't Connect

An application that can't reach Conductor keeps running and keeps retrying, so a misconfiguration shows up only as an application that never becomes available in the self-hosted Console. Check, in order: the URL, where the key came from, the registered name, the enterprise package, and container networking.

**Incorrect (application in a container, pointed at localhost):**

```yaml
services:
  my-app:
    build: .
    environment:
      DBOS_CONDUCTOR_URL: ws://localhost:8090/   # the app's own container, not Conductor
      DBOS_CONDUCTOR_KEY: ${DBOS_CONDUCTOR_KEY}
```

**Correct (application on the Compose network, using the service name):**

```yaml
services:
  my-app:
    build: .
    environment:
      DBOS_CONDUCTOR_URL: ws://conductor:8090/
      DBOS_CONDUCTOR_KEY: ${DBOS_CONDUCTOR_KEY}
    depends_on:
      conductor:
        condition: service_healthy
```

Checklist:
1. The URL is set. Without `conductor_url` (`conductorURL`, `ConductorURL`, `withConductorDomain`), the application connects to DBOS-hosted Conductor, which rejects a self-hosted key. Use `ws://localhost:8090/` from the host in a local trial, and your proxy's `wss://` URL, with any Conductor path prefix, in production
2. The API key was created on the self-hosted Conductor (its Console, or `dbosctl api-key create`), not at https://console.dbos.dev, and is not the license key
3. The name in the application's DBOS configuration matches the registered name exactly. Check with `dbosctl app list`
4. The enterprise package is installed at a matching version: `dbos[enterprise]` (Python), `@dbos-inc/dbos-enterprise` (TypeScript), `go.dbos.dev/dbos-enterprise-go` (Go), `dev.dbos:transact-enterprise` (Java). Without it, an application with a Conductor key fails when it starts
5. In a container, the application uses the Conductor service name, as above
6. Over `wss://`, the application trusts the proxy's certificate. A TLS handshake failure in the application's logs means it doesn't (see [production-tls-proxy](production-tls-proxy.md))
7. Frequent disconnects and reconnects can come from a proxy or load balancer idle timeout left at a short default; raise it to about 3600 seconds on every layer

Checking the connection:

```bash
dbosctl app list --profile local                 # STATUS is AVAILABLE once a process is connected
dbosctl app executors my-app --profile local     # the connected processes
```

If Conductor restarts, applications reconnect on their own.

Reference: [Conductor: Connecting To Conductor](https://docs.dbos.dev/conductor/overview#connecting-to-conductor)
