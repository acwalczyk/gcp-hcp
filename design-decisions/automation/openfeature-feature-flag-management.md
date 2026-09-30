# OpenFeature-Based GitOps Feature Flags

***Scope***: GCP-HCP

**Date**: 2026-09-28

## Decision

We will adopt OpenFeature as Gecko's application-facing feature-flag API and initially use the flagd file provider with Git-managed, environment-scoped definitions delivered through a dedicated ArgoCD feature-flags application as a ConfigMap-mounted file. The flag payload is sourced from [`gcp-hcp-infra/main`](https://github.com/openshift-online/gcp-hcp-infra/tree/main), so flag changes are independent of the Gecko application commit and standard software promotion branches. The platform API server is the first consumer; Gecko controller processes can adopt the same provider path next. ArgoCD remains the deployment authority for both lanes, and App Lifecycle Manager (ALM) is deferred as a possible future OpenFeature provider.

The initial implementation will not depend on ALM or another external flag-management service. Gecko business code must not depend directly on flagd or a specific future provider.

## Context

- **Problem Statement**: Gecko needs to roll out behavior changes independently by environment, including changes such as Cedar authorization enforcement, while retaining the existing GKE, Helm, ArgoCD, and GitOps promotion model. Cedar is the in-process policy engine used to authorize Gecko's public API requests; its design is documented in the [Cedar-based authorization decision](../identity/cedar-public-api-authorization.md).
- **Constraints**: ArgoCD remains the deployment authority. Initial flags are environment-scoped rather than tenant- or region-scoped. The initial solution must work in every Gecko deployment region without a new regional service dependency. Feature definitions must be reviewable in Git, while their payload must not be pinned to a Gecko application SHA.
- **Assumptions**: Initial flag changes should take effect after a reviewed merge to `main` and ArgoCD synchronization, without waiting for the standard software promotion chain. Flag definitions are non-secret. ALM's Preview/Pre-GA status and regional availability do not justify making it an initial dependency. A future requirement for out-of-band runtime changes, tenant targeting, or managed gradual flag rollouts may justify adding ALM or another provider.

## Alternatives Considered

1. **Main-sourced GitOps flagd file with OpenFeature (chosen)**: Store flag definitions in a dedicated chart on `gcp-hcp-infra/main`. A feature-flags ArgoCD Application, whose manifest is managed through the normal environment configuration, deploys that chart as a read-only ConfigMap; flagd evaluates it locally.
2. **Flags coupled to application promotion**: Store the flag payload in the Gecko application chart or promoted `dry_sha`. Rejected because flag changes wait for the software promotion chain and are coupled to the deployed application revision.
3. **Standalone ALM immediately**: Use ALM Units and `saasconfig.googleapis.com` for flag management while keeping ArgoCD for application deployment. This provides managed runtime rollouts but adds a Preview service, IAM, network, resource-model, and regional-availability dependency before those capabilities are required.
4. **Direct Helm values without OpenFeature**: Configure each behavior directly through environment variables or chart values. This is simple initially but couples application code to deployment configuration and makes a later provider migration more invasive.

## Decision Rationale

