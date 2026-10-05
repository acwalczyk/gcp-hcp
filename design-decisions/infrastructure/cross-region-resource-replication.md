# Cross-Region Resource Replication

***Scope***: GCP-HCP

**Date**: 2026-08-25

## Decision

We will implement cross-region replication for selected Gecko resources using a single-leader, follower-mirror model over Google Cloud Pub/Sub. Exactly one configured leader region accepts writes for replicated resource types. All non-leader regions keep read-only local mirrors of the leader's data and reject direct write attempts. The CLI discovers a stable global authorization endpoint through the existing [static endpoint discovery manifest](../networking/endpoint-discovery.md#global-authorization-endpoint). Infrastructure routes it to the designated ready leader, so customers need no explicit leader-region configuration. Other resources retain regional endpoints. Leadership changes are manual and controlled through GitOps/Argo/Helm configuration. Each promotion increments a leadership generation and requires configuration rollouts across regional API servers and replication controllers.

Replication is unidirectional: the leader publishes create, update, delete, bootstrap, and inventory events; followers subscribe to a leader-only data topic and apply only events from the configured leader and leadership generation. Followers send `RESYNC_REQUEST` on a separate control topic. Publishers preserve the leader API's resource versions; periodic LIST reconciliation repairs missed events. Resource changes remain individual events, and inventories are byte-bounded pages of keys and expected object versions with a consistent API snapshot boundary; no message contains the whole dataset. Followers never publish replicated resource changes and never become authoritative unless operators manually promote them by changing the leadership configuration. The initial use case is replicating authorization Roles and RoleBindings across regional Gecko instances, but the mechanism is designed to support any configured API resource type.

The design intentionally trades independent regional write authority for a simpler consistency model. Reads and authorization decisions can be served from local follower mirrors, but writes are available only through the leader until a manual failover promotes another region.

## Context

- **Problem Statement**: Gecko operates multiple regional instances, each with its own database. Certain resource types, starting with authorization Roles and RoleBindings, must be consistent across regions so that access granted through Gecko is available everywhere. The previous multi-region authoritative design allowed any region to write and relied on conflict resolution. We now want one authoritative write region to avoid concurrent-write conflicts and make behavior deterministic.
- **Constraints**: Writes for replicated resource types must be accepted by only one leader region at a time. Non-leader regions must reject direct public API writes. The CLI must resolve the global authorization URL through existing endpoint discovery; global-host requests, including reads, must be rejected outside the designated ready leader. Replication must remain asynchronous so follower read availability does not depend on cross-region calls. Leadership changes must be manual and auditable through GitOps/Argo/Helm. The private API must restrict human/operator writes in follower regions while preserving replication controller write permission so leader data can be applied locally. The mechanism must support namespaced and cluster-scoped resources, consume standard Kubernetes-style LIST/WATCH with correctly versioned mutations/deletes and consistent pagination, keep source version tracking internal to replication, and support local development via the Pub/Sub emulator.
- **Assumptions**: Eventual consistency is acceptable for follower reads and authorization cache updates. Replicated resource writes are infrequent administrative operations, so manual failover with minute-scale recovery is acceptable. There should always be exactly one configured leader region. PlatformRoles are system-defined, cluster-scoped resources deployed identically to all regions via Helm and do not need replication.

## Alternatives Considered

1. **Single-leader Pub/Sub replication with read-only followers**: One configured leader region accepts writes and publishes all replicated resource changes. Followers reject writes, apply only leader-origin events, and maintain local mirrors for reads and authorization decisions. Leadership changes are manual through GitOps/Argo/Helm.
2. **Multi-region authoritative Pub/Sub replication**: Every region accepts writes, publishes locally-owned resources, and resolves concurrent changes with last-writer-wins timestamps. This maximizes regional write availability but risks silent conflict loss and requires ownership-transfer semantics.
3. **Shared global Spanner instance**: All regions use one global database for replicated resource types.
4. **No replication**: Each region manages its own resources independently. Cross-region consistency is handled manually or by external orchestration.

## Decision Rationale

