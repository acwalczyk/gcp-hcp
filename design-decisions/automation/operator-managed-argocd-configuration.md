# Manage HCP Argo CD configuration through operator CR inputs

***Scope***: GCP-HCP

**Date**: 2026-10-13

**Status**: Proposed for team review; disposable validation complete, no rollout approval.

## Decision

For operator-managed Argo CD, declare required HCP configuration through supported
ArgoCD and NotificationsConfiguration inputs rather than competing generated
ConfigMap writers. Transfer bootstrap ownership through separately observed
protection, release and adoption gates; keep operator mode opt-in per target.

## Context

- **Problem:** GCP-1226 found operator reconciliation overwrote required HCP
  ConfigMap settings. Ready runtimes and Healthy Applications did not establish
  equivalent custom health, update filters, auth or notification configuration.
- **Constraints:** Preserve the Terraform → Fleet Config Sync → Argo CD boundary,
  bootstrap/root identities, disabled-admin baseline and external credential
  ownership. Never use drift suppression, operator disablement or local users to
  conceal a conflicting owner. MC and region require independent inputs/review.
- **Assumptions:** The selected operator/CRDs expose the inventoried supported
  fields. Revalidate contract and behavior for another artifact; trial images are
  not official signed/support certification. An unsupported required setting is
  an explicit product/integration blocker, not permission for an unsafe bypass.

This specializes the existing
[deployment-tooling swim lanes](deployment-tooling-swim-lanes.md); it does not
move Kubernetes resources into Terraform or add a separate infrastructure lane.

## Alternatives Considered

1. **Supported CR inputs with one owner (selected):** Private day-two Argo inputs
   own the existing CRs; operator controllers generate the ConfigMaps.
2. **Continue direct Helm/Config Sync CM writes:** Rejected; concurrent writers
   produce partial-sync fights and values that do not survive reconciliation.
3. **Ignore drift or stop operator reconciliation:** Rejected; green status would
   hide missing configuration or eliminate the ownership contract being adopted.

## Decision Rationale

- **Justification:** Structured fields express auth, RBAC, health and exclusions;
  supported extraConfig handles remaining required keys. Notification fields
  express service/context/templates/triggers without embedding credentials.
- **Evidence:** The [sanitized MC/region record](../../docs/openshift-gitops-configuration-validation.md)
  accounts for 26 CM/eight notification/two RBAC values and demonstrates staged
  ownership/prune transfer, representative behavior, two Application cycles and
  one intentional manager restart on each disposable topology.
- **Comparison:** CR ownership survives operator reconciliation without the
  competing CM declarations or drift-suppression workarounds in alternatives 2–3.

Final owners:

| Inputs/resources | Declarative owner | Runtime reconciler |
| --- | --- | --- |
| Operator/CRDs, namespace, bootstrap ESO/metadata/repository/root resources | Fleet Config Sync bootstrap | Existing bootstrap controllers |
| Existing ArgoCD spec | Private argocd-config chart after reviewed Config Sync release | Argo CD operator |
| Existing default NotificationsConfiguration spec | Private argocd-config chart | NotificationsConfiguration controller |
| Generated Argo CD ConfigMaps | No direct chart/bootstrap writer | Respective operator controllers |
| Registry/OAuth/GitHub credential values | Existing approved external secret path | ESO and existing Secret owner |

The opt-in
[handoff procedure](../../docs/openshift-gitops-configuration-handoff.md) requires
detach protection before release, including the mutation-ignore CM bridge;
live metadata/inventory removal before CR adoption; and value/owner verification
before guarded retirement of stale CM tracking and released seed declarations.
Do not collapse these into one source update.

## Consequences

### Positive

- Required configuration has one declarative input owner and supported runtime
  generation, with mapping/precedence tests and explicit per-cluster inventory.
- Legacy/default deployment behavior stays unchanged until reviewed target opt-in.
- Protected identities/root continuity and observed gates limit pruning risk.

### Negative

- Transfer requires coordinated public/private revisions and observed stages;
  Git source convergence alone is insufficient proof of ownership or behavior.
- Releasing the instance removes automatic Config Sync recreation on a fresh
  cluster. Protected-seed rebuild/reclaim orchestration remains GCP-1280/GCP-1288
  work; do not reverse ownership by blindly replaying an old spec.
- Configuration contracts are version-sensitive. The trial name `gitops` and
  engineering artifact choices are not production naming/support decisions.

## Cross-Cutting Concerns

### Security

- Preserve HTTPS/Gateway/IAP, Dex/RBAC configuration and disabled admin/anonymous
  access; never add a local-user fallback. Actual organizational authentication
  and signed-identity trust-boundary verification remain GCP-1283 acceptance.
- Credential values stay outside Git/logs/evidence. Configuration preservation
  is not proof of GitHub delivery or workload cloud identity (GCP-1284/GCP-1285).
- Chart tuning cannot override chart-owned auth settings, bypass the approved
  transport path or disable essential runtime components.

### Reliability and operability

- Compare rendered identity/prune effects, served schemas and all required values;
  verify actual behavior and persistence, not only readiness/key presence.
- Guard one-time metadata retirement with fresh UID/RV/owner/value tests; preserve
  data, lifecycle/operator metadata and adopted-CR prune/delete protections.
- Record stop/recovery boundaries. After adoption, roll forward or restore reviewed
  CR inputs; reintroducing legacy CM writers is not a safe rollback.
- No Terraform/IAM, artifact upgrade or default bootstrap replacement is part of
  this decision. Full release, lifecycle and production rollout remain separate
  adoption gates with explicit target authorization and applicable team DoD.
