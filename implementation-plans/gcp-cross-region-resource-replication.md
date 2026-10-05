# Cross-Region Resource Replication

## Overview

This document specifies the single-leader cross-region resource replication mechanism for Gecko. It defines leader/follower behavior, CLI routing, read-only enforcement, Pub/Sub event processing, follower mirror reconciliation, manual failover, and deployment topology.

This implements the architecture decided in [cross-region-resource-replication](../design-decisions/infrastructure/cross-region-resource-replication.md).

**Repository**: [gecko](https://github.com/openshift-online/gecko)

**Status**: Implementation plan. The API prerequisites and acceptance tests below are pending Gecko work; this documentation does not establish that the current implementation satisfies them.

---

## Design Decisions

| Decision | Choice |
|---|---|
| Transport | Google Cloud Pub/Sub (leader-only data topic, separate resync control topic) |
| Write authority | Exactly one configured leader region |
| Follower behavior | Read-only mirror for replicated resource types |
| CLI routing | Existing static endpoint discovery supplies a stable global authorization URL routed to the ready leader |
| Forced follower writes | Reject with structured read-only error; no server-side redirect |
| Leadership config | Leader region and generation in GitOps/Argo/Helm; applied at startup |
| Failover | Manual promotion by changing leadership config after fencing old leader |
| Replication direction | Leader to followers only |
| Resource type selection | Startup flags or Helm values (`--replicate`) |
| Event validation | Followers apply only events from the configured leader |
| Event ordering | Leader API resource versions within a leadership generation; unordered delivery is expected |
| Delete recovery | Versioned API watch deletes plus periodic authoritative LIST reconciliation |
| Namespace handling | Receiver auto-creates target namespace if missing |
| Private API writes | Restricted by follower RBAC to replication and explicitly scoped local finalization |
| Error handling | Permanent errors Ack'd, logged at ERROR, and counted; transient errors Nack'd for Pub/Sub retry |
| Observability | Structured audit logs + Prometheus metrics for mode, lag, events, rejections, inventory, and split-brain |
| Initial use case | Authorization Roles and RoleBindings |

---

## Terminology

| Term | Meaning |
|---|---|
| Leader region | The single region configured to accept writes for replicated resource types |
| Follower region | A non-leader region that mirrors leader state and rejects direct writes |
| Mirror object | A local follower copy of an object whose authoritative source is the leader |
| Leadership config | GitOps/Argo/Helm-delivered configuration naming the current leader region and generation |
| Source resource version | API-assigned version of a source mutation; comparable only within its resource type and leadership generation |
| Snapshot resource version | Collection version of a consistent source LIST; an unchanged object can have an older version |
| Observed-through version | Source version through which a follower has verified a key, including absence |
| RPO | Recovery Point Objective: accepted write loss during failover; unplanned loss may be unknown |
| RTO | Recovery Time Objective: time to restore write availability, bounded by detection, fencing, config rollout, and validation |

---

## Configuration

### Startup Flags

| Flag | Env Var | Default | Description |
|---|---|---|---|
| `--region` | `REPL_REGION` | (required) | This region's identifier |
| `--leader-region` | `REPL_LEADER_REGION` | (required) | Current leader region identifier |
| `--leader-generation` | `REPL_LEADER_GENERATION` | (required) | Positive integer increased on every leadership change, including failback |
| `--pubsub-project` | `PUBSUB_PROJECT` | `gecko-local` | GCP project ID for Pub/Sub |
| `--pubsub-topic` | `REPL_PUBSUB_TOPIC` | `resource-replication` | Leader-only data topic |
| `--pubsub-control-topic` | `REPL_PUBSUB_CONTROL_TOPIC` | `resource-replication-control` | Topic for `RESYNC_REQUEST` only |
| `--pubsub-control-subscription` | `REPL_PUBSUB_CONTROL_SUBSCRIPTION` | (required in leader) | This region's subscription to the control topic |
| `--pubsub-subscription` | `REPL_PUBSUB_SUBSCRIPTION` | (required) | Region-specific subscription name |
| `--replicate` | `REPL_RESOURCE_TYPES` | (required) | Comma-separated resource types to replicate, for example `roles.gcp.managed.openshift.io,rolebindings.gcp.managed.openshift.io` |
| `--resync-interval` | `REPL_RESYNC_INTERVAL` | `30m` | Leader interval for periodic resource resync and inventory publishing |

Mode is derived from `region == leaderRegion` at startup. Leadership configuration is shared with public API write guards. Changing leader or generation requires GitOps rollouts of the affected API servers and replication controllers in every region; controllers do not refresh leadership configuration at runtime, and there is no automatic election. The `PUBSUB_EMULATOR_HOST` environment variable is supported for local development with the Pub/Sub emulator.

---

## Discovery and CLI Routing

Use the existing per-environment static [endpoint discovery manifest](../design-decisions/networking/endpoint-discovery.md#global-authorization-endpoint). Terraform publishes a stable global authorization endpoint alongside regional endpoints through the same CDN/GCS discovery stack. The manifest advertises a URL; it neither identifies nor elects the current leader.

### Behavior

1. Resolve environment and public region through discovery. For Role and RoleBinding operations, use the versioned manifest's `global.authorization_endpoint` for reads and writes. Keep the selected regional `platform_api_endpoint` and OIDC issuer for other operations; adding replicated resource types requires an explicit client routing mapping.
2. The infrastructure routes the stable global hostname to the configured, ready leader's existing regional public frontend. GitOps manages its DNS target, TLS/ingress host configuration, and ESPv2 accepted audience. No regional server proxies or redirects requests to another region, and routing never falls back automatically to a healthy follower.
3. Preserve explicit endpoint override precedence. An explicitly selected regional follower endpoint may serve local reads, subject to replication lag, and must reject mutations. Overrides do not bypass authentication, authorization, or leadership gates.
4. Clients validate the manifest schema and HTTPS URLs. Unsupported schema versions or a missing global endpoint yield an actionable discovery/configuration error, unless an explicit endpoint override is supplied; do not silently choose an arbitrary regional endpoint. Publish the versioned schema alongside the legacy region-only artifact and retain existing regional fields as specified in the discovery decision.
5. On rejection or timeout, report the error and the stable authorization URL when configured. Refreshing discovery or reconnecting may resolve stale metadata/connections, but do not silently replay a mutation or ask customers to configure a new leader region after promotion. The global URL remains unchanged.

### Global Hostname and Readiness

All regional frontends that can receive the global hostname must recognize it separately from their normal regional hostname. Requests on the global hostname, including GET/LIST, succeed only at the designated leader after reconciliation/promotion checks and Cedar reloads have made the serving replica ready. A follower, fenced old leader, stale-generation process, or unready replica rejects global-host requests with a structured unavailable response. Ordinary authentication and Cedar authorization still apply. Restrict this hostname to the advertised authorization resource routes; other regional APIs keep their regional endpoint.

Install the expected global hostname from trusted deployment metadata, validate it at ingress, and preserve that validated routing identity to the API guard. Do not trust an arbitrary caller-supplied forwarded-host header. The guard checks durable active leadership generation and local readiness, so stale DNS, old connections, or rollout skew cannot serve a stale follower as the global authority. A request admitted before fencing may finish; planned transfer drains admitted mutations before its final snapshot.

Prepare and validate the target's certificate, ingress host, ESPv2 issuer/audience configuration, and leader readiness before changing the Terraform-managed DNS target. DNS TTL affects recovery time, not authority: keep the old endpoint fenced throughout cache/connection expiry and when it recovers. Health checks may remove an unavailable leader from service but cannot promote a follower. Discovery stays static during an ordinary leadership change.

### Follower Rejection Response

Regional follower APIs reject mutating requests for replicated resource types with a structured response; global-host requests to a non-serving region use an unavailable error, including for reads:

```json
{
  "error": "region is read-only",
  "authorizationEndpoint": "https://authz.integration.gcp-hcp.devshift.net",
  "retryable": false
}
```

The hostname above is illustrative; Terraform derives the deployed endpoint from environment metadata. Use HTTP `503 Service Unavailable` for the global-host availability gate. Regional read-only errors use a deterministic non-success status selected during API implementation. Neither response redirects or authorizes an automatic mutation replay. Optional leader-region diagnostics are not client configuration requirements.

---

## Public API Read-Only Enforcement

The public API server enforces leader-only writes for replicated resource types. The table below applies to regional hostnames; the global hostname additionally requires ready-leader status for every request, including reads.

| Request | Leader Region | Follower Region |
|---|---|---|
| `GET` | Allow | Allow if exposed locally; CLI normally uses leader |
| `LIST` | Allow | Allow if exposed locally; CLI normally uses leader |
| `POST` | Allow after normal authorization | Reject read-only |
| `PUT` | Allow after normal authorization | Reject read-only |
| `PATCH` | Allow after normal authorization | Reject read-only |
| `DELETE` | Allow after normal authorization | Reject read-only |

The read-only guard runs after authentication and before resource mutation. Authorization still applies normally to allowed leader and follower reads and leader mutations. Follower rejection is not an authorization success; it is a regional mode constraint.

---

## Private API and RBAC

Follower regions restrict human/operator writes through private API RBAC while preserving replication controller write access.

### Goals

* Human and operator identities should not create, update, patch, or delete replicated resource types in follower regions.
* The replication controller ServiceAccount must be able to create, update, patch, and delete replicated resource types in follower regions so it can apply leader state.
* Namespace read/create permissions remain available to the replication controller when namespace auto-creation is enabled.
* Break-glass access, if required, must be explicit, audited, and outside the normal role bindings.

Example replication controller ClusterRole:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: replication-controller
rules:
  - apiGroups: ["gcp.managed.openshift.io"]
    resources: ["roles", "rolebindings"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch", "create"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```

The RBAC rules for human/operator identities are environment-specific, but follower regions must not grant normal write verbs for replicated resource types to those identities.

---

## Annotations

The replication controller uses annotations to mark follower mirror objects. Annotations are not used for ownership transfer.

### Replicated-From Annotation

```text
replication.gcp.managed.openshift.io/replicated-from: <leader-region>
```

Set by the Receiver on objects it creates or updates from leader replication events. This indicates the object is a follower mirror of leader-authoritative state.

### Applied Event State

The Receiver persists, per resource key, the active leadership generation, the last applied source object's UID and resource version (when present), an **observed-through version**, and whether the key is deleted. An incremental mutation at version R advances the observed-through version to R. A validated snapshot at version S advances it to S even if that object's own version is older. It also persists the last completed snapshot version per resource type. These records survive controller restarts; annotations alone cannot track a deleted object.

The source version is internal replication state, separate from the follower's `metadata.resourceVersion`. Each local mutation receives its normal local API version. Leadership generation is not Kubernetes `metadata.generation`. Deletion records are not exposed as resources through ordinary GET/LIST. Preserve normal local UID allocation, update preconditions, validation, watch notifications, and finalizer behavior; if finalizers prevent removal, do not mark a deletion or prune complete until the object is gone.

Track the source UID only as internal incarnation identity. A changed source UID at the same key means replacement: delete the old local incarnation using its local UID/version preconditions, wait for normal deletion to finish, then create the new mirror with a new local UID. Persist pending replacement work and recheck generation and newer source state before every step; never let a delayed deletion remove a replacement. Apply this rule to incremental upserts and snapshots, including a missed source DELETE.

Source finalizers, owner references, deletion timestamps, and server-owned metadata are not copied to the mirror: they describe the source cluster's lifecycle. A source object remains desired while its source deletion is pending; actual source deletion removes the mirror. Preserve target-local safety finalizers and allow their explicitly authorized local controllers to complete finalization without granting general follower writes. A configured replicated type must have a documented metadata mapping and functioning local finalization path; unsupported lifecycle dependencies block enabling that type. Initial Role/RoleBinding replication must verify this contract. Do not unconditionally strip local finalizers to make reconciliation finish.

Atomically validate the active generation, compare source state, and persist each local mutation with its tracking state before acknowledging application. These are internal storage requirements, not a new customer-facing replication or snapshot API. Concurrent receivers must use transactional compare-and-set checks or equivalent storage serialization; a process-local mutex alone is insufficient during overlapping pods. Namespace creation can precede this transaction because it is idempotent.

---

## Kubernetes API Prerequisites

Replication consumes the leader's regional API using standard LIST/WATCH. It depends on the [Kubernetes API resource-version and list/watch contract](https://kubernetes.io/docs/reference/using-api/api-concepts/#resource-versions), rather than assigning versions in the publisher.

| API guarantee | Required behavior |
|---|---|
| Mutation versions | Creates, updates, and actual deletions receive increasing resource versions within the same API group/resource type. Versions order committed state, not request start times. |
| Version representation | Preserve decimal resource-version strings unmodified. Compare as arbitrary-precision integers; mutation versions are positive, while the initial empty-store snapshot boundary may be `0`; do not assume a fixed-width client representation or subtract versions to infer gaps. |
| LIST snapshot | The collection version identifies a consistent snapshot, including when empty. An unqualified most-recent LIST must include all mutations committed before it began. Every continuation page belongs to that snapshot and has the same collection version. Item versions identify their last mutation, not the snapshot time. |
| Expiration | An unavailable snapshot/continuation returns `410 Gone`; the client restarts the entire list. Never silently continue against newer state. |
| WATCH | Resume after the requested version with retained changes, including correctly versioned deletes, or report expiration and require relisting. A list-to-watch transition must not silently lose mutations. Retention can be bounded. |
| Local API behavior | Replication writes retain ordinary Kubernetes validation, concurrency, deletion, and watch behavior. Source identity/version tracking stays separate from target metadata. |

Versions are compared only within one resource type and one leadership generation. A source datastore restore or counter rollback must not reuse versions within that generation: fence writes, increase the leadership generation, and reconcile before restoring service. The development in-memory store has process-lifetime data and needs a new generation/bootstrap after losing that state.

### Pending Gecko Storage Work

Inspection of Gecko at `3d67af5` identified these prerequisites. They must be implemented and tested in Gecko before enabling this replication protocol; the table is not a claim of completed fixes.

| Component | Required implementation work |
|---|---|
| Spanner | Reuse the existing per-resource-type counter updated transactionally with mutations. Expose the deletion's new version instead of the deleted row's previous version. The application-generated `updated_at` is not a commit-order token. |
| PostgreSQL | Replace `pg_current_xact_id()` ordering with a per-resource-type counter whose update and resource mutation share one transaction and serialization lock. `NOW()` and transaction IDs are not commit-order guarantees. Fence existing writers during migration, initialize above versions retained in resources/watch history, and invalidate incompatible old watch/continue tokens before resuming; do not mix allocation schemes. |
| Memory | Keep mutation/counter changes under the store lock, increment on deletion, and provide snapshot-consistent lists for the process lifetime. |
| LIST implementations | Preserve one snapshot across continuation requests; expose a collection boundary that covers deletes and empty results, not the maximum version among returned rows. Expire unavailable snapshots explicitly. |
| WATCH implementations and API adapters | Carry the mutation version through to the watch object's metadata, especially for deletes. Detect unavailable history and relist rather than silently replaying only surviving objects. Test commit/broadcast interruption and reconnect behavior. |
| Receiver storage | Support atomic local mutation/tracking and active-generation checks while preserving normal API semantics, including ordinary local versions and watch notifications. |

The replication layer needs no publisher-owned counter, publisher ownership token, or unbounded watch history. No transaction couples a user write to Pub/Sub publication; committed-but-unpublished loss remains possible during unplanned regional failover.

---

## Replication Event Model

Each create/update message contains one resource. Inventories contain bounded pages of resource keys and expected object versions; the full dataset is never placed in one Pub/Sub message.

```go
type ReplicationEvent struct {
    EventType               string          `json:"eventType"` // CREATE_OR_UPDATE, DELETE, RESYNC_REQUEST, INVENTORY
    ResourceKind            string          `json:"resourceKind"` // Registered group/resource identifier
    OriginRegion            string          `json:"originRegion"`
    Generation              uint64          `json:"generation"` // Replication leadership, not metadata.generation
    SourceResourceVersion   string          `json:"sourceResourceVersion,omitempty"`
    SnapshotResourceVersion string          `json:"snapshotResourceVersion,omitempty"`
    Namespace               string          `json:"namespace,omitempty"`
    Name                    string          `json:"name,omitempty"`
    UpdatedAt               time.Time       `json:"updatedAt"` // Diagnostics only
    SyncID                  string          `json:"syncID,omitempty"` // Unique capture ID, not an ordering token
    Object                  json.RawMessage `json:"object,omitempty"`
    PageIndex               uint32          `json:"pageIndex"` // Zero-based, INVENTORY only
    PageCount               uint32          `json:"pageCount"` // INVENTORY only
    Inventory               []ObjectRef     `json:"inventory,omitempty"`
}

type ObjectRef struct {
    Namespace             string `json:"namespace"`
    Name                  string `json:"name"`
    SourceResourceVersion string `json:"sourceResourceVersion"`
    SourceUID             string `json:"sourceUID"`
}
```

| Event Type | Topic | Required version/capture fields |
|---|---|---|
| Incremental `CREATE_OR_UPDATE` | Data | `SourceResourceVersion` from the source object's metadata; no snapshot fields |
| `DELETE` | Data | `SourceResourceVersion` of the source deletion event; no snapshot fields |
| Snapshot `CREATE_OR_UPDATE` | Data | Original object `SourceResourceVersion`, LIST `SnapshotResourceVersion`, and `SyncID` |
| `INVENTORY` | Data | LIST `SnapshotResourceVersion`, `SyncID`, page metadata, and expected object versions; no envelope `SourceResourceVersion` |
| `RESYNC_REQUEST` | Control | No object data or resource versions |

Snapshot object versions must be at or below their snapshot boundary. A snapshot object at version `37` in a LIST at `42` is valid. Do not overwrite its source metadata with `42`. Both versions are preserved for their separate purposes. Validate `ResourceKind` against the registered group/resource identifier (for example `rolebindings.gcp.managed.openshift.io`), not an ambiguous Kind name.

### Size Limits

[Pub/Sub limits](https://docs.cloud.google.com/pubsub/quotas) message data and total publish request size to 10 MB. Use a conservative application limit of 1 MiB per encoded event, including the envelope, and configure batching below the total request limit. Build inventory pages by encoded byte size, not object count. A single resource must fit its event limit; public and private write validation must enforce that limit before accepting a replicated resource. An existing oversized resource causes a visible resync failure, with no completed inventory or pruning. Do not truncate objects or silently skip them.

### API Versions and Immutable Publication

The API owns mutation ordering. Every event captures its payload and API-assigned version together. Retrying publication preserves both. A later read can produce another event using that read's actual resource version, but must never relabel previously captured data. Timestamps and Pub/Sub delivery order are not ordering authorities.

Overlapping publishers in the same leader region may publish duplicate or differently timed snapshots. Their events remain safe because source versions and snapshot boundaries come from the same authoritative API. Correctness does not depend on one exclusive publisher process.

---

## Publisher

The Publisher is active only in the configured leader region. Run one LIST/WATCH loop per configured resource type against that region's own API. Capture the full replicated scope, across all namespaces for namespaced types, without selectors that could omit objects later subject to pruning. Followers receive data through Pub/Sub; they do not fetch resources directly from another cluster.

### Publishing Logic

1. On startup, LIST the resource type, following continuation tokens to capture one complete snapshot. Publish its resource payloads and inventory as described below, then WATCH from the returned collection version. The API must either replay changes since that boundary or return expiration.
2. Publish `ADDED`/`MODIFIED` watch payloads as incremental `CREATE_OR_UPDATE` events with the original object version. Publish `DELETED` with the deletion event's version and resource key. Watch bookmarks are resume information, not resource mutations.
3. On watch interruption, reconnect from the last safely handled watch position. A publisher must not advance that position past unhandled events. On `410 Gone`, lost local progress, or an uncertain gap, perform a fresh LIST. Periodic inventories also repair events lost between API observation and publication.
4. A key-only controller-runtime reconcile that returns `NotFound` cannot assign an authoritative deletion version. Schedule reconciliation from a complete LIST instead of fabricating a versioned delete. An informer resync against cached objects is not a substitute for an authoritative LIST.
5. Validate encoded size and retry transient publish failures with the same captured version and content. A process restart may recover through a fresh LIST instead of keeping a durable publisher event queue.

Followers do not register these data publication loops. Any follower attempt to publish `CREATE_OR_UPDATE`, `DELETE`, or `INVENTORY` is an error counted by `replication_events_rejected_total{reason="follower_publish"}`. Their control-topic `RESYNC_REQUEST` remains permitted.

### Periodic Resync and Inventory

On startup, on `--resync-interval`, and on a resync request:

1. Create a unique `SyncID` for this capture and obtain all pages of one API LIST snapshot. Preserve the collection's `SnapshotResourceVersion` and each object's own `SourceResourceVersion`. An expired continuation abandons the attempt and restarts the entire LIST with a new `SyncID`; never combine pages from different attempts.
2. After a complete successful LIST, publish individual snapshot `CREATE_OR_UPDATE` payloads and byte-bounded `INVENTORY` pages containing keys and expected object versions. Every message carries the same snapshot version and `SyncID`. Capture storage must be bounded; a capture that cannot fit the configured local budget fails visibly without publishing a complete inventory.
3. Represent an empty list with one empty inventory page carrying its collection version. Do not infer the boundary from the maximum item version.

Publication may be asynchronous and unordered. Receivers stage snapshot data until its complete set can be validated. Use one in-progress resync per kind per publisher, coalesce and rate-limit requests, and leave the watch loop running during periodic captures. A completed snapshot supplies authoritative repair even when no source mutations have occurred since the previous resync; its collection version may be unchanged.

### Resync Request Handling

Followers publish `RESYNC_REQUEST` only to the control topic. The configured leader consumes requests through its regional control subscription and schedules resync. Followers do not register data publishers. Receivers reject data event types arriving on the control subscription, even if their JSON claims leader origin. The `follower_publish` metric covers attempted data publication, not legitimate control requests.

---

## Receiver

The Receiver consumes data from its region-specific subscription. The leader additionally consumes its own control subscription. Each subscription has a fixed topic role; the JSON payload cannot choose that role.

### Message Processing

1. Validate message size, schema, event type for the subscription, registered resource type, version/capture fields, and agreement between envelope and object identity/version.
2. Route control requests only to the configured leader's resync handler. They can never reach resource mutation handlers.
3. On the data subscription, require configured leader origin and generation. IAM supplies publishing authority; `OriginRegion` alone does not authenticate the message. The leader skips its own data events.
4. Ack and count stale or mismatched generations. A delayed follower rollout requests a fresh resync after installing its new configuration; periodic resync repairs acknowledged events missed during rollout.
5. Dispatch incremental events to application and snapshot payloads/pages to durable assembly. Ack an incremental event only after its mutation/tracking is durable; Ack snapshot messages after durable staging. Staging is not application or inventory completion. Nack transient failures; Ack, log, and count permanent invalid events.

### Incremental Upsert and Delete Flow

1. In the active generation, compare the incoming source version R against both the key's observed-through version and the last completed snapshot boundary for that resource type. Older-generation/no tracking state can be replaced by current-generation state.
2. If R is at or below either current-generation boundary, Ack as a stale duplicate. A deletion record prevents delayed upserts from resurrecting an object.
3. For upsert, ensure the namespace exists, handle source-UID replacement as specified in [Applied Event State](#applied-event-state), and apply the desired resource through the normal local mutation path, setting `replicated-from`. Allocate a local UID on create and use the local resource version for update preconditions; do not copy source UID/resourceVersion/generation into local server-owned metadata.
4. For delete, remove the local mirror if present using local identity/preconditions. Record absence at R even if already absent. If local finalizers delay removal, retain pending work and complete it through normal deletion processing.
5. Atomically recheck the active generation and ordering conditions, persist the local result and tracking state with observed-through R, then Ack application. Retry conflicts against newly read local state.

Snapshot messages follow [inventory reconciliation](#completion-and-pruning), which permits validated same-version repair. Ordinary duplicate upserts never bypass the stale-event checks.

### Namespace Auto-Creation

```go
func (r *Receiver) ensureNamespace(ctx context.Context, namespace string) error {
    ns := &corev1.Namespace{
        ObjectMeta: metav1.ObjectMeta{Name: namespace},
    }
    err := r.client.Create(ctx, ns)
    if err != nil && !apierrors.IsAlreadyExists(err) {
        return err
    }
    return nil
}
```

This requires the replication controller's RBAC to include `get`, `list`, `watch`, and `create` on `namespaces`.

---

## Leader Inventory Reconciliation

### Inventory Page

```json
{
  "eventType": "INVENTORY",
  "resourceKind": "rolebindings.gcp.managed.openshift.io",
  "originRegion": "us-east1",
  "generation": 3,
  "snapshotResourceVersion": "42",
  "syncID": "capture-7b423ac1",
  "pageIndex": 0,
  "pageCount": 1,
  "inventory": [
    {"namespace": "customer-a", "name": "service-admin", "sourceResourceVersion": "37", "sourceUID": "aa6517e9-4c78-4d20-8482-f969298107e5"}
  ]
}
```

The corresponding snapshot payload carries source version `37`, snapshot version `42`, and the same `SyncID`; its object metadata still has source version `37`. An empty type has one empty inventory page. `SyncID` is unique per capture, including repeated captures at the same snapshot version.

### Completion and Pruning

1. Group pages and snapshot payloads by `(generation, resourceKind, SyncID)`. Validate a shared snapshot version S, a consistent page count, every page index, unique resource keys, and expected source versions at or below S. Match each payload's identity, source UID, and version to its inventory entry. Identical redelivery is harmless; conflicting duplicates or unreferenced payloads invalidate the set. Persist staging before Ack and bound its disk usage and lifetime. On timeout/capacity failure, abandon the attempt and request a fresh one; never prune from it.
2. Ignore captures with S **below** the last completed boundary. For every inventory entry require its matching staged payload, unless current-generation durable tracking proves state strictly newer than S. A fresh validated capture at the same boundary is allowed to repair drift. Do not require unchanged object versions to equal S or treat a stale duplicate event as proof of snapshot repair.
3. Apply the captured objects using transactional generation/order checks. Preserve a key with observed-through version greater than S. Otherwise restore the snapshot's desired state and persist its original object version plus observed-through S. Equal-boundary repair is permitted only through this validated snapshot path; it must not roll back newer mutations or bypass local preconditions/finalizers.
4. Scan all local objects of the configured resource type, including former-leader objects with old/missing annotations. For objects absent from the inventory, prune only if their current-generation observed-through version is at or below S, or their state is old-generation/untracked. Preserve newer objects. Atomically record each completed prune as absence observed through S. Namespaces and non-replicated types are excluded.
5. Persist completion at S only after every referenced key has been reconciled (or superseded by state newer than S) and pruning has succeeded. A crash leaves the staged capture retryable. The completed boundary rejects subsequently delayed incremental events at or below S and allows older per-key deletion records to be reclaimed. The global completed boundary never decreases; a later capture that has already completed supersedes older work.

Every application/prune/completion transaction rechecks the active generation and latest completed boundary. Serialize reconciliation/application per type across receivers or provide equivalent transactional checks. A newer completed snapshot makes an older in-flight capture ineligible to continue. Partial work remains safe and retryable, but a capture is not reported complete until all work succeeds.

This repairs dropped deletes and local drift without a global atomic replacement of the dataset. Different objects may temporarily represent different source states. Eventual convergence requires the leader API, publication, receiver storage, and periodic reconciliation to recover; it does not supply a fixed authorization-staleness bound during outages.

### Leadership Transition

Increment `leaderGeneration` for every promotion, including return to a previous region, and whenever source versions could be reused after datastore restoration. Retain the active `(leader, generation)` in durable regional replication state. Installing a new generation atomically invalidates mutations, pruning, and completion by old receivers, even for keys without current-generation tracking. A controller starting with older configuration must not lower this generation or become ready for replication work.

Discard incomplete older-generation captures and request a fresh LIST-based resync. Existing objects remain until current-generation reconciliation replaces or prunes them. A newly added or recovered region stays out of customer traffic until all configured resource types complete a resync and each serving API replica reloads Cedar policies and invalidates caches. Healthy followers may continue serving their existing, potentially stale local authorization state during rollout.

---

## Pub/Sub Topology

```text
Leader --resource events / inventory pages--> Data topic --> Regional data subscriptions
Followers --------RESYNC_REQUEST-----------> Control topic --> Leader's control subscription
```

### IAM Permissions

| Regional controller identity | Data topic | Control topic | Subscriptions |
|---|---|---|---|
| Current leader | Publish | No publish required | Consume its own data and control subscriptions |
| Follower | No publish | Publish | Consume its own data subscription |

Use distinct regional workload identities and Terraform-managed topic IAM. No broader project-level publisher grant may give followers access to the data topic. Give each region a control subscription for use when promoted; enable control consumption only in the leader. Provisioning subscriptions before publishing and periodic resync cover startup and rollout gaps. Kubernetes RBAC for local replication writes is separate from Pub/Sub IAM.

During promotion, revoke the old leader's data publishing permission and confirm effective denial before granting the new leader access. This accompanies fencing its public/private writes and stopping its publisher; permission changes alone do not prevent accepted-but-unpublished writes. A control-message body or attribute claiming leader identity never grants data publishing authority.

---

## Deployment

### Controller Binary

The replication controller is a subcommand of the existing `gecko-controllers` binary:

```text
gecko-controllers replication \
  --region=us-east1 \
  --leader-region=us-east1 \
  --leader-generation=1 \
  --pubsub-subscription=repl-us-east1 \
  --replicate=roles.gcp.managed.openshift.io,rolebindings.gcp.managed.openshift.io
```

### Containerfile

Reuses the `gecko-controllers` binary image. The entrypoint specifies the `replication` subcommand:

```dockerfile
FROM registry.access.redhat.com/ubi9/ubi-micro:latest
COPY gecko-controllers /app/gecko-controllers
USER 65532:65532
ENTRYPOINT ["/app/gecko-controllers", "replication"]
```

### Kubernetes Deployment

Use one replica per region in `gecko-system` as the operational default. A Recreate rollout can reduce overlap but does not guarantee exclusivity during pod replacement. Overlapping same-leader publishers must be safe because payloads retain API-assigned versions; no custom publisher ownership or fencing protocol is required. Receivers still enforce transactional generation and application checks. Environment variables configure region, leader region/generation, Pub/Sub project, data/control topics and subscriptions, and emulator host for local development. Persist ordering and inventory state in the regional store, not pod-local memory.

---

## Local Development

### Pub/Sub Emulator

Local development uses the Google Cloud Pub/Sub emulator (`gcr.io/google.com/cloudsdktool/google-cloud-cli:emulators`). In a multi-cluster Kind setup, the emulator runs as a standalone container on the Docker/Podman network shared by all clusters.

### Kind Multi-Cluster Setup

The `deploy/kind/setup-multi-region.sh` script:

1. Creates at least two Kind clusters, for example `gecko-us-east1` and `gecko-eu-west1`.
2. Starts a shared Pub/Sub emulator container.
3. Creates data and control topics and per-region subscriptions.
4. Builds and loads controller images into all clusters.
5. Deploys via Kustomize or Helm with one leader overlay and one or more follower overlays.
6. Configures in-cluster Services pointing to the shared emulator.

### Kustomize or Helm Overlays

Each region patches:

* `REPL_REGION`
* `REPL_LEADER_REGION` and `REPL_LEADER_GENERATION`
* `REPL_PUBSUB_SUBSCRIPTION` and `REPL_PUBSUB_CONTROL_SUBSCRIPTION`
* `REPL_PUBSUB_TOPIC` and `REPL_PUBSUB_CONTROL_TOPIC`
* Pub/Sub emulator host for local development

---

## Manual Failover

Leadership changes are manual. Automatic cross-region leader election is intentionally out of scope for the initial implementation.

### Data Loss and Freshness

An unplanned failover may lose writes committed but not published, and messages published but not yet applied by the target follower. A recently applied event can coexist with an older undelivered revocation. Last-event age and the largest observed source resource version are freshness indicators; neither proves a complete applied history or bounds RPO. Resource versions are not counts of missing writes, and subtracting them is not a backlog or RPO measurement.

An outbox could durably record publication intent alongside a resource write, but it cannot recover data stored only in an unavailable region. It is outside this implementation. When the old leader is unavailable, document the unknown loss window and explicit operator acceptance of that risk. Promotion does not establish that pre-outage revocations reached the target.

### Failover Procedure

1. Select the target and declare planned transfer or unplanned failover. Keep target replicated-resource writes gated until the checks below finish.
2. Fence all old-leader public and private writers. For a planned transfer, wait for all previously accepted mutations to finish committing, then keep its publisher available long enough to run one final resync with writes still fenced. Each final capture must start with a most-recent consistent API LIST after the drain barrier; `resourceVersion=0`/Any semantics or an older exact snapshot cannot establish final-transfer freshness. Record each final capture's generation, `SyncID`, and snapshot resource version, require the target to complete those specific inventories (even if their snapshot versions equal earlier captures), and explicitly reload policies and invalidate caches on every API replica that will serve the promoted region. Since writes are fenced, these inventories describe the final authoritative state.
3. For an unplanned failover, inspect completed inventories, event age, apply/publish failures, and the target's local authorization state. Record acceptance of possible lost writes, including revocations. Reload Cedar from that local state on each serving replica; a rebuild timestamp alone does not establish source convergence.
4. Stop the old publisher, revoke its data-topic publishing permission, and confirm effective denial. Ensure that restart/recovery cannot bring the old leader back with write authority.
5. Update GitOps/Argo/Helm with the new `leaderRegion` and an increased `leaderGeneration`. Grant data publishing only to the promoted regional identity and install the corresponding control permissions.
6. Roll out the target's controllers and public API configuration. Verify durable active generation, the planned-transfer or accepted unplanned-recovery checks, Cedar readiness on every serving replica, and global-host TLS/ingress/ESPv2 configuration before enabling replicated-resource writes. Its publisher begins a resync in the new generation from the promoted local state.
7. Roll out remaining regions' controllers, API write guards, and follower RBAC. Each requests a fresh resync after installing the new configuration. Events rejected before rollout are repaired by this or the periodic resync.
8. Change the global authorization hostname's Terraform-managed DNS target to the ready new leader. Verify discovered Role/RoleBinding reads and writes with unchanged customer CLI configuration, including authentication/TLS, and verify that stale DNS or persistent connections to the old leader receive rejection for reads and writes. Keep the old endpoint fenced. Verify every follower completes current-generation inventories for every configured kind. Monitor failures and freshness indicators; do not interpret them as a measured RPO.

### Failback Procedure

1. Recover the old leader with follower configuration and no data-topic publish permission. Keep its customer API out of traffic while it reconciles.
2. Complete current-generation inventories for all replicated types. This includes removing locally authored objects absent from the new leader, even when they have no `replicated-from` annotation.
3. Explicitly reload Cedar policies and invalidate caches on every serving API replica, then restore customer traffic. A reload failure keeps that replica unready.
4. Optional promotion back uses the same failover procedure with another generation increment.

### Split-Brain Guardrails

* Mode is derived from `region == leaderRegion`.
* Followers reject public writes on regional hostnames and all authorization requests on the global hostname.
* Follower RBAC restricts private writes by human/operator identities.
* Followers reject data events with a non-leader origin or mismatched generation. Topic IAM restricts publishing to the leader.
* Alerts fire if more than one region reports leader mode.
* Alerts fire if a follower publishes replication events.

---

## Observability

### Audit Logs

All replication operations emit structured log entries using the controller-runtime logger.

**Publisher:**

| Level | Event | Fields |
|---|---|---|
| INFO | Published event | `eventType`, `resourceKind`, `originRegion` |
| INFO | Periodic resync completed | `resourceKind`, `resourcesPublished`, `syncID` |
| INFO | Published inventory | `resourceKind`, `objectCount`, `syncID` |
| WARN | Publish failed, requeuing | `resourceKind`, `error` |
| ERROR | Follower attempted publish | `region`, `leaderRegion`, `eventType` |

**Receiver:**

| Level | Event | Fields |
|---|---|---|
| INFO | Upserted resource | `resourceKind`, `originRegion`, `outcome` |
| INFO | Deleted resource | `resourceKind`, `originRegion` |
| INFO | Created namespace | `namespaceHash` |
| INFO | Received `RESYNC_REQUEST` | `originRegion` |
| INFO | Applied inventory | `resourceKind`, `originRegion`, `syncID`, `generation`, `snapshotResourceVersion`, `objectCount`, `prunedCount` |
| WARN | Rejected stale event | `resourceKind`, `originRegion`, `generation`, `sourceResourceVersion`, `snapshotResourceVersion` |
| ERROR | Rejected non-leader event | `eventType`, `resourceKind`, `originRegion`, `leaderRegion` |
| ERROR | Permanent error, Ack'd | `eventType`, `resourceKind`, `originRegion`, `error` |

**Public API:**

| Level | Event | Fields |
|---|---|---|
| INFO | Rejected follower write | `region`, `leaderRegion`, `resourceKind`, `verb` |

**Sensitive-data policy**: Log fields must not expose raw customer-controlled values such as resource `namespace` or `name`. Use opaque identifiers or omit these fields from structured logs. This follows the project's No-Sensitive-Data-In-Logs rule.

### Prometheus Metrics

| Metric | Type | Labels | Description |
|---|---|---|---|
| `replication_region_mode_info` | Gauge | `region`, `leader_region`, `mode` | Current regional replication mode |
| `replication_events_published_total` | Counter | `region`, `event_type`, `resource_kind` | Events published by the leader |
| `replication_events_applied_total` | Counter | `region`, `event_type`, `resource_kind`, `origin_region` | Events applied by receivers |
| `replication_events_rejected_total` | Counter | `region`, `event_type`, `resource_kind`, `reason`, `origin_region` | Events rejected before apply |
| `replication_events_skipped_total` | Counter | `region`, `event_type`, `resource_kind`, `reason` | Events skipped without applying, for example echo, stale duplicate, or stale delete |
| `replication_events_dropped_total` | Counter | `region`, `event_type`, `resource_kind`, `reason` | Permanent errors Ack'd |
| `replication_readonly_write_rejections_total` | Counter | `region`, `resource_kind`, `verb` | Public API writes rejected in followers |
| `replication_publish_errors_total` | Counter | `region`, `resource_kind` | Publish failures that are requeued |
| `replication_publish_duration_seconds` | Histogram | `event_type` | Time to publish a single event |
| `replication_receive_duration_seconds` | Histogram | `event_type` | Time to process a received event |
| `replication_resync_duration_seconds` | Histogram | `resource_kind` | Time for leader resync and inventory |
| `replication_inventory_sync_total` | Counter | `region`, `resource_kind`, `outcome` | Inventory sync outcomes |
| `replication_inventory_pruned_objects_total` | Counter | `region`, `resource_kind` | Mirror objects pruned from completed leader inventory |
| `replication_last_applied_event_age_seconds` | Gauge | `region`, `leader_region` | Time since last successful data-event application; freshness only, not backlog or RPO |
| `replication_last_leader_event_timestamp` | Gauge | `region`, `leader_region` | Local Unix time of last successful data-event application |
| `replication_inventory_last_completed_age_seconds` | Gauge | `region`, `leader_region`, `resource_kind` | Age of last successfully completed capture in the configured generation; reset completion state on generation change |
| `replication_leader_generation` | Gauge | `region`, `leader_region` | Configured leadership generation |
| `replication_oversized_events_total` | Counter | `resource_kind` | Encoded events rejected for exceeding the application size limit |
| `replication_split_brain_detected_total` | Counter | `region` | Split-brain detection events |

Exact resource-version strings, generation, and completed `SyncID` belong in durable diagnostic state and completion logs. Do not convert resource versions into floating-point gauges or high-cardinality metric labels. Promotion checks query exact completion records; counters and ages support monitoring, not proof of source convergence.

### Recommended Alerts

| Alert | Condition | Severity |
|---|---|---|
| No leader configured | No region reports `mode="leader"` | Critical |
| Multiple leaders configured | More than one region reports `mode="leader"` | Critical |
| Follower publishing events | `rate(replication_events_rejected_total{reason="follower_publish"}[5m]) > 0` | Critical |
| Non-leader events received | `rate(replication_events_rejected_total{reason="non_leader_origin"}[5m]) > 0` | Critical |
| No recent follower application | `replication_last_applied_event_age_seconds > threshold` (allow for idle periods and resync interval); also alert on absent telemetry | Warning |
| Read-only write rejections spike | Unexpected increase in `replication_readonly_write_rejections_total` | Warning |
| Inventory pruning failed | `replication_inventory_sync_total{outcome="failed"}` increases | Warning |
| Sustained publish failures | `rate(replication_publish_errors_total[5m]) > 0.1` for 10m | Warning |

---

## Integration: Authorization Use Case

The initial use case for cross-region replication is authorization Roles and RoleBindings. The integration points with the Cedar authorization system are:

* **Replicated resources**: `roles.gcp.managed.openshift.io` and `rolebindings.gcp.managed.openshift.io`
* **Write authority**: Role and RoleBinding writes use the discovered global authorization endpoint, routed to the ready leader region.
* **Follower reads**: Follower regions can evaluate authorization from local mirror data, subject to replication lag. The CLI uses the discovered global authorization endpoint for reads by default; this does not make regional Cedar caches immediately consistent.
* **PlatformRoles are NOT replicated**: They are system-defined and deployed identically to all regions via Helm.
* **Cedar hot-reload interaction**: When the receiver creates, updates, or deletes a mirrored Role or RoleBinding in the local database, the Cedar authorizer's watch mechanism detects the change and triggers policy rebuild and cache invalidation.
* **Marketplace integration**: The Marketplace controller creates the initial `service-admin` RoleBinding through the leader. The replication controller propagates it to follower regions.
* **Leader outage behavior**: During a leader outage, follower regions continue to evaluate Cedar authorization decisions from their local mirror data. Authorization remains available but may become stale — no new Roles or RoleBindings can be created, updated, or deleted until a new leader is promoted. Existing authorization grants remain in effect. There is no guaranteed staleness bound during an outage; previously missed changes and subsequent policy reload failures can extend it beyond promotion.
* **Authorization convergence delay**: Replication convergence, where a follower mirror is up to date, does not guarantee immediate Cedar authorization convergence. After the receiver applies a leader event to the local store, the Cedar authorizer's watch mechanism must detect the change, rebuild affected policies, and invalidate caches. This adds watch and reload delay. The Cedar periodic resync is optional and disabled by default, so it supplies no unconditional staleness bound. Planned promotion and recovery explicitly reload policies and invalidate caches on every serving replica after reconciliation; a wall-clock rebuild timestamp alone does not prove that a specific revocation was loaded.

---

## E2E Test Coverage

The planned E2E suite runs against one leader Kind cluster and at least one follower Kind cluster. All cases below are future implementation acceptance criteria, not tests implemented or executed by this documentation PR.

| # | Test | What It Validates |
|---|---|---|
| 1 | Leader create replication | Resource created in leader appears in follower with `replicated-from` annotation |
| 2 | Leader update replication | Resource updated in leader updates follower mirror |
| 3 | Leader delete replication | Resource deleted in leader is removed from follower |
| 4 | RoleBinding replication | Namespace-scoped binding replicates correctly |
| 5 | Namespace auto-creation | Receiver creates missing namespace in follower |
| 6 | CLI discovery routing | Role/RoleBinding reads and writes use the manifest's global URL; other resources and OIDC discovery retain regional selection; explicit overrides take precedence |
| 7 | Forced follower public write rejected | Direct mutating request to follower returns read-only error |
| 8 | Private follower write blocked | Non-replication private API identity cannot write replicated resource type in follower |
| 9 | Replication controller follower write allowed | Replication controller can apply leader data in follower |
| 10 | Follower does not publish | Local follower changes do not produce replication events |
| 11 | Non-leader event rejected | Follower rejects replication event whose origin is not configured leader |
| 12 | New region bootstrap | Follower publishes `RESYNC_REQUEST`; leader republishes current state |
| 13 | Periodic resync repairs drift | Follower object manually deleted reappears after leader resync |
| 14 | Inventory prunes stale mirror | Follower object absent from completed leader inventory is deleted |
| 15 | Incomplete inventory does not prune | Follower preserves mirrors when inventory is incomplete or invalid |
| 16 | Manual failover | Follower promoted through config accepts writes; old leader becomes follower |
| 17 | Old leader rejoins as follower | Recovered old leader catches up from current leader |
| 18 | Split-brain detection | Multiple leader reports trigger detection/alert metric |
| 19 | Stale delete does not remove newer mirror | An out-of-order DELETE older than the current mirror is treated as a no-op |
| 20 | Revoke-then-promote | A revocation confirmed by the final planned-transfer inventory is enforced by every promoted API replica; unplanned transfer does not claim this guarantee |
| 21 | Delayed inventory versus newer create | A snapshot at API version S cannot prune a key observed at a later source version |
| 22 | Inventory pages arrive before object events | No pruning/completion until all referenced state is applied |
| 23 | Missing, conflicting, or expired pages | Incomplete assemblies never prune; a fresh resync repairs them |
| 24 | Empty and multi-page inventories | Empty kinds clear stale state; large sets use bounded messages |
| 25 | Delayed upsert after delete/prune | Persisted deletion state or completed boundary prevents resurrection, including after restart |
| 26 | Former-leader and previous-origin objects | Missed deletes remove objects regardless of old/missing annotations |
| 27 | A → B → A leadership | Earlier-generation events remain invalid when a region leads again |
| 28 | Publisher restart, retry, and overlap | Two same-leader processes preserve payload/source versions; duplicates and reordered captures converge without publisher fencing |
| 29 | Delayed configuration rollout | New-generation resync repairs data events acknowledged before rollout |
| 30 | Forged data through control topic | Receiver rejects data payloads even if they claim leader origin |
| 31 | Oversized resource and batch | Write validation rejects oversized encoded resources; publish batches stay below service limits |
| 32 | Fresh event with older revocation pending | Freshness does not satisfy planned-transfer convergence checks |
| 33 | Recovery interrupted during pruning | Retained pages and ordering state allow safe retry; APIs remain gated until recovery completes |
| 34 | Write in flight when fencing begins | Planned transfer waits for its commit and then requests a most-recent consistent LIST; a stale but internally consistent snapshot cannot satisfy the barrier |
| 35 | Concurrent API mutation contract | Each backend orders committed state correctly, including deletes; counters and mutations commit together |
| 36 | Watch deletion version | Deletes carry their new mutation version through API adapters; old object versions are not reused |
| 37 | Paginated API snapshot under writes | All pages retain one snapshot boundary; object versions may be older; empty lists still cover deletes |
| 38 | Snapshot expires mid-LIST | `410 Gone` abandons all pages, restarts with a new capture ID, and cannot authorize partial pruning |
| 39 | List/watch gap and reconnect | Required history is replayed or explicitly expires; interruption around commit/broadcast repairs through relisting |
| 40 | Unchanged-source drift repair | A new capture at the same boundary restores a missing/modified mirror without accepting ordinary duplicate grants after deletion |
| 41 | Delete and recreate same name | A missed DELETE followed by a new source UID replaces the old local incarnation through normal deletion/create, including changed immutable fields and delayed local finalizers; stale events cannot damage the replacement |
| 42 | Old receiver overlaps generation change | Old processes cannot mutate, prune, or mark completion after a newer generation is installed |
| 43 | Source datastore restore/reset | Reused versions require a new leadership generation and bootstrap; old messages remain invalid |
| 44 | Kubernetes metadata and deletion | Mirrors receive local UID/resourceVersion/generation; source versions stay internal; source lifecycle metadata is not copied; local preconditions and authorized finalization remain effective |
| 45 | PostgreSQL version migration | Writer cutover prevents mixed schemes, avoids version reuse, and invalidates incompatible watch/continue tokens |
| 46 | Versioned discovery compatibility | Legacy region-only clients remain supported; new clients validate the schema and report absent/unsupported global endpoint configuration without arbitrary regional fallback |
| 47 | Global endpoint planned failover | Terraform routing changes after target readiness; customer configuration and discovered URL stay unchanged; stale DNS/connections receive rejection from the old endpoint for reads and writes |
| 48 | Global endpoint readiness and outage | Healthy followers and unready/stale-generation leader replicas cannot serve global GET/LIST or mutations; regional follower reads remain available and normally authorized |
| 49 | Global frontend identity | Every promotion target serves the global TLS hostname and validates the configured ESPv2 issuer/audience; untrusted forwarded-host headers cannot bypass the gate; failed mutations are not silently replayed |

Run IAM integration tests against real Pub/Sub with regional workload identities: a follower publishing a forged leader event to the data topic must receive permission denied, and old-leader publishing must be denied after handoff. Emulator tests do not validate production IAM. Test the storage ordering/atomicity contract on each supported backend. These are implementation acceptance criteria, not tests implemented in this documentation repository.

Tests use polling with a 30-second timeout and shortened resync intervals in test deployments. A `--no-pause` flag supports CI execution.

---

## File Structure

```text
controllers/
  replication/
    publisher.go                        # Leader-only API LIST/WATCH publication loops
    publisher_test.go
    receiver.go                         # Pub/Sub message handler with validation, upsert, delete, inventory
    receiver_test.go
    inventory.go                        # Leader inventory publishing and follower pruning helpers
    inventory_test.go
    types.go                            # Replication event structs + constants
  cmd/replication/
    cmd.go                              # Cobra subcommand + wiring
deploy/kind/
  replication/
    Containerfile.controller            # Container image for replication controller
    controller-deployment.yaml          # Kubernetes deployment
    kustomization.yaml                  # Base kustomization
    rbac.yaml                           # ServiceAccount + ClusterRole + ClusterRoleBinding
    test/
      e2e-test.sh                       # Leader/follower E2E test suite
  setup-multi-region.sh                 # Multi-cluster Kind setup script
  teardown-multi-region.sh              # Cleanup script
deploy/multi-region/
  us-east1/kustomization.yaml           # Region overlay, leader in default local setup
  eu-west1/kustomization.yaml           # Region overlay, follower in default local setup
```