* **Justification**: OpenFeature separates Gecko evaluation code from flag storage and management. A dedicated ArgoCD flag application lets a reviewed merge to `main` update flags independently of Gecko binaries, while retaining Git auditability and local evaluation. The file provider works in unsupported ALM regions. A future ALM, Unleash, GO Feature Flag, or other provider can be selected through deployment configuration without changing flag call sites.
* **Evidence**: OpenFeature providers are explicitly designed to abstract the underlying flag-management system ([OpenFeature provider concept](https://openfeature.dev/docs/reference/concepts/provider)). The flagd Go provider supports local file evaluation and file refresh ([flagd Go provider](https://flagd.dev/providers/go/)). Gecko's existing deployment swim lanes make ArgoCD the owner of Kubernetes resources ([deployment tooling swim lanes](https://github.com/openshift-online/gcp-hcp/blob/main/design-decisions/automation/deployment-tooling-swim-lanes.md)), and the existing promotion flow already advances environment-specific content through gated branches ([environment promotions](https://github.com/openshift-online/gcp-hcp-infra/blob/main/docs/promotions.md)).
* **Comparison**: Coupling flags to the software promotion chain was rejected because it defeats independent activation. Immediate ALM adoption is deferred because the current requirement does not need ALM's runtime rollouts or targeting, while ALM feature flags are still Preview/Pre-GA and require additional regional, IAM, and network integration ([ALM standalone flags](https://docs.cloud.google.com/app-lifecycle-manager/flags/flags-standalone-quickstart)). Direct Helm flags are rejected because they lose the provider-neutral application boundary.

## Consequences

### Positive

* No ALM API, IAM role, or SaaS Config network dependency is required initially.
* Git review and ArgoCD remain authoritative, while flag changes no longer wait for the application promotion chain.
* Flag evaluation is local to each Gecko process; the request path has no flag-service network dependency.
* OpenFeature keeps application code portable across flagd, ALM, and other providers.
* Unsupported ALM regions require no special provider behavior.

### Negative

* A merge to `main` becomes a production-impacting flag change for environments consuming the flag chart, so flag-specific ownership, review, and rollback controls are required.
* ConfigMap propagation and flagd file refresh are eventually consistent; applications must not assume an atomic fleet-wide change.
* A future provider migration still requires translating definitions, variants, targeting rules, rollout procedures, and operational metadata.
* Flag lifecycle governance is required to prevent stale or permanent flags.

## Cross-Cutting Concerns

### Reliability:

* **Scalability**: Flag evaluation is in-process and does not add a request-time service call. Each process maintains its own local flag state.
* **Observability**: Record the flag configuration revision/commit, provider state, reload failures, evaluation errors, default-value use, flag key, and variant. Do not record sensitive evaluation context by default.
* **Resiliency**: If no provider is configured, callers use their code-supplied defaults; if a configured file cannot be initialized, the application follows its startup failure policy rather than silently selecting a different state. After a valid file is loaded, a malformed refresh retains the last valid state and raises an alert. ConfigMap updates may take time to reach mounted files; do not promise immediate propagation. Do not use `subPath` mounts when live file refresh is required, because ConfigMap files mounted through `subPath` do not receive later ConfigMap updates. Use a deliberate Deployment rollout when restart-time activation is preferred.

### Security:

* ConfigMaps contain only non-secret flag definitions. Credentials, tokens, signing material, and sensitive tenant data remain governed by the [secret-management strategy](https://github.com/openshift-online/gcp-hcp/blob/main/design-decisions/identity/secret-management-strategy.md).

### Performance:

* Local flag evaluation avoids network latency on the API and controller hot paths.
* File refresh intervals and ConfigMap propagation should be measured and configured for the required rollout latency.

### Cost:

* The initial design adds no managed flag-service cost or additional always-on service.
* Future ALM or hosted-provider costs can be evaluated when runtime-management requirements are concrete.

### Operability:

* The flag manifest and chart are the Git source of truth on `main`; the ConfigMap is only their deployment artifact. The dedicated feature-flags ArgoCD Application is environment-aware but its flag payload source is `main`, not `dry_sha` or a Gecko application commit.
* ArgoCD deploys the ConfigMap and application provider configuration. GitOps promoter continues to gate and promote application code and deployment configuration; it does not gate the independently sourced flag payload or provide flag evaluation.
* Provider selection is deployment configuration, for example `file`, `alm`, or `off`. ALM can later be enabled by changing provider configuration and migrating definitions, without changing application evaluation call sites.
* Each flag requires an owner, a documented default, an intended scope, and an expiry/cleanup plan.

#### Flag definition format

The initial source file uses flagd's native JSON definitions schema, not a separate OpenFeature management manifest. OpenFeature defines the evaluation API; flagd defines the local source format ([flagd flag definitions](https://flagd.dev/reference/flag-definitions/)). For example:

```json
{
  "$schema": "https://flagd.dev/schema/v0/flags.json",
  "flags": {
    "gecko.public-api.authorization.enabled": {
      "state": "ENABLED",
      "variants": {
        "enabled": true,
        "disabled": false
      },
      "defaultVariant": "enabled",
      "metadata": {
        "owner": "gecko-platform-api",
        "scope": "environment",
        "expires": "2027-01-31"
      }
    }
  }
}
```

`owner`, `scope`, and `expires` are Gecko governance metadata and must be validated in CI; flagd does not interpret them. `defaultVariant` is the provider-evaluated default.

#### Testing

Unit tests should use OpenFeature's in-memory test provider to set deterministic variants and exercise enabled, disabled, and default/error behavior. The file provider is reserved for provider/configuration integration tests and Helm rendering tests.
