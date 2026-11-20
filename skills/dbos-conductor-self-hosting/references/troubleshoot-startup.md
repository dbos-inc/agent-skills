---
title: Diagnose Conductor Startup Failures
impact: MEDIUM
impactDescription: Most startup failures are a missing, invalid or expired license key, named in the logs
tags: troubleshooting, startup, exits, logs, invalid license key, expired, port, docker compose
---

## Diagnose Conductor Startup Failures

Conductor validates its license key as it starts and exits if validation fails: `docker compose up --wait` reports the Conductor container as exited or unhealthy, and on Kubernetes the pod goes into `CrashLoopBackOff`. Read Conductor's logs before changing anything: the last lines name the cause. On Kubernetes, use `kubectl logs deployment/conductor -n dbos --previous`.

**Incorrect (restarting without reading the logs):**

```bash
docker compose restart conductor
docker compose up -d --force-recreate
# Same key, same failure
```

**Correct (read why it exited):**

```bash
docker compose ps -a
docker compose logs conductor --tail 30
```

| Message | Cause | Fix |
|---------|-------|-----|
| `required variable DBOS_CONDUCTOR_LICENSE_KEY is missing a value` (from `docker compose`) | The key isn't exported in this shell or in `.env` | Export it, or add it to a `.env` file next to `docker-compose.yml` |
| `DBOS_CONDUCTOR_LICENSE_KEY is required` | Conductor received an empty key | Same as above |
| `Invalid license key` | The key is mistyped, expired (trial keys last a limited time), or isn't a license key (for example a Conductor API key) | For a trial, get a new trial key and recreate Conductor (see [local-docker-compose](local-docker-compose.md)). In production, check the secret holds the production key; if it is wrong or expired, the user emails contact@dbos.dev |
| `Failed to validate license key against server` | Conductor can't reach `https://cloud.dbos.dev` | Allow outbound HTTPS (on Kubernetes, a NAT gateway for private subnets and an egress rule); check proxies and VPNs. Where that isn't possible, the user can email contact@dbos.dev to request an offline license key, which needs no network validation and works air-gapped |
| `License server returned status ...` | DBOS returned an error | Retry shortly; if it persists, the user contacts DBOS support |
| Errors connecting to Postgres | `DBOS__CONDUCTOR_DB_URL` is wrong, Postgres isn't reachable, or the role lacks privileges | Locally, check `docker compose logs postgres` and `PGPASSWORD`; in production, the URL secret, network access to the database, and the role's grants (see [production-deployment](production-deployment.md)) |
| `port is already allocated` | Another process uses port 8090 or 80 | Stop it, or set `DBOS_CONSOLE_PORT` for the Console |

After a successful start, Conductor logs `Conductor License activated` with the license type (`free` for a trial key). A warning that `DBOS__ADVERTISE_ADDRESS` is unset is expected for a single Conductor and can be ignored.

Reference: [Self-Hosting Guide: Network Access](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#network-access)
