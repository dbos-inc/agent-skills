---
title: Put Conductor Behind a TLS Proxy That Keeps WebSockets Open
impact: HIGH
impactDescription: Short proxy timeouts drop application connections; an untrusted certificate stops them connecting at all
tags: production, tls, https, wss, reverse proxy, ingress, nginx, load balancer, websocket, idle timeout, certificate
---

## Put Conductor Behind a TLS Proxy That Keeps WebSockets Open

Conductor and the Console serve plain HTTP. In production, put them behind a reverse proxy or load balancer (Nginx, an ingress controller, a cloud load balancer) that terminates TLS and supports WebSockets, since each application process keeps a long-lived WebSocket open to Conductor. Raise idle timeouts to about 3600 seconds on every layer: short defaults (often 60 seconds) drop connections that go quiet, for example during a network hiccup. Applications then connect with a `wss://` URL and must trust the proxy's certificate.

**Incorrect (default timeouts, plain WebSocket URL):**

```yaml
# Ingress with default timeouts (60 seconds in ingress-nginx): quiet connections get dropped
metadata:
  annotations: {}
```

```bash
DBOS_CONDUCTOR_URL=ws://conductor.example.com:8090/   # unencrypted, bypasses the proxy
```

**Correct (ingress-nginx routing one hostname, long timeouts, wss URL):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dbos-ingress
  namespace: dbos
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [conductor.example.com]
      secretName: dbos-tls
  rules:
    - host: conductor.example.com
      http:
        paths:
          - path: /conductor-api(/|$)(.*)   # Conductor, prefix stripped
            pathType: ImplementationSpecific
            backend: {service: {name: conductor, port: {number: 8090}}}
          - path: /()(.*)                    # everything else: the Console
            pathType: ImplementationSpecific
            backend: {service: {name: console, port: {number: 80}}}
```

```bash
DBOS_CONDUCTOR_URL=wss://conductor.example.com/conductor-api
```

Routing:
- Either route a path prefix to Conductor (stripping it) and everything else to the Console, as above, or give each its own hostname. The application URL and dbosctl's `--url` include the prefix
- The `host` in `tls` and `rules` must match, or Nginx serves its default certificate
- Set the same long idle timeout on any load balancer in front of the proxy, for example the AWS ELB annotation `service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"` on the ingress controller's Service. The SDK pings and reconnects on its own, but short timeouts cause needless disconnects

Certificates:
- Applications verify the proxy's certificate against their default trust store. Use a certificate from a public certificate authority for a domain you control (for example with cert-manager)
- With a private or self-signed certificate, make every application host trust it: `SSL_CERT_FILE` (Python, Go), `NODE_EXTRA_CA_CERTS` (TypeScript), or `-Djavax.net.ssl.trustStore` and `-Djavax.net.ssl.trustStorePassword` (Java)
- An application that doesn't trust the certificate logs the failure and keeps retrying, but never connects

Reference: [Self-Hosting Guide: Reverse Proxy and TLS](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#reverse-proxy-and-tls), [Kubernetes guide: Ingress](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor-with-kubernetes#ingress)
