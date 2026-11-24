---
title: Get a Trial License Key for Development and Evaluation
impact: CRITICAL
impactDescription: Conductor exits at startup without a valid license key
tags: license, license key, trial, DBOS_CONDUCTOR_LICENSE_KEY, dbosctl, license-key, mcp, create_license_key
---

## Get a Trial License Key for Development and Evaluation

A trial license key lets you run self-hosted Conductor for development, testing and evaluation. It is self-serve, free, and lasts a limited time. Trial keys are for trial use only: a production deployment needs a production license key (see [license-production](license-production.md)). Pass the key to the Conductor container as `DBOS_CONDUCTOR_LICENSE_KEY`; it is not the Conductor API key your applications use.

**Incorrect (license key committed in the Compose file, or given to the application):**

```yaml
services:
  conductor:
    image: dbosdev/conductor:latest
    environment:
      # A secret, committed to the repository
      DBOS_CONDUCTOR_LICENSE_KEY: 3f6c2a1e-....
```

```python
config: DBOSConfig = {
    "name": "my-app",
    # Wrong key: applications use a Conductor API key from your self-hosted Conductor
    "conductor_key": os.environ.get("DBOS_CONDUCTOR_LICENSE_KEY"),
}
```

**Correct (trial key from DBOS, exported for Docker Compose):**

```bash
# Option 1: create it in the DBOS Console at https://console.dbos.dev/settings/license-key, then
export DBOS_CONDUCTOR_LICENSE_KEY="<trial license key>"

# Option 2: create it with dbosctl, logged in as a user on a DBOS-managed profile
dbosctl config set managed --managed
dbosctl login --profile managed
export DBOS_CONDUCTOR_LICENSE_KEY="$(dbosctl license-key create --profile managed)"
```

```yaml
services:
  conductor:
    environment:
      DBOS_CONDUCTOR_LICENSE_KEY: ${DBOS_CONDUCTOR_LICENSE_KEY:?set DBOS_CONDUCTOR_LICENSE_KEY to your license key}
```

Option 3: if the DBOS MCP server (`dbos-mcp`) is installed, call its `login` and `login_complete` tools, then `create_license_key`. It returns the key with its type and expiry.

Ways to get a trial key:
- `dbosctl license-key create` prints only the key on stdout and its expiry on stderr; `-o json` prints the full response. It needs a profile created with `--managed` and a user login from `dbosctl login`: a `dbos_` API key in `DBOS_TOKEN` is refused. `--org` creates the key for another organization you belong to
- All three ways create the key for a DBOS organization you are a member of

Limits:
- A trial key lasts a limited time. Conductor checks the key only when it starts, so after expiry Conductor won't start again until it has a new key
- Organizations can create a limited number of trial keys. To request more, the user emails contact@dbos.dev
- Trial keys are for development, testing and evaluation. For production, the user requests a production license key from contact@dbos.dev

Handling the key:
- The key is shown once. Treat it as a secret: keep it in an environment variable or a gitignored `.env` file next to `docker-compose.yml` (Docker Compose reads `.env` automatically), and don't print it in logs or commit it
- Never email DBOS or contact it on the user's behalf; tell the user whom to email

Reference: [Self-Hosting Guide: Licensing](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#licensing)
