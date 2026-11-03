# Problem 1 — Service availability

## Cause and risk

The API has one replica, no probes, resource sizing, or graceful shutdown, so a restart, rollout, or drain can remove all API capacity. The two GKE nodes share one zone; Cloud SQL is single-zone, but the assignment does not say whether it is the same zone.

## Safer releases and node maintenance

The [API Deployment and PDB example](snippets/api-deployment.yaml) assumes capacity across multiple zones; replace placeholders and size resources from measurements. Replace `REPLACE_WITH_IMMUTABLE_TAG` with a unique tag for the built release (for example, the commit plus build ID) that is never reused or overwritten. Prefer deploying by registry digest (`image@sha256:...`) when available, so each rollout uses the exact image that was validated. Three replicas with topology spread reduce single-pod/node risk. `maxUnavailable: 0` and `maxSurge: 1` retain ready capacity during rollout; startup/readiness/liveness probes separate slow start, traffic eligibility, and stuck processes. Keep liveness independent of PostgreSQL to avoid restart storms. Requests/limits guide placement and cap consumption. A 60-second termination grace period requires the API to handle `SIGTERM` and drain requests.

The PDB's `minAvailable: 2` limits voluntary eviction during maintenance; it cannot prevent node/zone failures or create capacity. Apply the same availability goal to **frontend and worker**, not just the API: run multiple spread frontend replicas; ensure enough worker replicas/capacity in surviving zones, and have workers stop gracefully and safely retry or release in-flight work. The queue/worker protocol is not specified, so validate its acknowledgment and retry semantics.

### Safe rollout and verification

In a non-production cluster, test slow startup, readiness failure, and an unresponsive process; confirm only ready pods receive traffic and liveness restarts a stuck process. Send requests during rollout and drain one node; verify service continues and the PDB blocks eviction that would leave fewer than two API pods. Confirm worker shutdown/retry does not lose or duplicate work. These are proposed tests; none were run.

## Zone outage — separate infrastructure change

**Zone outage (infrastructure, separate from rollout):** losing the GKE node's zone removes the current cluster's compute; losing Cloud SQL's zone removes its database. Use a regional GKE Standard cluster with workers across zones and enough surviving capacity for all tiers. Enable Cloud SQL regional HA with primary/standby in separate zones and verify client reconnect/retry behavior. Expect brief rescheduling/failover interruption; this does not cover regional outage or replace backups. The corrected topology is in [architecture.md](architecture.md).
