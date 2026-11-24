---
title: Advertise Each Conductor Replica's Address for High Availability
impact: MEDIUM
impactDescription: Without a routable advertised address, requests for executors on other replicas fail
tags: production, high availability, ha, replicas, DBOS__ADVERTISE_ADDRESS, DBOS__CONDUCTOR_PORT, peer, load balancer, multi-az
---

## Advertise Each Conductor Replica's Address for High Availability

For fault tolerance, run several Conductor instances on the same Postgres behind a load balancer, across availability zones. Each application process's WebSocket lands on one instance, which owns it. A request for that process (from the Console, say) can arrive at any instance, which looks up the owner in Postgres and forwards the request to it directly. So every instance must advertise an address its peers can reach, in `DBOS__ADVERTISE_ADDRESS`. It defaults to `127.0.0.1`, which works only for a single instance.

**Incorrect (replicas without an advertised address):**

```yaml
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: conductor
          image: dbosdev/conductor:<version>
          # DBOS__ADVERTISE_ADDRESS unset: every replica advertises 127.0.0.1
```

**Correct (each pod advertises its own IP):**

```yaml
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: conductor
          image: dbosdev/conductor:<version>
          env:
            - name: DBOS__ADVERTISE_ADDRESS
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
```

Notes:
- `DBOS__ADVERTISE_ADDRESS` is a hostname or IP without a port. Peers connect to it on the Conductor port, 8090 unless `DBOS__CONDUCTOR_PORT` changes it
- Instances must reach each other directly on that port, not only through the load balancer: allow Conductor-to-Conductor traffic in network policies and security groups
- Use a highly available Postgres too, such as RDS or Aurora across availability zones: it holds all of Conductor's state
- Replicas add availability, not capacity: one instance can serve tens of thousands of application servers
- A single instance logs a warning that `DBOS__ADVERTISE_ADDRESS` is unset; it is harmless there

Reference: [Self-Hosting Guide: High Availability](https://docs.dbos.dev/conductor/self-hosting/hosting-conductor#high-availability)