* **Justification**: The single-leader model provides a clear source of truth for globally replicated Gecko control data. It removes normal-path concurrent writes, ownership transfer, and last-writer-wins conflict resolution. Static discovery of the global authorization URL gives users a stable endpoint across leadership changes, while follower write rejection protects against forced or stale requests. Manual leadership changes through GitOps/Argo/Helm are auditable and safer than automatic cross-region leader election for this low-write-volume use case. Pub/Sub keeps replication asynchronous and allows followers to continue serving local reads when the leader or transport is unavailable.
* **Evidence**: The existing proof-of-concept showed that Pub/Sub fan-out can replicate Roles and RoleBindings across regions and that namespace auto-creation works for non-primary regions. The new model keeps the proven transport and receiver mechanics but removes multi-writer ownership transfer and conflict-resolution complexity.
* **Comparison**: Multi-region authoritative replication provides better write locality but creates ambiguous behavior when the same object is changed in multiple regions. Shared global Spanner simplifies consistency at the storage layer but introduces a cross-region request-path dependency and conflicts with the broader regional architecture. No replication pushes consistency management to external systems and does not scale to user-defined resources that must be available across regions.

## Consequences

### Positive

* Clear write authority: exactly one leader region owns replicated resource writes.
* Simpler consistency model: no normal-path multi-writer conflict resolution or ownership transfer.
* Followers provide local mirrored reads and authorization decisions.
* Forced writes to read-only regions are rejected deterministically.
* CLI Role/RoleBinding reads and writes use the discovered global endpoint for leader-local resource consistency, without promising immediate convergence of regional Cedar caches. Customer configuration remains unchanged during failover.
* Replication uses the API's existing resource-version and LIST/WATCH model. Source versions are distinct from follower-local API versions; replication tracking does not change Kubernetes metadata or deletion semantics.
* Namespace auto-creation in follower regions ensures replicated namespaced resources have valid target namespaces.
* Manual leadership changes are auditable through GitOps/Argo/Helm.
* Leader inventory reconciliation can repair dropped delete events and follower drift without lease expiration.

### Negative

* A leader outage makes writes unavailable until operators manually promote another region.
* Cross-region write latency increases for users far from the leader.
* Follower reads can lag behind the leader because replication is asynchronous.
* Unplanned failover can lose committed-but-unpublished writes and published-but-unapplied messages. Last-event age cannot bound that loss: a recent event can arrive before an older revocation. Operators must explicitly accept an unknown loss window when the old leader is unavailable. Planned transfers fence writes, drain accepted mutations, and verify a final per-kind resync before promotion.
* Failover has an RTO bounded by detection, fencing, GitOps/Argo/Helm rollout, and validation time.
* GitOps rollout skew can create split-brain risk if more than one region accepts writes; fencing and alerts are required.
* Pub/Sub remains at-least-once and unordered. Receivers persist source versions, observed-through snapshot boundaries, and internal deletion records. Periodic complete snapshots repair drift even when source objects are unchanged. Publishers need no separate sequence counter or ownership protocol.
* Gecko must first close the documented resource-version, delete-event, and LIST/WATCH compatibility gaps in each backend. These are implementation prerequisites, not guarantees of the current proof-of-concept.
* Eventual consistency does not bound authorization revocation delay during outages. Recovery requires successful reconciliation and local Cedar reloads.
* This is an intentional exception to the regional independence architecture for globally consistent Gecko control data.

## Cross-Cutting Concerns

### Reliability:

