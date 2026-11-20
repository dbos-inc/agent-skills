---
title: Point dbosctl at Self-Hosted Conductor
impact: MEDIUM
impactDescription: Register applications, create API keys and manage workflows from a terminal
tags: dbosctl, cli, profile, config set, url, register, api-key, local
---

## Point dbosctl at Self-Hosted Conductor

dbosctl talks to one Conductor per profile. A DBOS-managed profile (`--managed`) targets DBOS-hosted Conductor; for a self-hosted Conductor, create a profile with `--url`. Without OAuth (the local trial) there is nothing to log in to: every request runs as the built-in `local` organization. With OAuth (production), the profile also names your OIDC issuer and client, and you log in. Keep a separate managed profile for creating trial license keys, which come from DBOS.

**Incorrect (the managed profile for self-hosted work):**

```bash
# Registers the application and creates the key on DBOS-hosted Conductor,
# so the self-hosted Conductor never sees them
dbosctl app register my-app --profile managed
dbosctl api-key create my-app-key --profile managed
```

**Correct (a profile for the self-hosted Conductor):**

```bash
# Local trial, no authentication
dbosctl config set local --url http://localhost:8090
dbosctl whoami --profile local                 # organization "local"
dbosctl app register my-app --profile local    # the name in the application's DBOS config
export DBOS_CONDUCTOR_KEY="$(dbosctl api-key create my-app-key --profile local)"
dbosctl app list --profile local

# Production, with OAuth: the HTTPS URL routed to Conductor, plus your OIDC issuer and CLI client
dbosctl config set prod --url https://conductor.example.com/conductor-api \
  --issuer https://your-oauth-provider.com/ --client-id <CLI client ID>
dbosctl login --profile prod
dbosctl app list --profile prod
```

Notes:
- Install dbosctl with `curl -sSfL https://raw.githubusercontent.com/dbos-inc/dbos-ctl/main/install.sh | sh`, or `go install github.com/dbos-inc/dbos-ctl/cmd/dbosctl@latest`
- `dbosctl config use local` makes the profile the default, so `--profile local` can be dropped
- `api-key create` prints the key alone on stdout and is the only time it is shown; `--app` limits it to some applications
- The `--url` is Conductor's HTTP(S) address (`http://localhost:8090`, or your proxy's `https://` URL with its Conductor prefix), not the `ws://` or `wss://` URL applications use
- The OIDC client for dbosctl must allow the device authorization flow (see [production-authentication](production-authentication.md))
- Creating a trial license key needs the managed profile, not this one: `dbosctl license-key create --profile managed` (see [license-trial-key](license-trial-key.md)). Production license keys don't come from dbosctl: the user requests one from contact@dbos.dev

Reference: [dbosctl](https://docs.dbos.dev/conductor/reference/dbosctl)
