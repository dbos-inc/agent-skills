# dbos-conductor-self-hosting

> **Note:** `CLAUDE.md` is a symlink to this file.

## Overview

Self-host DBOS Conductor and the DBOS Console, from a local trial to a production deployment. Use this skill when deploying Conductor on your own infrastructure (Docker Compose, Kubernetes or another container platform), getting a trial license key or a production license key (DBOS_CONDUCTOR_LICENSE_KEY), configuring OAuth authentication, TLS, a reverse proxy or high availability for Conductor, connecting DBOS applications to a self-hosted Conductor (conductor_url, conductorURL, ConductorURL), pointing dbosctl at it, upgrading it, or debugging a self-hosted Conductor that won't start or an application that won't connect to it.

## Structure

```
dbos-conductor-self-hosting/
  SKILL.md       # Main skill file - read this first
  AGENTS.md      # This navigation guide
  CLAUDE.md      # Symlink to AGENTS.md
  references/    # Detailed reference files
```

## Usage

1. Read `SKILL.md` for the main skill instructions
2. Browse `references/` for detailed documentation on specific topics
3. Reference files are loaded on-demand - read only what you need

## Reference Categories

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | License | CRITICAL | `license-` |
| 2 | Production | CRITICAL | `production-` |
| 3 | Local | HIGH | `local-` |
| 4 | Connect | HIGH | `connect-` |
| 5 | Troubleshooting | MEDIUM | `troubleshoot-` |

Reference files are named `{prefix}-{topic}.md` (e.g., `query-missing-indexes.md`).

## Available References

**Connect** (`connect-`):
- `references/connect-applications.md`
- `references/connect-dbosctl.md`

**License** (`license-`):
- `references/license-production.md`
- `references/license-trial-key.md`

**Local** (`local-`):
- `references/local-docker-compose.md`

**Production** (`production-`):
- `references/production-authentication.md`
- `references/production-deployment.md`
- `references/production-high-availability.md`
- `references/production-tls-proxy.md`

**Troubleshooting** (`troubleshoot-`):
- `references/troubleshoot-connection.md`
- `references/troubleshoot-startup.md`

---

*11 reference files across 5 categories*