* **Scalability**: Each replicated resource type adds one leader-side API LIST/WATCH loop. Followers process all replicated resource types from their regional Pub/Sub subscription. Inventory pages add traffic proportional to object count without requiring a single message to fit the dataset. Encoded resource events and inventory pages have an application size limit below Pub/Sub limits; replicated-resource write validation rejects oversized resources. Incomplete inventories never authorize pruning.
* **Observability**: Regions expose their configured `region`, `leaderRegion`, and derived mode (`leader` or `follower`). Metrics track leader events published, follower events applied, rejected events, read-only write rejections, event freshness, inventory sync outcomes, inventory pruning, and split-brain detection. Alerts fire when no leader is configured, more than one region reports leader mode, a follower publishes events, a follower receives non-leader events, event freshness exceeds its expected resync interval, or inventory pruning fails.
* **Resiliency**: Pub/Sub unavailability does not affect leader-local writes or follower-local reads, but followers become stale until delivery resumes. The leader periodically captures a complete consistent API LIST and publishes its per-object state and bounded inventory pages for each configured type. Followers validate every page and referenced payload (or newer local source evidence), then reconcile only keys not newer than the snapshot. Snapshot reconciliation can repair unchanged-source drift; ordinary duplicate events cannot resurrect deleted grants. Reconciliation includes objects from previous leaders and objects with no mirror annotation. Manual failover promotes a follower by updating GitOps/Argo/Helm leadership config after fencing the old leader's writes and data publishing.

### Security:

* Public API write authorization is leader-aware. Non-leader regions reject mutating requests for replicated resource types even when the caller is otherwise authorized.
* The global authorization hostname serves requests only at the designated ready leader, with normal authentication and authorization. Stale DNS or connections to a follower or fenced old leader receive rejection for reads and writes. Explicit regional follower endpoints may serve local reads but reject mutations; no mutation is automatically redirected or replayed.
* Private API/RBAC restricts human and operator writes to replicated resource types in follower regions.
* The replication controller ServiceAccount keeps the local write permissions required to create, update, and delete mirrored objects in follower regions.
* Only the leader's workload identity may publish to the data topic. Followers may publish control requests on a separate topic, whose consumer rejects all resource data. Receivers additionally validate configured leader origin and generation; caller-supplied `OriginRegion` alone is not authentication. No inherited project-level publisher grant may bypass this separation.
* The `replication.gcp.managed.openshift.io/replicated-from` annotation is diagnostic. Pruning covers every object of a configured replicated type, including former-leader objects without annotations; other resource types and namespaces are excluded.
* Pub/Sub authentication uses GCP Workload Identity in production or the emulator for local development.

### Performance:

* Replication is asynchronous and not on the follower read path.
* Leader writes incur normal local write latency plus asynchronous Pub/Sub publish latency.
* Users far from the leader may see higher write latency because the CLI routes reads and writes to the leader.
* Followers serve local reads from their mirrored database state, subject to replication lag.

### Cost:

* Pub/Sub cost scales with leader event volume plus periodic inventory events.
* Mirrored resources and small durable replication metadata records use the existing regional database. Inventory assembly uses bounded temporary storage.
* One replication controller deployment runs per region, with publisher behavior enabled only in the leader. Same-leader publisher overlap produces safe duplicates because source versions are immutable.

### Operability:

* Leadership region and generation are configured through GitOps/Argo/Helm at startup. Mode derives from `region == leaderRegion`; all regions require configuration rollouts after promotion. Runtime leadership refresh and automatic election are deferred.
* Manual failover fences old-leader public/private writes, stops its publisher, and confirms data publishing permission is revoked before granting it to the new leader. Operators update the leader/generation configuration, roll out deployments, and validate follower reconciliation. After the target passes promotion, Cedar, TLS/ingress, and ESPv2 checks, GitOps changes the global hostname's DNS target. The discovery manifest and customer URL remain stable; routing never automatically promotes a healthy follower. Followers whose rollout is delayed recover through a new resync.
* A recovered old leader stays out of customer traffic until complete inventories reconcile all configured resource types and each serving API replica reloads Cedar policies and invalidates caches. Optional promotion back uses the same manual procedure with a new generation. Authorization freshness is not established by a rebuild timestamp alone.
* Receiver mutations and reconciliation completion validate the active leadership generation transactionally, so older processes cannot commit after a rollout. Datastore restoration that reuses source versions requires a new generation and reconciliation.
* Local development uses the Google Cloud Pub/Sub emulator with one leader Kind cluster and one or more follower Kind clusters.
* The planned E2E acceptance suite must validate leader-to-follower replication, follower read-only enforcement, CLI leader routing, inventory drift repair, manual failover, and split-brain detection.
