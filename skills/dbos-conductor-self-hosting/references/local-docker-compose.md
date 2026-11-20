---
title: Run a Local Trial with Docker Compose
impact: HIGH
impactDescription: A working Conductor and Console on one machine, on current images
tags: docker, docker compose, local, trial, conductor, console, postgres, upgrade, healthz, expired key
---

## Run a Local Trial with Docker Compose

For a trial, run Conductor, the DBOS Console and Conductor's Postgres on one machine with Docker Compose and a trial license key. Pull the images before starting: Docker Compose doesn't update an image that is already on the machine, so a plain `docker compose up` can run months-old Conductor and Console images. Wait for the health checks, then confirm the license activated. This setup has no authentication and runs Postgres in a container, so it is not a production deployment (see [production-deployment](production-deployment.md)).

**Incorrect (stale local images, no check that Conductor started):**

```bash
docker compose up -d
# Reuses whatever dbosdev/conductor and dbosdev/console images are already local,
# and returns before Conductor has validated its license (or exited)
```

**Correct (current images, wait for health, confirm the license):**

```bash
export DBOS_CONDUCTOR_LICENSE_KEY="<trial license key>"   # or put it in a gitignored .env
docker compose pull
docker compose up -d --wait
docker compose logs conductor | grep "License activated"
curl -f http://localhost:8090/healthz
```

The `docker-compose.yml`:

```yaml
# DBOS Conductor, the DBOS Console, and Conductor's Postgres database, for a local trial.
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${PGPASSWORD:-dbos}
      POSTGRES_DB: dbos_conductor
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      retries: 30
  conductor:
    image: dbosdev/conductor:latest
    environment:
      DBOS__CONDUCTOR_DB_URL: postgresql://postgres:${PGPASSWORD:-dbos}@postgres:5432/dbos_conductor?sslmode=disable
      DBOS_CONDUCTOR_LICENSE_KEY: ${DBOS_CONDUCTOR_LICENSE_KEY:?set DBOS_CONDUCTOR_LICENSE_KEY to your license key}
    ports:
      - "8090:8090"
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8090/healthz"]
      interval: 5s
      retries: 12
  console:
    image: dbosdev/console:latest
    environment:
      DBOS_CONDUCTOR_URL: conductor:8090
    ports:
      - '${DBOS_CONSOLE_PORT:-80}:8080'
    depends_on:
      conductor:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 5s
      retries: 12
volumes:
  postgres_data:
```

A healthy start logs `Conductor License activated` with the license type, then `Granted all entitlements to self-hosted organization`. The Console is at http://localhost, and applications on the host connect to `ws://localhost:8090/`.

Ports:
- The file publishes Conductor on port 8090 and the Console on port 80. Without OAuth they have no authentication (see [production-authentication](production-authentication.md))
- If port 80 is taken, set `DBOS_CONSOLE_PORT` (for example `DBOS_CONSOLE_PORT=8081`) and open http://localhost:8081
- An application running in a container reaches Conductor at `ws://conductor:8090/` as a service in the same Compose project, not at `localhost`

Upgrading:
- Pull and recreate: `docker compose pull && docker compose up -d --wait`. Applications reconnect on their own
- To pin versions instead of `latest`, tag each image with a release from Docker Hub (for example `dbosdev/conductor:0.22.0` and `dbosdev/console:0.6.43`). The two are versioned separately, so pick a Console release no newer than the Conductor release

Replacing an expired trial key:

```bash
export DBOS_CONDUCTOR_LICENSE_KEY="<new license key>"
docker compose up -d --wait conductor   # recreates Conductor with the new key
```

Registered applications, API keys and settings are in Postgres, so they survive the restart.

Stopping:
- `docker compose stop` stops the containers; `docker compose up -d --wait` resumes them
- `docker compose down` removes the containers but keeps Conductor's database volume
- `docker compose down -v` also deletes Conductor's database: registered applications, API keys and settings. Your applications' own system databases are separate and unaffected

Reference: [Self-Hosting Guide: Trying It Locally with Docker Compose](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#trying-it-locally-with-docker-compose)
