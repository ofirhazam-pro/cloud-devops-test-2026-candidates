# Problem 1 — Service availability

## Cause and risk

The API has one replica, no probes, resource sizing, or graceful shutdown, so a restart, rollout, or drain can remove all API capacity. The two GKE nodes share one zone; Cloud SQL is single-zone, but the assignment does not say whether it is the same zone.

## Safer releases and node maintenance

The [API Deployment and PDB example](snippets/api-deployment.yaml) shows the API changes. Replace `REPLACE_WITH_IMMUTABLE_TAG` with a unique tag for each image release, and size resource values from measurements.

| Change | Reason |
|---|---|
| Three API replicas, spread across zones and nodes | Keeps API capacity when a pod or node is unavailable. Requires eligible nodes and capacity in the target zones. |
| Startup probe | Gives a slow-starting API time to initialize before liveness/readiness checks take effect. |
| Readiness probe | Removes starting or unhealthy pods from Service traffic until they can serve requests. |
| Liveness probe | Restarts a stuck process. Check process health, not PostgreSQL, to avoid restart storms during a database outage. |
| `maxUnavailable: 0`, `maxSurge: 1` | Keeps existing ready capacity while a replacement pod starts during rollout. |
| Resource requests and limits | Requests inform scheduling; limits cap container resource use. Tune values from observed workload. |
| `terminationGracePeriodSeconds: 60` and graceful `SIGTERM` handling | Lets the API stop accepting requests and drain in-flight work before Kubernetes terminates it. |
| PDB with `minAvailable: 2` | Limits voluntary eviction during maintenance so at least two API pods remain available. It cannot prevent involuntary failures or provide missing capacity. |
| Multiple, spread frontend replicas and sufficient worker replicas/capacity | Keeps the whole service working if a node/zone is lost. Workers should stop gracefully and safely retry/release in-flight work; validate the unspecified queue acknowledgment semantics. |

### Safe rollout and verification

In a non-production cluster, test slow startup, readiness failure, and an unresponsive process; confirm only ready pods receive traffic and liveness restarts a stuck process. Send requests during rollout and drain one node; verify service continues and the PDB blocks eviction that would leave fewer than two API pods. Confirm worker shutdown/retry does not lose or duplicate work. These are proposed tests; none were run.

## Zone outage — separate infrastructure change

**Zone outage (infrastructure, separate from rollout):** losing the GKE node's zone removes the current cluster's compute; losing Cloud SQL's zone removes its database. Use a regional GKE Standard cluster with workers across zones and enough surviving capacity for all tiers. Enable Cloud SQL regional HA with primary and standby in separate zones, and verify client reconnect/retry behavior. Expect brief rescheduling/failover interruption; this does not cover a regional outage or replace backups. See the [corrected topology](architecture.md).
