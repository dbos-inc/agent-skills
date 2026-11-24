---
title: Use a Production License Key in Production
impact: CRITICAL
impactDescription: A trial key in production expires and then stops Conductor from starting, and is not licensed for production use
tags: license, production license, license key, contact, secret, rotation, DBOS_CONDUCTOR_LICENSE_KEY
---

## Use a Production License Key in Production

Commercial or production use of self-hosted Conductor needs a production license key from DBOS. The user requests one by emailing contact@dbos.dev. Don't send that email, fill in a contact form, or otherwise contact DBOS yourself: tell the user to email contact@dbos.dev to request a production license key for self-hosted Conductor, and continue once they have it. Self-serve trial keys are never a substitute: they are licensed for evaluation only, and they last a limited time, after which Conductor won't start on its next restart or rollout.

**Incorrect (a trial key in a production deployment):**

```bash
# A limited-time trial key, created with dbosctl and baked into a production secret
kubectl create secret generic conductor-license -n dbos \
  --from-literal=license-key="$(dbosctl license-key create --profile managed)"
```

**Correct (the production license key, from DBOS, in the secret store):**

```bash
# The user requested the key from contact@dbos.dev and provides it, for example:
read -rs CONDUCTOR_LICENSE_KEY   # paste the production license key, not echoed
kubectl create secret generic conductor-license -n dbos \
  --from-literal=license-key="$CONDUCTOR_LICENSE_KEY"
```

```yaml
# In the Conductor Deployment
env:
  - name: DBOS_CONDUCTOR_LICENSE_KEY
    valueFrom:
      secretKeyRef:
        name: conductor-license
        key: license-key
```

Notes:
- Keep the license key in your platform's secret store (Kubernetes Secrets with Sealed Secrets or SOPS, AWS Secrets Manager, Vault) and inject it as `DBOS_CONDUCTOR_LICENSE_KEY`. Never commit it
- Conductor validates the key against `https://cloud.dbos.dev` at startup and exits if it can't, so production hosts need outbound HTTPS (for example a NAT gateway for private subnets). Alternatively, the user can email contact@dbos.dev to request an offline license key, which needs no network validation and can run in an air-gapped environment
- To replace a key, update the secret and restart Conductor (for example `kubectl rollout restart deployment/conductor -n dbos`). Applications reconnect on their own
- A trial key is fine for evaluating a production-style deployment, such as a staging cluster, as long as it is replaced before production use

Reference: [Self-Hosting Guide: Licensing](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#licensing)
