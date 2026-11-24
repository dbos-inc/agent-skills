---
title: Enable OAuth Before Exposing Conductor
impact: CRITICAL
impactDescription: Without OAuth, anyone who can reach Conductor has full admin access
tags: production, security, oauth, oidc, sso, authentication, okta, auth0, google, keycloak, dbosctl login
---

## Enable OAuth Before Exposing Conductor

Without OAuth, Conductor authenticates nothing: every request runs as an admin of the built-in `local` organization, and Conductor doesn't verify the API keys applications connect with. Anyone who can reach it can register applications, create API keys, and cancel, resume, fork or delete workflows. Configure OAuth with any OIDC provider (Okta, Auth0, Google, Keycloak, ...) on both Conductor and the Console before either is reachable from an untrusted network. OAuth also enables user and organization management in the Console.

**Incorrect (Conductor reachable without authentication):**

```yaml
# No DBOS_OAUTH_* settings, published to the network
conductor:
  image: dbosdev/conductor:<version>
  ports:
    - "8090:8090"
```

**Correct (OAuth on Conductor and on the Console):**

```yaml
conductor:
  environment:
    DBOS_OAUTH_ENABLED: "true"
    DBOS_OAUTH_ISSUER: "https://your-oauth-provider.com/"
    DBOS_OAUTH_AUDIENCE: "your-api-audience"
console:
  environment:
    DBOS_OAUTH_ENABLED: "true"
    DBOS_OAUTH_AUTHORIZATION_URL: "https://your-oauth-provider.com/.../authorize"
    DBOS_OAUTH_TOKEN_URL: "https://your-oauth-provider.com/.../token"
    DBOS_OAUTH_CLIENT_ID: "your-client-id"
    DBOS_OAUTH_SCOPE: "openid profile email"
    DBOS_OAUTH_USERINFO_URL: "https://your-oauth-provider.com/.../userinfo"
    DBOS_OAUTH_LOGOUT_URL: "https://your-oauth-provider.com/.../logout"
    DBOS_OAUTH_AUDIENCE: "your-api-audience"
```

Provider setup:
- Register the Console as an application and Conductor as an API (the audience)
- Allow `https://your-domain/oauth/callback` as a callback URL and `https://your-domain` as a web origin, where `your-domain` is the Console's public address
- Allow the authorization code grant with PKCE, and the scopes `openid profile email`
- The values above are the provider's endpoints and client ID. None are secrets, so they can go directly in the deployment configuration

Clients:
- With OAuth on, users sign in to the Console through your provider, and applications must present a valid Conductor API key generated in the Console
- dbosctl signs in through the same provider with the device authorization flow, so register a client for it that allows that grant: `dbosctl config set prod --url https://your-domain/conductor-api --issuer https://your-oauth-provider.com/ --client-id <CLI client ID>`, then `dbosctl login --profile prod` (see [connect-dbosctl](connect-dbosctl.md))

Reference: [Self-Hosting Guide: Security](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#security)
