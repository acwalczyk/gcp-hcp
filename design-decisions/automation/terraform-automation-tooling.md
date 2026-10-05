# Terraform Automation Tooling: HCP Terraform and Progressive Promotion

***Scope***: GCP-HCP

**Original decision**: 2025-10-10 (Atlantis; superseded)
**Current decision update**: 2026-10-05 (GCP-945)

## Current decision: HCP Terraform with GitOps Promoter

HCP Terraform is the current Terraform execution platform. The Atlantis choice below is historical: the migration completed for Integration and Staging, the Atlantis runtime was removed, and the current workspace and cutover details are recorded in the [GCP-532 implementation plan](../../implementation-plans/gcp-532-atlantis-to-tfc-cutover.md). Terraform changes now flow through GitOps Promoter's environment branches before they reach the HCP Terraform workspaces. This progressive promotion model postdates the original GCP-945 investigation.

GitOps Promoter already propagates Terraform revisions through environment branches. HCP Terraform workspaces track those branches and run plans/applies when promotion updates them, so the old proposal to conditionally approve equivalent plans in each environment does not fit the current delivery flow. Do not add a second OPA conditional-apply gate for ordinary promotion.

There is a narrower gap to verify: current GitOps Promoter configuration includes timer and ArgoCD health statuses, but not an HCP Terraform apply-success status. The [GCP-985 story](https://redhat.atlassian.net/browse/GCP-985) explicitly deferred that gate. Promotion ordering exists today; whether an environment's successful Terraform apply gates advancement to the next environment is a separate question and should be tracked against GCP-985.

An initial local OPA canary-rule experiment rejected ordinary historical plans. That experiment was removed and is not recommended for attachment. Any additional conditional plan-action gate should be considered only if owners identify a safety requirement that progressive promotion does not already meet; it is separate from the existing pure-delete policy.

The existing OPA deletion-protection policy is a separate resource-safety control. The [`infra-platform` PR #206](https://github.com/openshift-online/infra-platform/pull/206) expands the tenant configuration to target the Integration, Staging, Production, Org, Tooling, Commons, meta, and Test projects, plus the bootstrap workspace. The `gcp-hcp-ci` workload project stays on its separate CI security policy set; the shared `meta-gcp-hcp` project remains in scope, including its CI workspace-definition workspace. The CI security set retains its owner-grant and folder-IAM rules, but does not include the deletion policy, so CI is an explicit exception to this shared deletion-protection scope. The existing Rego policy rejects pure managed-resource deletes unless an unexpired, address-specific approval is supplied; it does not reject replacement actions. The policy set is configured as overridable.

PR #206 leaves replacement matching unchanged. A failed mandatory OPA check halts the affected HCP Terraform run until an authorized override or a passing plan ([HCP Terraform policy enforcement](https://developer.hashicorp.com/terraform/cloud-docs/policy-enforcement/manage-policy-sets)). Integration and Staging workload runs auto-apply, and current GitOps Promoter configuration has no HCP apply-success status. Do not assume a failed Terraform run stops branch promotion. Assess historical replacement plans and the promotion/apply interaction before proposing that broader gate.

Project targeting protects HCP remote-execution runs only. The current Org, Tooling access, Commons access, and Production access workspaces use local execution and do not receive HCP remote OPA evaluations. Production currently has no workload workspace; adding the Production project covers future remote workspaces but does not change apply settings. Integration and Staging workload workspaces remain auto-apply, and passing this deletion policy does not enable auto-apply. The HCP connector cannot enumerate project-level policy attachments, and sampled run details did not expose policy evaluations, so the source scope must not be described as verified live enforcement until actual run evaluations confirm it.

Org Terraform remains a local, human-operated workflow. Production workload applies remain manual; the current Production access workspace is local and no Production workload workspace exists. Policy checks do not change workspace `auto_apply` settings.

### Policy framework comparison

| Mechanism | Fit for GCP-945 | Trade-off |
|---|---|---|
| **OPA** | The existing Rego deletion policy can inspect current-run plan actions. A separate conditional-apply rule could also inspect plan attributes, but it would not compare a plan with another workspace's historical plan. | A second conditional gate after PR review/promotion duplicates GitOps Promoter's current delivery role and needs an allowlist; it is not selected. |
| **Sentinel** | Can inspect plan data and enforce equivalent workspace/environment-specific rules. | Introduces a second policy language and duplicates the existing Rego controls; it does not make a prior workspace plan available for comparison. |
| **Post-plan run task** | Possible if owners still require an exact comparison between plans from different workspaces or a separate external approval service. | Requires an external endpoint, authentication, secure plan handling, availability, and run-task lifecycle/configuration. |

OPA and Sentinel evaluate Terraform plan data using their own framework-specific input formats; OPA uses its documented `input.plan`/`input.run` data, not Sentinel's `tfplan/v2` import. A run task can call an external system after planning, but the comparison logic and its operational dependencies would still need to be built.

### Rollout gates and baseline

- No new conditional-apply policy is selected or proposed for attachment.
- Recent HCP runs for Integration and Staging report `auto_apply = true`; sampled run details contain no policy-check relationships. The policy lookup connector reports direct/workspace/global attachments but cannot enumerate project-scoped attachments, and the HCP UI was not accessible in this audit. Therefore live project attachment, override permissions, and successful/failed policy evaluations remain unverified. Source configuration sets the deletion policy set to overridable; the identity authorized to use an HCP override still needs live verification.
- The non-production Integration access workspace `gcp-hcp-int-tfc-access` is remote with `auto_apply = false`; it is not a workload canary for the auto-applied Integration workload runs.
- Production currently has no workload workspaces; the Production access workspace is local/manual. The Production meta-workspace is an HCP workspace-definition runner, not a Production workload apply. Keep future Production workload applies manual until separately approved.
- The infra-platform change adds project-level deletion-policy scope for Integration, Staging, and Production alongside Org, Tooling, Commons, meta, and Test, plus the bootstrap workspace. The `gcp-hcp-ci` workload project remains excluded under its separate CI security policy; `meta-gcp-hcp` remains included. This is the intended/source-configured scope until HCP run evaluations verify the live project attachments and policy results.

The GCP-945 proposal is superseded for routine cross-environment delivery by GitOps Promoter. The remaining follow-up is to verify and, if still absent, implement the GCP-985 HCP apply-success status gate. The existing deletion-protection policy is separate; [infra-platform PR #206](https://github.com/openshift-online/infra-platform/pull/206) expands its project scope and leaves its pure-delete matching unchanged.

---

## Historical decision: Atlantis (superseded 2026-10-05)

### Decision

The GCP-HCP platform will adopt Atlantis as the pull request automation tool for Terraform workflows. Atlantis will be self-hosted on a global GKE cluster per environment (integration, stage, production), providing automated plan/apply workflows within the PR lifecycle while maintaining complete control over security and infrastructure. Complementary automation tooling may be added in the future for Terraform operations outside of the pull request context (e.g., scheduled operations, testing pipelines).

## Context

### Problem Statement

The GCP-HCP platform requires an automated Terraform workflow that:
- Provides PR-based plan/apply automation to streamline infrastructure changes
- Maintains security-first principles with controlled secret management
- Supports OpenTofu alongside Terraform for tooling flexibility
- Displays plan output directly in PRs for efficient developer feedback
- Minimizes vendor lock-in and operational costs
- Enables self-hosting for complete infrastructure control

### Constraints

- **Security:** Must support workload identity federation and avoid credential proliferation
- **Cost:** Preference for open-source solutions
- **Infrastructure:** Team comfortable with self-hosting on GKE or Cloud Run
- **Scale:** Must support multiple environments (integration, stage, production) and regional deployments

### Assumptions

- A global GKE cluster per environment will be created to host Atlantis
- GitHub service accounts with appropriate PATs will be configured for Atlantis
- Google Secret Manager will store credentials accessed via External Secrets Operator
- CNCF's stewardship of Atlantis ensures continued community support

## Alternatives Considered

### 1. **Atlantis (Selected)**

Open-source, self-hosted Terraform PR automation tool recently donated to CNCF (June 2024). Runs as a dedicated server that listens to GitHub webhooks and executes Terraform commands based on PR comments.

**Key Characteristics:**
- Self-hosted dedicated server (runs on GKE or Cloud Run)
- Recently donated to CNCF (June 2024), ensuring community governance
- Active development with regular releases
- Executes Terraform within the same pod as the API server
- Inline plan output in PR comments
- PR-level locking (locks entire PR lifecycle vs. apply-time only)

### 2. **Digger**

Open-source CI-native Terraform automation that runs within existing GitHub Actions workflows.

**Key Characteristics:**
- Executes Terraform via CI jobs (GitHub Actions)
- Free unlimited tier with optional $50/month support plan
- PR-level locking (locks entire PR lifecycle vs. apply-time only)
- Inspired by Atlantis design patterns

**Why Not Selected:**
- Sustainability concerns around unlimited free tier business model
- Younger project without established governance (no large foundation backing)
- Security model distributes Terraform execution across CI infrastructure

### 3. **Google Cloud Infrastructure Manager**

GCP's managed service for Terraform automation using Cloud Build orchestration.

**Key Characteristics:**
- Fully managed by Google (no infrastructure to maintain)
- IAM-based authentication with service accounts
- Terraform-only (no OpenTofu support)

**Why Not Selected:**
- No inline PR plan feedback (requires navigating to Cloud Build)
- Complex setup requiring GitHub App, PAT in Secret Manager, and per-environment configuration
- Does not support OpenTofu
- Poor UX compared to inline alternatives
- Multi-deployment trigger pollution (all environments trigger on every PR)

### 4. **Terraform Cloud**

HashiCorp's SaaS platform for enterprise Terraform automation.

**Key Characteristics:**
- Fully managed SaaS offering
- Private agents run in your infrastructure
- Enterprise support and SLA guarantees
- Polished UX and feature set

### 5. **Tekton**

Open-source CI/CD framework and Continuous Delivery Foundation project for creating flexible, cloud-native pipeline systems.

**Key Characteristics:**
- Event-driven architecture with EventListeners for webhook triggers
- Highly flexible and standardizes CI/CD across vendors and platforms
- Reusable Tasks (e.g., terraform-cli task) for pipeline composition
- Kubernetes-native (runs on existing GKE clusters)

**Why Not Selected:**
- Requires more setup to achieve Atlantis "out of the box" functionality
- Would need custom configuration for:
  - Event triggers to respond to GitHub PR webhooks
  - Writing plan output to PR comments
  - State locking management across PRs
  - PR lifecycle automation (plan on push, apply on merge)
- Higher operational complexity for this specific use case
- Better suited for broader CI/CD automation rather than Terraform-specific workflows

**Note:** Tekton is being evaluated separately for automated pipeline tooling (particularly for nightly end-to-end tests) and may be integrated alongside Atlantis in the future for complementary use cases.

## Decision Rationale

### Justification

Atlantis provides the optimal balance of security, cost, community support, and operational control for the GCP-HCP platform's Terraform automation needs. Its recent donation to CNCF (June 2024) demonstrates renewed community commitment and governance stability, addressing previous concerns about the project's future during HashiCorp ownership.

The self-hosted architecture aligns with the team's security-first principles by:
- Isolating Terraform execution to controlled pods with specific service account permissions
- Enabling per-environment deployments with isolated credentials (integration, stage, production)
- Avoiding credential distribution across multiple CI systems

### Evidence

**POC Validation:**
- Successfully tested workload identity federation and GCS bucket creation
- Verified inline plan output in PR comments
- Confirmed state locking mechanisms prevent concurrent modifications
- Validated compatibility with team's GitHub workflow

**Community Health:**
- CNCF sandbox project (donated June 2024)
- Active development with commits as recent as October 2025
- Regular releases (latest: 3 weeks ago as of October 2025)
- Community meetings and established governance

**Team Meeting Consensus (October 10, 2025):**
- Atlantis features influenced Digger's design, demonstrating proven patterns
- CNCF donation indicates long-term sustainability
- Self-hosting requirement is manageable with Cloud Run or GKE options
- Security model superior to CI-distributed approach for team's use case

### Comparison

| Criterion | Atlantis | Digger | Infrastructure Manager | Terraform Cloud |
|-----------|----------|--------|----------------------|-----------------|
| **Cost** | Self-host only | Free/Self-host | GCP service costs | Paid subscription |
| **Community** | CNCF (established) | Independent (newer) | Google-backed | HashiCorp (enterprise) |
| **PR Output** | Inline comments | Inline (prettier) | External (Cloud Build) | External link |
| **OpenTofu** |  Supported |  Supported | L Not supported | L Not supported |
| **Security Model** | Dedicated pod | CI-distributed | IAM/Service accounts | Private agents |
| **Infrastructure** | Self-managed | Uses CI | Fully managed | SaaS + agents |
| **Setup Complexity** | Medium | Low | High | Low |
| **Reliability** | High | High | Low (preview failures) | High |

Atlantis was selected over Digger because:
- CNCF governance provides better long-term stability than unlimited free tier business model
- Dedicated service architecture preferred over CI-distributed execution for security isolation
- More established project with proven production usage

Atlantis was selected over Infrastructure Manager because:
- Inline PR feedback significantly improves developer experience
- OpenTofu support provides tooling flexibility
- Simpler setup and better reliability (preview jobs work consistently)

Atlantis was selected over Terraform Cloud because:
- Zero ongoing subscription costs
- Complete control over infrastructure and data
- Inline PR comments (vs. external links)

## Consequences

### Positive

- **Zero Licensing Costs:** Open-source solution with no subscription fees
- **Complete Infrastructure Control:** Self-hosted architecture provides full control over security and data
- **Superior Developer Experience:** Plan output directly in PR comments eliminates context switching
- **Community-Backed Stability:** CNCF governance ensures long-term project sustainability
- **Security Isolation:** Dedicated pods per environment isolate credentials and limit blast radius
- **Flexible Deployment:** Multiple options (GKE, Cloud Run) enable cost/complexity optimization
- **OpenTofu Support:** Not locked into Terraform-only tooling
- **Proven Reliability:** Established tool with predictable behavior and extensive documentation
- **Automated Workflows:** State locking and PR-based automation streamline infrastructure changes
- **Workload Identity Compatibility:** Verified support for GCP workload identity federation

### Negative

- **Self-Hosting Overhead:** Team must maintain Atlantis infrastructure (monitoring, upgrades, availability)
- **Initial Setup Complexity:** Requires bootstrapping global clusters, GitHub service accounts, and webhook configuration
- **Less Polished UX:** Plan output formatting not as refined as some alternatives (e.g., Digger)
- **Infrastructure Dependencies:** Requires persistent server infrastructure (global GKE cluster per environment)
- **Public Endpoint Requirement:** GitHub webhooks require publicly accessible endpoint (or ngrok for development)
- **Manual Configuration:** Atlantis YAML file in repo root requires updates when adding new Terraform workspaces
- **Limited Scheduled Operations:** No native support for scheduled provision/destroy workflows (requires external automation)
- **Backlog of Open Issues:** Period of uncertainty during HashiCorp ownership created backlog that may impact feature velocity

## Cross-Cutting Concerns

### Reliability

**Scalability:**
- Atlantis runs as a single-instance service per environment (not designed for horizontal scaling)
- Each environment (integration, stage, production) has isolated Atlantis deployment
- Cloud Run option available for serverless scaling if needed
- Workload scales with PR velocity, not infrastructure size
- Repository-level atlantis.yaml defines Terraform workspace paths, triggering only relevant plans

**Observability:**
- GUI interface provides job monitoring and configuration visibility
- Plan/apply output logged directly in PR comments for auditing
- State lock status visible through Atlantis web interface
- GitHub webhook delivery monitoring required to detect failures

**Resiliency:**
- Single point of failure per environment (mitigated by global cluster per environment)
- State locking prevents concurrent modification conflicts
- Automatic unlock on PR merge prevents dangling locks
- Global cluster setup ensures Atlantis availability separate from regional clusters
- No SLA guarantees (self-hosted responsibility)

### Security

- **Authentication:** GitHub webhooks authenticated via shared secret stored in Google Secret Manager
- **Authorization:** GitHub service account PATs with minimal permissions (PR read/write for specific repositories)
- **Secret Management:** Credentials stored in Google Secret Manager, accessed via External Secrets Operator using workload identity
- **Workload Identity:** Atlantis service account assumes GCP permissions via workload identity federation (no credential keys)
- **Network Security:** Identity-Aware Proxy (IAP) protects Atlantis GUI with Google authentication and RBAC
- **Isolation:** Per-environment Atlantis deployments isolate credentials (integration cannot access production)
- **Least Privilege:** Service accounts scoped to minimum permissions required for Terraform operations
- **Audit Trail:** All Terraform operations logged in PR comments and GitHub audit logs

### Performance

- **Latency:** Plan/apply operations run directly in Atlantis pod (no external job queuing)
- **Concurrency:** State locking serializes operations per Terraform workspace (prevents conflicts)
- **Resource Utilization:** Terraform execution shares pod resources with API server (potential resource contention)
- **Network Performance:** Atlantis pod network access to GCP APIs and GitHub
- **Optimization:** Autopilot GKE option provides right-sized compute for Atlantis workloads

### Cost

- **Compute Costs:** GKE cluster per environment (integration, stage, production) - mitigated by autopilot for cost optimization
- **Networking Costs:** Egress charges for GitHub webhook traffic and Terraform API calls
- **Storage Costs:** GCS buckets for Terraform state storage (minimal)
- **Operational Costs:** Team time for maintenance, upgrades, and troubleshooting
- **No Subscription Fees:** Zero licensing costs compared to Terraform Cloud
- **Cost Optimization Opportunities:**
  - GKE Autopilot reduces idle compute costs
  - Cloud Run option for serverless cost model
  - Shared global cluster may host Atlantis + ArgoCD

### Operability

- **Deployment:** Helm chart or Cloud Run deployment via Terraform (bootstrapped manually per environment)
- **Maintenance:** Requires regular Atlantis version upgrades (community releases cadence)
- **Configuration Management:** Atlantis YAML file in repository root requires updates for new Terraform workspaces (make target planned)
- **Monitoring Requirements:** Need alerting for webhook failures, plan/apply errors, and state lock issues
- **Secret Rotation:** GitHub PATs and webhook secrets require rotation procedures
- **Backup/Recovery:** Terraform state in GCS (versioned buckets), Atlantis configuration in GitOps repository
- **Troubleshooting:** GUI interface simplifies debugging, but requires access through IAP authentication

---

## Template Validation Checklist

### Structure Completeness
- [x] Title is descriptive and action-oriented
- [x] Scope is GCP-HCP
- [x] Date is present and in ISO format (YYYY-MM-DD)
- [x] All core sections are present: Decision, Context, Alternatives Considered, Decision Rationale, Consequences
- [x] Both positive and negative consequences are listed

### Content Quality
- [x] Decision statement is clear and unambiguous
- [x] Problem statement articulates the "why"
- [x] Constraints and assumptions are explicitly documented
- [x] Rationale includes justification, evidence, and comparison
- [x] Consequences are specific and actionable
- [x] Trade-offs are honestly assessed

### Cross-Cutting Concerns
- [x] Each included concern has concrete details (not just placeholders)
- [x] Irrelevant sections have been removed
- [x] Security implications are considered where applicable
- [x] Cost impact is evaluated where applicable

### Best Practices
- [x] Document is written in clear, accessible language
- [x] Technical terms are used appropriately
- [x] Document provides sufficient detail for future reference
- [x] All placeholder text has been replaced
- [x] Links to related documentation are included where relevant

**See also**: [Deployment Tooling Policy](./deployment-tooling-swim-lanes.md) — Atlantis automates Lane 1 (Terraform) infrastructure changes via PR-based plan/apply workflows.
