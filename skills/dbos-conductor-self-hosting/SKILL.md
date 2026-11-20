---
name: dbos-conductor-self-hosting
description: Self-host DBOS Conductor and the DBOS Console, from a local trial to a production deployment. Use this skill when deploying Conductor on your own infrastructure (Docker Compose, Kubernetes or another container platform), getting a trial license key or a production license key (DBOS_CONDUCTOR_LICENSE_KEY), configuring OAuth authentication, TLS, a reverse proxy or high availability for Conductor, connecting DBOS applications to a self-hosted Conductor (conductor_url, conductorURL, ConductorURL), pointing dbosctl at it, upgrading it, or debugging a self-hosted Conductor that won't start or an application that won't connect to it.
license: MIT
metadata:
  author: dbos
  version: "0.1.0"
  organization: DBOS
  date: November 2026
  abstract: Guide for self-hosting DBOS Conductor and the DBOS Console. Covers trial and production license keys, a local Docker Compose setup, production deployments (managed Postgres, secrets, OAuth, TLS reverse proxies, high availability, Kubernetes), connecting applications and dbosctl, and fixing common startup and connection problems.
---

# Self-Hosting DBOS Conductor

Run DBOS Conductor and the DBOS Console on your own infrastructure, from a local trial to production. Conductor is the control plane DBOS applications connect to for workflow recovery, observability and management; the Console is its web UI. Both are stateless containers that keep all their state in Conductor's own Postgres database, which is separate from your applications' system databases. Conductor is out of band: applications keep running while it is unavailable, and reconnect when it returns.

Self-hosting is one of two ways to use Conductor, and which one is the user's choice. The other, DBOS-hosted Conductor, needs no deployment: register the application and create an API key at https://console.dbos.dev. Use this skill once the user has chosen to self-host.

## When to Apply

Reference these guidelines when:
- Trying out self-hosted Conductor locally
- Deploying Conductor to production on Kubernetes, ECS, Cloud Run, Nomad or VMs
- Getting a trial license key, or a production license key
- Configuring OAuth, TLS and a reverse proxy, or high availability for Conductor
- Connecting a DBOS application (Python, TypeScript, Go or Java) or dbosctl to a self-hosted Conductor
- Upgrading Conductor and the Console
- Debugging a Conductor that exits at startup, or an application that never shows as connected

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | License | CRITICAL | `license-` |
| 2 | Production | CRITICAL | `production-` |
| 3 | Local | HIGH | `local-` |
| 4 | Connect | HIGH | `connect-` |
| 5 | Troubleshooting | MEDIUM | `troubleshoot-` |

## Critical Rules

### Trial and Production

| | Trial | Production |
|---|---|---|
| License key | Trial key: self-serve, lasts a limited time | Production license key: the user requests it by emailing contact@dbos.dev |
| Runs on | Docker Compose on one machine | Any container platform (Kubernetes, ECS, Cloud Run, Nomad, VMs) |
| Conductor's Postgres | A container | Managed Postgres with a dedicated role |
| Authentication | None | OAuth for Conductor and the Console |
| Application URL | `ws://localhost:8090/` | `wss://` through a TLS reverse proxy |

### License Keys

Conductor won't start without a license key in `DBOS_CONDUCTOR_LICENSE_KEY`, which it validates against `https://cloud.dbos.dev` at startup. For a deployment without that network access, the user can email contact@dbos.dev to request an offline license key, which needs no network validation and can run in an air-gapped environment.

- **Trial keys** are for development, testing and evaluation only, and last a limited time. They are self-serve: the DBOS Console at https://console.dbos.dev/settings/license-key, `dbosctl license-key create` (with a DBOS-managed profile and a user login), or the DBOS MCP server's `create_license_key` tool. Organizations can create a limited number of trial keys; to request more, the user emails contact@dbos.dev. See `references/license-trial-key.md`.
- **Production** needs a production license key. Never deploy a trial key to production. Tell the user to email contact@dbos.dev to request one. See `references/license-production.md`.
- Never email DBOS, fill in a DBOS contact form, or otherwise contact DBOS on the user's behalf, for a license or anything else: tell the user whom to email and what to ask for.
- License keys are secrets: keep them in environment variables, a gitignored `.env` file or a secret store, never in committed files.

### Two Different Keys

The license key is not the key applications use. Never use one in place of the other.

