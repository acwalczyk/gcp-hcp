# Implementation Plan: Firestore-Backed Database Transport

**Status**: Implemented (as-built record following the GCP-813 spike)
**Jira**: [GCP-813](https://redhat.atlassian.net/browse/GCP-813)
**Last Updated**: 2026-09-22

---

## Context

GCP-813 was a spike that evaluated replacing the superseded Maestro transport with a Google Cloud-native database transport. This document records the deployed result of that work: Gecko's HostedCluster and NodePool controllers write kube-applier-gcp desire documents to Firestore, and kube-applier-gcp applies those desires to each management cluster and writes status feedback back to Firestore. It is an as-built integration record, not a claim that the original CLM adapter phases were implemented as written.

Design decision: [`design-decisions/datastore-transport.md`](../design-decisions/networking/datastore-transport.md). Study: [`studies/datastore-transport.md`](../studies/datastore-transport.md). This document records the implemented architecture and the deployed repository locations.

## Architecture Overview

```text
Gecko controllers (region)                  Management cluster
──────────────────────────                  ──────────────────
       │ write desires
       v
  Firestore specs ──snapshot listener──> kube-applier-gcp ──apply/read/delete──> Kubernetes API
       ^                                      │                                      │
       │ read status during reconciliation   │ write status                         │ resource state
       └────────────── Firestore status <─────┴──────────────────────────────────────┘
```

**Key patterns:**
- Gecko controllers reconcile Platform API resources and read status from Firestore on demand while asynchronous operations are pending.
- kube-applier-gcp uses a Firestore real-time snapshot listener on the `specs` database for low-latency desire delivery.
- Two Firestore databases per MC: `specs` (Gecko writes, kube-applier-gcp reads) and `status` (kube-applier-gcp writes, Gecko reads) — IAM-enforced directional isolation.
- Gecko uses the kube-applier-gcp API types; no separate adapter or Pub/Sub transport is involved.

## Deployed integration contract

### Gecko transport

Gecko's Firestore transport is implemented in
[`controllers/client/transport/firestore/`](https://github.com/openshift-online/gecko/tree/main/controllers/client/transport/firestore).
It uses the management-cluster project ID to open the `specs` and `status`
databases, writes Apply/Read/Delete desires to `specs`, and reads operation
feedback and observed resource content from `status` during reconciliation.

### kube-applier-gcp contract

The per-management-cluster agent is implemented in the standalone
[`kube-applier-gcp` repository](https://github.com/openshift-online/kube-applier-gcp).
Its [README](https://github.com/openshift-online/kube-applier-gcp/blob/main/README.md)
is the source of truth for desire schemas, snapshot listeners, controller
behavior, status cleanup, and runtime flags. In brief, the agent reads desire
collections from `specs`, applies or observes Kubernetes resources, and writes
matching status documents to `status`.

### Infrastructure contract

The management-cluster Terraform module creates the two databases and grants
directional access through direct WIF principals. The deployed roles are:

| Workload | `specs` | `status` |
| --- | --- | --- |
| kube-applier-gcp | `roles/datastore.viewer` | `roles/datastore.user` |
| Gecko HC and NodePool | `roles/datastore.user` | `roles/datastore.viewer` |

See [`terraform/modules/management-cluster/firestore.tf`](https://github.com/openshift-online/gcp-hcp-infra/blob/main/terraform/modules/management-cluster/firestore.tf)
for the infrastructure-owned database and IAM contract.

## Validation

- Gecko transport unit tests cover desire construction, deterministic IDs,
  status aggregation, and deletion behavior.
- Gecko and kube-applier-gcp integration tests run against the Firestore
  emulator when `FIRESTORE_EMULATOR_HOST` is configured.
- The deployed IAM conditions enforce that Gecko cannot write `status` and
  kube-applier-gcp cannot write `specs`.
- The current controller and agent tests validate ApplyDesire, DeleteDesire,
  ReadDesire, status cleanup, and optimistic concurrency behavior. Snapshot
  listener reconnection is implemented in the informer layer and should be
  validated with the Firestore emulator or a deployed environment.

## Historical implementation decisions

The following decisions from the original plan remain part of the implemented
contract:

1. Gecko reads status on demand during reconciliation; the agent is the only
   side that uses a Firestore snapshot listener for desire delivery.
2. There is one desire per Kubernetes resource, with matching spec and status
   documents and no document bundling.
3. Document IDs are deterministic UUID v5 values derived from the group key and
   resource identity, providing idempotent retries.
4. Normal Kubernetes manifests are stored as raw JSON and must remain below
   Firestore's 1 MiB document limit.
5. Firestore is transient transport state. After an MC project or database
   rebuild, Gecko repopulates desired specs and kube-applier-gcp processes them
   to emit new status; this is reconciliation, not status restoration, and no
   backup-based recovery objective is defined.

The earlier Maestro-based transport is retained only in explicitly superseded
or historical documents, including the study linked above.
