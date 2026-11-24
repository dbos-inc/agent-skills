# Section Definitions

This file defines the rule categories for self-hosting DBOS Conductor. Rules are automatically assigned to sections based on their filename prefix.

---

## 1. License (license)
**Impact:** CRITICAL
**Description:** Self-serve trial license keys for development and evaluation, production license keys (requested by the user from contact@dbos.dev), and passing a key to Conductor. Conductor does not start without one.

## 2. Production (production)
**Impact:** CRITICAL
**Description:** Running Conductor and the Console in production: managed Postgres, secrets, images and probes, upgrades, OAuth authentication, TLS reverse proxies, and high availability.

## 3. Local (local)
**Impact:** HIGH
**Description:** Running Conductor, the Console and Conductor's Postgres on one machine with Docker Compose for a trial: starting, checking, upgrading, stopping, and replacing an expired trial key.

## 4. Connect (connect)
**Impact:** HIGH
**Description:** Registering applications and creating Conductor API keys on a self-hosted Conductor, configuring DBOS applications in each language to connect to it, and pointing dbosctl at it.

## 5. Troubleshooting (troubleshoot)
**Impact:** MEDIUM
**Description:** Diagnosing a Conductor that exits at startup and an application that never shows as connected.