| Key | Issued by | Used by | Set as |
|-----|-----------|---------|--------|
| License key | DBOS | The Conductor container | `DBOS_CONDUCTOR_LICENSE_KEY` |
| Conductor API key | Your self-hosted Conductor | Your DBOS application | `conductor_key` / `DBOS_CONDUCTOR_KEY` |

### Local Trial

With a trial key, run Conductor, the Console and Conductor's Postgres with the `docker-compose.yml` in `references/local-docker-compose.md`:

```bash
export DBOS_CONDUCTOR_LICENSE_KEY="<trial license key>"
docker compose pull
docker compose up -d --wait
docker compose logs conductor | grep "License activated"
```

The Console is at http://localhost and Conductor at http://localhost:8090; applications connect to `ws://localhost:8090/`.

### Production

A production deployment runs the same two images with:

1. A production license key, requested by the user from contact@dbos.dev, injected from a secret store
2. A managed Postgres for Conductor with a dedicated role, its URL (`DBOS__CONDUCTOR_DB_URL`) injected from the secret store
3. Conductor (port 8090) and the Console (port 8080) as stateless services with pinned image versions and `/healthz` probes; the Console's `DBOS_CONDUCTOR_URL` is Conductor's internal `host:port`
4. OAuth on Conductor and the Console before they are reachable from any untrusted network
5. A reverse proxy that terminates TLS, supports WebSockets and has long idle timeouts (3600 seconds); applications connect with a `wss://` URL and must trust its certificate
6. Outbound HTTPS from Conductor to `https://cloud.dbos.dev` for license validation, or, for an air-gapped environment, an offline license key that needs no network validation, which the user requests from contact@dbos.dev
7. For high availability, several Conductor replicas on a highly available Postgres, each advertising an address its peers can reach (`DBOS__ADVERTISE_ADDRESS`)

See `references/production-deployment.md`, `references/production-authentication.md`, `references/production-tls-proxy.md` and `references/production-high-availability.md`, and the [Kubernetes guide](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor-with-kubernetes) for a complete walkthrough.

### Connect an Application

1. In your self-hosted Console, not https://console.dbos.dev, register the application under the exact name in its DBOS configuration, and generate an API key. Or use dbosctl (see `references/connect-dbosctl.md`).
2. Install the DBOS enterprise package for the application's language.
3. Give the application the API key and your Conductor's WebSocket URL:

```python
# pip install "dbos[enterprise]"
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    "conductor_key": os.environ.get("DBOS_CONDUCTOR_KEY"),  # API key from your Conductor
    "conductor_url": os.environ.get("DBOS_CONDUCTOR_URL"),  # ws://localhost:8090/ or wss://...
}
```

Without `conductor_url`, the application connects to DBOS-hosted Conductor instead, where a self-hosted API key is not valid. See `references/connect-applications.md` for TypeScript, Go and Java. DBOS for Rust does not support Conductor yet.

### Key Constraints

- Conductor checks its license key only when it starts, and exits if the key is missing, invalid or expired, or if it can't reach `https://cloud.dbos.dev`. An offline license key, which the user can request from contact@dbos.dev, needs no network validation and works air-gapped. A replaced key takes effect when Conductor restarts
- Without OAuth, Conductor has no authentication: anyone who can reach it is an admin of its built-in `local` organization, and it doesn't verify application API keys
- An application's Conductor URL is a full WebSocket URL (`ws://` or `wss://`); the Console container's `DBOS_CONDUCTOR_URL` is a bare `host:port`
- Upgrade Conductor and the Console together, and never run a Console newer than Conductor. Upgrading from any version to any later one is safe and doesn't interrupt applications
- Conductor's database holds registered applications, API keys and settings; your applications' workflow data stays in their own system databases

## How to Use

Read individual rule files for detailed explanations and examples:

```
references/license-trial-key.md
references/license-production.md
references/production-deployment.md
references/production-authentication.md
references/production-tls-proxy.md
references/production-high-availability.md
references/local-docker-compose.md
references/connect-applications.md
references/connect-dbosctl.md
references/troubleshoot-startup.md
references/troubleshoot-connection.md
```

## References

- https://docs.dbos.dev/conductor/self-hosting/hosting-conductor
- https://docs.dbos.dev/conductor/self-hosting/hosting-conductor-with-kubernetes
- https://docs.dbos.dev/conductor/overview
- https://docs.dbos.dev/conductor/reference/dbosctl
