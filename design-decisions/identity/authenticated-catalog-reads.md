# Allow Authenticated Version and Channel Catalog Reads Outside Cedar Authorization

***Scope***: GCP-HCP

**Date**: 2026-10-06

## Decision

Version and Channel catalog collection and item reads require a valid public API
identity but do not require a Cedar `RoleBinding`. This decision explicitly exempts
only their `get` and `list` operations from Cedar authorization; writes,
watches, and every other resource remain subject to the existing authorization
path. This is a catalog-specific exception, not a general authorization model
for non-namespaced resources.

## Context

- **Problem Statement**: `gcphcpctl` validates a requested Version before
  creating a Cluster. Version and Channel are non-namespaced, shared catalog
  resources, so their routes have no namespace. The existing Cedar model is
  intentionally namespace-pinned: a `RoleBinding` grants a permission only in
  its namespace. Without an explicit authorization rule, the middleware fails
  closed for direct non-namespaced catalog routes.
- **Constraints**: Authentication remains mandatory through ESPv2. The current
  Cedar entity and permission model contains only namespace-scoped grants;
  non-namespaced catalog authorization would require a separate grant model
  that is not tied to a namespace.
- **Assumptions**: Release and channel metadata is appropriate for every
  authenticated customer to read.
- **Non-goal**: This decision does not define how Cedar authorizes future
  non-namespaced resources. That requires a separate resource-scope and grant
  model decision.

## Alternatives Considered

1. **Bypass Cedar authorization for authenticated catalog reads**: Require an
   ESPv2-authenticated identity, then bypass Cedar authorization for a fixed
   allowlist of Version and Channel collection and item `get`/`list`
   operations. Every authenticated caller is allowed these reads without a
   Cedar RoleBinding. This is the chosen approach.
2. **Authorize catalog reads through Cedar**: Model Version and Channel as
   non-namespaced Cedar resources, add actions for their `get` and `list`
   operations, and evaluate every catalog request through Cedar. This requires
   extending the current namespace-scoped authorization model with a
   non-namespaced resource and policy scope. It would retain Cedar's
   policy-controlled access decisions for catalog access, but establishes
   authorization semantics outside the current namespace-isolated model. It must
   define who receives catalog access and how that non-namespaced grant is
   administered.

| Alternative | Benefits | Trade-offs |
| --- | --- | --- |
| Bypass Cedar authorization for authenticated catalog reads (chosen) | Meets the pre-create validation use case without a namespace binding or non-namespaced Cedar model; keeps ESPv2 authentication mandatory; narrowly expresses that the catalog is shared metadata. | Cedar cannot grant, revoke, or record a determining policy for these reads; changing access requires changing and deploying the application rule. |
| Authorize catalog reads through Cedar | Every catalog read follows the same policy engine; access can be changed through Cedar policy and can support policy-controlled allow/deny decisions. | Introduces a non-namespaced authorization scope alongside the existing namespace-isolated model, and requires explicit catalog grant and administration semantics. |

## Decision Rationale

* **Justification**: Version validation is a prerequisite to cluster creation,
  and the catalog describes shared platform capabilities rather than
  customer-owned resources. A caller needs an applicable `cluster.create` grant
  in the target namespace, whether through `cluster-admin` or a user-defined
  Role, but that namespace-bound grant cannot authorize the non-namespaced
  catalog route in the current model.
* **Evidence**: The current Cedar decision defines only `User`,
  `NamespaceRole`, and `Namespace` entities and explicitly pins generated
  policies to a namespace. The [CLI calls the non-namespaced
  `/versions/{name}` route](https://github.com/openshift-online/gcp-hcp-ctl/blob/main/pkg/cluster/create.go)
  before it derives the namespace for Cluster create.
* **Comparison**: Non-namespaced catalog authorization would preserve
  Cedar-controlled access decisions, but introduces a non-namespaced policy
  scope alongside the existing namespace-isolated model, with corresponding
  policy and operational complexity. The narrow application rule keeps shared
  catalog metadata outside namespace authorization while retaining mandatory
  authentication.

## Consequences

### Positive

* Every authenticated caller can validate Versions and discover Channels before
  cluster creation, without a pre-existing namespace RoleBinding.
* Existing namespace-scoped Cedar policy generation, role bindings, and
  cross-namespace filtering remain unchanged.
* The exception is explicit, generated from resource metadata, and restricted
  to non-mutating operations.

### Negative

* Catalog reads cannot be revoked for an individual caller through Cedar while
  the exemption exists.
* Removing catalog access requires changing the exemption and deploying it, not
  editing a RoleBinding or Cedar policy.

## Cross-Cutting Concerns

### Security

* ESPv2 authentication and Gecko identity validation run before the Cedar
  exemption. Requests without a valid identity remain denied.
* The exemption is limited to public Version and Channel `get` and `list`
  operations. Generator validation rejects mutating exempt verbs; watch is not
  declared exempt for either catalog resource. Customer-owned resources are not
  eligible for this exception.
* Rate limiting and user bans are separate concerns. This decision does not
  depend on a particular edge or application control, and adding one does not
  require broadening the catalog exemption. If audit for exempt reads is
  required, it must come from request/authentication observability rather than
  a Cedar determining-policy record.
* Reassess this decision if catalog fields become sensitive,
  customer-specific, or require per-principal policy control. In that case,
  extend Cedar with an explicit non-namespaced-resource authorization model
  rather than adding further exemptions.

### Operability

* Public catalog access continues to depend on the ESPv2 trust boundary: the
  application listener must not be directly reachable outside the proxy.
* The exemption remains visible in the resource definition and is covered by
  generator, middleware, and public API tests.

## Related

* [Cedar-based public API authorization](cedar-public-api-authorization.md)
* [Cedar authorization implementation plan](../../implementation-plans/gcp-cedar-public-api-authorization.md)
* [GCP-1266](https://redhat.atlassian.net/browse/GCP-1266)
