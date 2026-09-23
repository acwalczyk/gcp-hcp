# Use Firestore as the Database-Backed Transport for Gecko

***Scope***: GCP-HCP

**Date**: 2026-06-05

**Status**: Implemented
**Supersedes**: [RC-MC Transport Layer (Maestro)](rc-mc-transport-layer.md)
**Study**: [`studies/datastore-transport.md`](../../studies/datastore-transport.md)
**Implementation Plan**: [`implementation-plans/gcp-813-datastore-transport.md`](../../implementation-plans/gcp-813-datastore-transport.md)

## Decision

Gecko's HostedCluster and NodePool controllers use Cloud Firestore (Native mode, regional) as the transport layer between region clusters and management clusters. Each management-cluster project hosts two Firestore databases, `specs` and `status`, for IAM-enforced directional isolation. Gecko writes desire documents to `specs`; `kube-applier-gcp` consumes them with Firestore snapshot listeners, applies them to the management cluster, and writes feedback to `status`. Gecko reads that feedback during reconciliation. Both sides authenticate through direct GKE Workload Identity Federation principals, with no intermediary service account or static credential. Document types are aligned with the kube-applier-gcp APIs (`Spec.TargetItem`, `Spec.KubeContent`, `Status.Conditions`).

The deployed flow is:

1. Gecko HC and NodePool controllers write Apply, Read, and Delete desires to the MC's `specs` database.
2. kube-applier-gcp watches the `specs` database with persistent Firestore snapshot listeners and applies or observes the target resources.
3. kube-applier-gcp writes operation conditions and observed resource content to the MC's `status` database.
4. Gecko reads status documents on reconciliation and requeues while asynchronous operations remain pending.

## Context

- **Problem Statement**: The superseded Maestro design introduced an intermediary between region controllers and management clusters. Other teams (ARO-HCP on Azure) demonstrated a simpler database-backed transport using CosmosDB, achieving the same spec delivery and status feedback with fewer moving parts. Gecko uses the equivalent Firestore design on GCP.
- **Constraints**:
  - Per-MC access segregation must be IAM-enforced, not application-enforced only
  - All data must remain within a single GCP region
  - Region clusters and management clusters run in separate GCP projects — cross-project IAM via Workload Identity Federation is required
  - **No static credentials**: no service account keys, database passwords, connection strings with embedded secrets, or API keys may be created or distributed. All authentication must flow through GKE Workload Identity → GCP IAM. This is a hard requirement, not a preference.
  - The solution should be compatible with the kube-applier-gcp document-store interaction pattern and APIs
- **Assumptions**:
  - The number of management clusters per region will be in the low hundreds (scaling to ~100-200 in the medium term)
  - Individual K8s resource specs (HostedCluster, NodePool, etc.) are under 1 MiB (Firestore document size limit)
  - The Platform API's Cluster and NodePool resources are the source of truth for desired state. Gecko reconciliation repopulates Firestore specs in an empty or rebuilt database.
  - Each MC runs in its own GCP project

## Alternatives Considered

1. **Cloud Firestore (per-MC databases)**: Two Firestore databases per management cluster (`specs` + `status`), enabling IAM-enforced directional isolation. Native real-time listeners provide change notification to kube-applier-gcp. Serverless, no infrastructure to manage. Document schema aligned with kube-applier-gcp desire types.

2. **Cloud Spanner (single database with FGAC + views)**: Fine-grained access control with definer's-rights views for read isolation. Change streams for near-real-time notification. However, write isolation is application-enforced only (views are read-only), 100 database roles per database ceiling, and not a document store (breaks ARO compatibility). Higher cost ($200-600/month vs. $10-50/month).

3. **Cloud SQL PostgreSQL (schema-per-MC)**: Strongest single-instance structural isolation via PostgreSQL GRANT per schema. IAM Database Authentication maps GCP service accounts to PostgreSQL roles. However, no native change notification (LISTEN/NOTIFY is unreliable), Auth Proxy sidecars required in every MC pod, connection pooling mandatory, schema migrations × N, and not a document store. Cost $100-300/month.

4. **Cloud Firestore (single shared database)**: Lowest cost and simplest model, but IAM cannot scope below the database level for server-side Go SDK. Security Rules are bypassed by the Admin SDK. IAM Conditions only resolve to database-level resource names, not document paths. Per-MC isolation would be application-enforced only — does not meet the hard isolation requirement.

5. **Cloud Bigtable**: Authorized views with row-prefix IAM. No native change notification. Minimum cost ~$1400/month for a 3-node HA cluster. Massively over-provisioned for this workload.

6. **AlloyDB for PostgreSQL**: Similar to Cloud SQL but with higher entry cost (~$250/month minimum) and more complex cross-project networking (Private Service Connect). No advantages over Cloud SQL for this workload.

## Decision Rationale

* **Justification**: Firestore per-MC databases provide the best combination of IAM-enforced isolation, operational simplicity, cost efficiency, and kube-applier-gcp compatibility. It is the only option that satisfies all hard requirements while also being document-store-native.

