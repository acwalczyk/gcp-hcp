# GCP-1282: Operator configuration ownership and cutover options

> **TL;DR:** Keep one owner per setting. Prefer clean replacement only with
> approved downtime/deletion and ready bootstrap/recovery inputs; use the optional
> in-place handoff when existing identities must remain. Neither path requires
> parallel Argo installations or authorizes a rollout.

**Opt-in guidance; no default rollout.** The reviewed in-place transition completed on
disposable MC and region trials on October 13, 2026; see the
[sanitized validation record](openshift-gitops-configuration-validation.md).
The companion implementation [PR #2315](https://github.com/openshift-online/gcp-hcp-infra/pull/2315)
has merged; chart/matrix/tests are documented in
`gcp-hcp-infra/docs/ARGOCD_OPERATOR_CONFIGURATION.md`. Pin reviewed bootstrap and
chart revisions before proposing a live transition. The existing
`bootstrap/kustomization.yaml` does not reference these opt-in components;
ordinary bootstrap output is unchanged.

## Choose a cutover

The required design is **one owner per setting**, with the private chart supplying
supported CR inputs and the operator generating ConfigMaps. Keeping two Argo
installations running together is not required.

| Option | When to use it | Evidence in this PR |
| --- | --- | --- |
| Clean replacement | Prefer when downtime/deletion is approved and complete bootstrap/recovery inputs are ready; existing runtime identities may intentionally change | Preparation requirements below, not an implemented or validated replacement runbook |
| In-place handoff | Use when the existing instance and resource identities must remain | Optional procedure/components below; validated on disposable MC and region |

For a clean replacement, first review the exact old resources approved for
deletion, the pinned new instance/settings/artifact inputs, and a bootstrap/recovery
path that works **without the old Argo runtime**. The private chart alone is not
a bootstrap mechanism if Argo is unavailable. Retired trial seeds cannot recreate
the instance on their own.

Include ownerReferences, finalizers, garbage-collection and prune effects in the
retained/deleted-resource review; do not assume a Secret or root resource survives
just because it was not named in the deletion list. Do not strip finalizers as a
shortcut.

Preserve namespace/root resources, shared CRDs, ESO stores/Secrets, metadata/repo
access and the region platform API route; do not treat replacement approval as
permission to delete a namespace or shared APIs. Remove or release competing old
declarations as part of the reviewed plan. Deleting the runtime does not release
a surviving Config Sync seed owner or stop an old chart writing ConfigMaps.

Execute only a separately approved, target-specific plan and repeat the required
configuration/behavior/persistence checks. Productized bootstrap/rebuild and full
recovery testing remain GCP-1280/GCP-1288 work; this guide supplies no teardown
commands or claim of completed clean-replacement validation.

## Optional in-place handoff

This is the identity-preserving path, not a mandatory sequence for every new
installation. The supplied components do not install a second Argo runtime.

The validated operator-managed trials started with two bootstrap seeds overlapping
day-two configuration: `ArgoCD/argocd/gitops` and the Dex seed
`ConfigMap/argocd/argocd-cm`. The private chart will own supported ArgoCD and
NotificationsConfiguration CR inputs, not directly generated ConfigMaps.
Simply adding a private CR writer without releasing the public seed is unsafe.

### Supported release mechanism

Google documents both [preventing deletion and stopping management without
deleting objects](https://cloud.google.com/kubernetes-engine/enterprise/config-sync/docs/how-to/managing-objects).
Use separate observed revisions; do not collapse protection, release and
removal into one source update.

| Stage | Opt-in component | Required observed result |
| --- | --- | --- |
| Protect | `bootstrap/components/argocd-config-handoff` | Both seeds have `client.lifecycle.config.k8s.io/deletion: detach` with original UIDs |
| Release | `bootstrap/components/argocd-config-release` | Config Sync stops managing both seeds, removes its management metadata/inventory references, and does not delete them |
| Adopt | Private chart's reviewed operator mode | ArgoCD/default NotificationsConfiguration spec has one declarative owner; operator-generated CMs retain required values |
| Retire | Remove released seed resource references/files and temporary components | No generated resource deletion; namespace/ESO/metadata/root ApplicationSet/shared CRDs remain managed |

The release annotation `configmanagement.gke.io/managed: disabled` is an explicit
handoff, **not** a final drift-suppression workaround. The new private CR inputs
and their operator controllers must continue reconciling required settings.
Removing a managed resource without release/detach would prune it. A missing
ArgoCD CR can garbage-collect the runtime; never use deletion to clear OutOfSync.

### Candidate assembly and checks

These components are specifically scoped to the trial instance `gitops` in
`argocd`. That is not a production naming decision. Customize and independently
review selectors if another instance/namespace is chosen. Verify each selector
matches exactly one expected seed before any source publication.

In a **local reviewed copy** of the existing operator trial bootstrap, place the
team component next to the root and add:

```yaml
components:
  - components/argocd-config-handoff
```

Offline render with `kubectl kustomize <candidate-directory>`. Compare parsed
objects/resource identities with the preceding bootstrap: protection must only
add detach annotations to the two seeds. Release must only add the disabled
management annotation, preserving previous annotations and specs. Keep namespace,
ESO stores/ExternalSecrets, metadata/repo resources, root ApplicationSet,
operator/CRDs, controller RBAC and pull access unchanged. Preserve RootSync sizing.

The two known operator trial roots are on separate public fork revisions/dirs.
Do not add the component to shared org `main/bootstrap` or switch both clusters
as an incidental effect. Publication to a branch followed by a live RootSync
can itself deploy changes and needs explicit approval.

After each separately approved source stage, verify fetched revision, RootSync
Sync Completed/no errors, live annotations and original seed UIDs. Config Sync
does not leave `managed: disabled` on the live object; confirm its **management
metadata and inventory entry are removed**, not the presence of that value.
Then follow the private chart's protect/adopt/ConfigMap tracking-retirement
procedure. Argo prune protection and obsolete tracking cleanup are separate
from Config Sync release, and must not be skipped.

An existing `client.lifecycle.config.k8s.io/mutation: ignore` on argocd-cm can
cause Config Sync to skip an ordinary detach-annotation update. The private
chart's preceding CM-protection stage supplies the matching deletion-detach
bridge. Observe the live annotation on both seeds before release; a Synced
source alone is not that proof. The release stage was verified to remove only
management metadata and the two inventory references, without changing data or
UIDs. Do not remove mutation-ignore to force a sparse seed over the full CM.

## Bootstrap continuity and recovery

Keep the operator and root bootstrap resources under Config Sync. This preserves
the Terraform → Fleet Config Sync → Argo CD chain, but releasing the instance
means Config Sync cannot recreate that instance on a new cluster from the
released stage. Review protected installation-seed or exact-spec reclaim inputs
before replacement/rebuild. If bootstrap creates another Config Sync-owned
instance seed, its ownership still needs release before private-chart adoption.
Do not blindly replay an old sparse seed over current required settings.
Productized bootstrap/rebuild orchestration is separate GCP-1280/GCP-1288 work.

Before ownership release, a reviewed reversal may restore the preceding protected
inputs. After release, keep the retained resources protected and stop for an
exact-spec Config Sync reclaim plan if needed; never blindly replay an old seed.
After adoption, fix/roll back the private CR inputs rather than reintroducing
competing ConfigMaps or the old managed seed. Setting `operator.enabled=false`
is not a rollback: protected CRs continue reconciling while the chart resumes
direct ConfigMap writes. Reverse ownership transfer needs
its own protection, Argo tracking release, Config Sync reclaim and exact-spec
review. No reset/teardown, Route CRD removal, or shared-state Terraform action is
authorized by this procedure.

## Evidence boundary

Offline render tests verify preservation of seed specs/UID-independent resource
identities and existing annotations. The October 13 disposable trials additionally
verified live release/prune transitions, retained settings through two subsequent
Argo Application reconciliations and a relevant manager restart on each cluster,
and representative health/watch behavior **for the in-place path**. They do not
validate deleting/recreating Argo or rebuilding the cluster. These are bounded,
dated observations; repeat the reviewed gates for another target or artifact. Existing engineering
artifacts are not a certified supported release. Configuration preservation does
not establish authenticated IAP login, real GitHub delivery or complete rebuild
recovery. Team PR review/merge and applicable DoD remain separate from trial proof.
