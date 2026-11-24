---
title: Deploy Conductor as Stateless Services on Managed Postgres
impact: CRITICAL
impactDescription: A production deployment that survives restarts, upgrades and node loss
tags: production, deployment, kubernetes, ecs, cloud run, postgres, database role, secrets, probes, upgrade, image version
---

## Deploy Conductor as Stateless Services on Managed Postgres

In production, run Conductor and the DBOS Console as stateless container services on any orchestrator: a Kubernetes Deployment (not a StatefulSet), an ECS service, Cloud Run, Nomad, or plain VMs. All of Conductor's state lives in its Postgres database, so use a managed Postgres with a dedicated database and role, inject the database URL and license key from a secret store, pin image versions, and probe `/healthz`. Don't run the local Docker Compose trial in production: it runs Postgres in a container, has no authentication, and uses a trial key.

**Incorrect (the local trial setup, promoted to production):**

```yaml
services:
  postgres:
    image: postgres:16            # Conductor's state in a container volume
  conductor:
    image: dbosdev/conductor:latest   # unpinned
    environment:
      DBOS_CONDUCTOR_LICENSE_KEY: ${DBOS_CONDUCTOR_LICENSE_KEY}   # a limited-time trial key
    ports:
      - "8090:8090"               # no OAuth, no TLS
```

**Correct (Kubernetes Deployment: pinned image, secrets, probes):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: conductor
  namespace: dbos
spec:
  replicas: 1
  selector:
    matchLabels: {app: conductor}
  template:
    metadata:
      labels: {app: conductor}
    spec:
      containers:
        - name: conductor
          image: dbosdev/conductor:<version>
          env:
            - name: DBOS__CONDUCTOR_DB_URL
              valueFrom: {secretKeyRef: {name: conductor-db, key: database-url}}
            - name: DBOS_CONDUCTOR_LICENSE_KEY
              valueFrom: {secretKeyRef: {name: conductor-license, key: license-key}}
            - name: DBOS__ADVERTISE_ADDRESS
              valueFrom: {fieldRef: {fieldPath: status.podIP}}
          ports:
            - containerPort: 8090
          readinessProbe:
            httpGet: {path: /healthz, port: 8090}
          livenessProbe:
            httpGet: {path: /healthz, port: 8090}
            initialDelaySeconds: 15
          resources:
            requests: {cpu: 250m, memory: 256Mi}
            limits: {cpu: "1", memory: 512Mi}
---
apiVersion: v1
kind: Service
metadata:
  name: conductor
  namespace: dbos
spec:
  selector: {app: conductor}
  ports:
    - port: 8090
      targetPort: 8090
```

Database:
- Use a managed Postgres (RDS, Aurora, Cloud SQL, Azure Database, ...), highly available if Conductor is. It is Conductor's own database, separate from your applications' system databases; Conductor never connects to those
- Give Conductor a dedicated database and role:

```sql
CREATE DATABASE dbos_conductor;
CREATE ROLE dbos_conductor_role WITH LOGIN PASSWORD '<password>';
GRANT ALL PRIVILEGES ON DATABASE dbos_conductor TO dbos_conductor_role;
\c dbos_conductor
GRANT ALL ON SCHEMA public TO dbos_conductor_role;
```

- `DBOS__CONDUCTOR_DB_URL` is then `postgresql://dbos_conductor_role:<password>@<host>:5432/dbos_conductor`, plus the TLS settings your provider requires

Console:
- Run it the same way, listening on port 8080, from `dbosdev/console:<version>`
- Set its `DBOS_CONDUCTOR_URL` to Conductor's internal `host:port`, for example `conductor.dbos.svc.cluster.local:8090`, not a WebSocket URL

Secrets and network:
- The database URL, the license key and applications' API keys are secrets: keep them in the platform's secret store, restrict which identities can read them, and never commit them. OAuth settings are not secrets
- Conductor needs outbound HTTPS to `https://cloud.dbos.dev` to validate its license at startup, unless it uses an offline license key, which needs no network validation and can run in an air-gapped environment (the user requests one from contact@dbos.dev). The Console must reach Conductor on port 8090. Applications need only outbound access to the reverse proxy
- Put both behind a TLS reverse proxy (see [production-tls-proxy](production-tls-proxy.md)) and enable OAuth before they are reachable from any untrusted network (see [production-authentication](production-authentication.md))

Upgrades and sizing:
- Upgrade by changing both image tags and restarting (`kubectl rollout restart`). Upgrading from any version to any later one is safe, and applications reconnect without interruption. Upgrade the Console and Conductor together, and never run a Console newer than Conductor
- Conductor is off the workflow execution path, so it needs few resources: a single instance can serve tens of thousands of application servers. Run several for availability, not load (see [production-high-availability](production-high-availability.md))

Reference: [Self-Hosting Guide: Deploying to Production](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#deploying-to-production), [Kubernetes guide](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor-with-kubernetes)