* **Evidence**:
  - Firestore's multi-database feature (GA since 2023) maps directly to CosmosDB's per-MC container model used by ARO-HCP
  - Hosting two Firestore databases (`specs`, `status`) in each MC's own project ties DB lifecycle to MC lifecycle and keeps kube-applier-gcp access local (no cross-project IAM needed for the agent)
  - IAM grants differentiated roles per database (`datastore.user` for write, `datastore.viewer` for read) — directional isolation is structurally enforced, not application-level
  - Native real-time listeners eliminate the need for polling infrastructure or CDC pipelines
  - Authentication is fully credential-free: Go SDK uses Application Default Credentials via GKE Workload Identity — no service account keys, database passwords, Auth Proxy sidecars, or secrets to create, rotate, or distribute
  - Compared with the Cloud Spanner estimate, Firestore is approximately 4-60x less expensive depending on usage ($10-50/month vs. $200-600/month)
  - kube-applier-gcp's typed CRUD, lister, and snapshot-listener interfaces map directly to Firestore operations

* **Comparison**:
  - **vs. Spanner**: Spanner provides stronger read isolation via FGAC views but cannot enforce write isolation at the row level (views are read-only). The 100-role-per-database ceiling and node-based pricing ($200-600/month) make it less suitable. Not a document store.
  - **vs. Cloud SQL**: Strongest single-instance isolation via PostgreSQL GRANT, but lacks native change notification, requires Auth Proxy sidecars in every pod (additional operational burden despite using Workload Identity — no static credentials, but still a sidecar to deploy and maintain), connection pooling, and schema migration × N. Not a document store — would require a fundamentally different interface from ARO's.
  - **vs. Firestore single DB**: Cannot meet IAM-enforced isolation requirement. Server-side SDK bypasses Security Rules; IAM Conditions cannot scope to document paths.

## Consequences

### Positive

* IAM-enforced per-MC isolation with zero application-layer trust assumptions
* DB pair lifecycle is tied to MC provisioning; teardown behavior follows the environment's Firestore delete-protection setting
* Native real-time listeners eliminate polling latency and infrastructure
* Lowest operational burden — fully managed, serverless, no proxies, no connection pooling, no schema migrations
* Lowest cost of all evaluated options ($10-50/month)
* Document schema aligned with the kube-applier-gcp desire types and integration contract
* Per-MC database pair provides independent lifecycle and recovery boundaries
* Recovery after an MC project or database rebuild relies on the Platform API state, Terraform database provisioning, and Gecko reconciliation repopulating specs. Status data is transient and will be re-reported by kube-applier-gcp; backup and point-in-time recovery are not configured.
* Regional Firestore provides managed service recovery for zonal failures; recovery objectives for region loss or document deletion are not defined by this design because backups and point-in-time recovery are not configured

### Negative

* Two Firestore databases per MC (specs + status) doubles the number of databases and connections compared to a single-database model
* Firestore is schemaless — no DDL-enforced schema validation; validation must be in application code
* 1 MiB document size limit could be a constraint for very large K8s resources
* IAM role changes have up to 5-minute cache lag

## Cross-Cutting Concerns

### Reliability:

* **Scalability**: Firestore scales automatically with no provisioned capacity. Each MC has two independent databases in its own GCP project (2 DBs per project, well under the 100 DBs/project limit).
* **Observability**: Firestore provides built-in metrics in Cloud Monitoring (read/write counts, latency, error rates) per database. Custom metrics can be added in Gecko and kube-applier-gcp for per-resource-type tracking.
* **Resiliency**: 99.99% SLA for regional Firestore with transparent zone failover. Recovery after MC project or database recreation relies on Terraform and Gecko reconciliation; this design does not claim a region-loss or document-deletion recovery objective.

### Security:

* The kube-applier-gcp pod's direct WIF principal is granted `roles/datastore.viewer` on `specs` (read only) and `roles/datastore.user` on `status` (read/write) — lateral movement between MCs is prevented by IAM.
* Gecko HC and NodePool direct WIF principals are granted cross-project `roles/datastore.user` on `specs` and `roles/datastore.viewer` on `status` for each MC — directional isolation is IAM-enforced.
* Cross-project IAM bindings are standard GCP IAM — no VPC peering, no network-level trust required
* Firestore data is encrypted at rest by default (Google-managed keys); CMEK is available if required
* **Spec/status directional isolation**: IAM-enforced via two databases per MC. Gecko cannot write status and kube-applier-gcp cannot write specs. A single-database simplification is possible but reduces directional isolation to application-level enforcement only

### Performance:

* Single-digit millisecond read/write latency within a region
* Real-time listeners deliver change notifications within tens of milliseconds of a write commit
* Strong consistency for reads within a session (read-after-write is immediate)

### Cost:

* Per-operation pricing: reads $0.06/100k, writes $0.18/100k, storage $0.18/GB/month
* With real-time listeners, read billing is only on changed documents — significantly cheaper than polling
* No idle compute cost (serverless)
* Estimated $10-50/month for hundreds of MCs with thousands of resources each

### Operability:

* Zero credential management — no service account keys, database passwords, or secrets to create, rotate, or distribute. Authentication flows entirely through GKE Workload Identity → IAM → Firestore Go SDK (Application Default Credentials) for both Gecko and kube-applier-gcp.
* No proxy sidecars, no connection pools, no node sizing, no version upgrades to manage
* Database lifecycle (create/delete per MC) is a single API call, scriptable in provisioning automation
* Recovery is scriptable through Terraform reconciliation and Gecko spec repopulation; this design does not configure Firestore backups or point-in-time recovery
* Go SDK with Workload Identity ADC — no credential management beyond IAM bindings
