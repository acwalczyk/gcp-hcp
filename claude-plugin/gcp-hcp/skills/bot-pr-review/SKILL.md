---
name: bot-pr-review
description: >
  Use when explicitly asked to perform a first-pass pull request review for GCP
  HCP team repositories or commonly contributed upstream repositories.
argument-hint: "[PR URL]"
allowed-tools: Read, Grep, Glob, mcp__github__get_me, mcp__github__get_file_contents, mcp__github__pull_request_read
disable-model-invocation: true
---

# GCP HCP Pull Request Review

Use this skill for a first-pass review of a GitHub pull request. The review has
two separate tracks:

1. **Code review:** find correctness, security, test, operational, and
   architectural problems in the proposed change.
2. **Merge readiness:** explain what the repository and current PR state still
   require before the PR can merge.

Do not collapse these tracks into a single “looks good” conclusion. A PR can
have no substantive code findings and still be blocked by labels, CI, review,
or repository workflow requirements.

## Operating rules

- Treat the PR's current head SHA, base branch, labels, reviews, checks, and
  comments as time-sensitive evidence. Record the head SHA used for the review.
- Read the complete diff and changed-file list before forming a conclusion. Do
  not review only the PR title, description, or a bot summary.
- Read repository-local `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, testing
  guidance, and the closest applicable directory guidance before judging style,
  tests, or workflow requirements. Follow the closest file when guidance is
  nested.
- Use the repository's current branch and CI configuration as the source of
  truth. The profiles below are review prompts, not a substitute for checking
  the live PR or current configuration.
- Never invent a required check, label, Jira rule, or merge blocker. If the
  relevant configuration or GitHub data is unavailable, report **Unknown** and
  say exactly what could not be verified.
- Use only the read-only tools allowed by this skill. If a needed read operation
  is unavailable, report **Unknown**; do not bypass the allowlist or use
  write-capable GitHub tools.
- Do not mutate GitHub state during a first-pass review. Do not add labels,
  approve, request changes, post `/lgtm`, post `/approve`, trigger tests, merge,
  or resolve review threads unless the user separately asks for that action.
- Do not expose credentials, kubeconfigs, Terraform state, private URLs, or
  unbounded command output in the review.
- A first pass is not a substitute for a maintainer approval, a live deployment
  test, an end-to-end run, or a security sign-off.

## Phase 1: Establish scope and evidence

Parse the supplied URL only into `owner/repository` and PR number. Obtain the
base branch and current head SHA from live PR metadata; never infer either from
the URL, a local checkout, the default branch, or an older comment. If the URL
is missing or ambiguous, ask for the PR URL rather than guessing. If live PR
metadata cannot be read, report **Unknown** and stop the merge-readiness
assessment.

Collect, using the available GitHub tools:

- PR metadata: title, author, draft/open state, base branch, head SHA, labels,
  requested reviewers, mergeability, and linked issues.
- Changed files, patch/diff, commits, existing reviews, review threads, and
  issue comments. Distinguish unresolved current comments from outdated or
  resolved comments.
- Commit checks and combined status for the exact head SHA. Capture each
  context's conclusion and URL when available.
- Repository instructions and the workflow/configuration files that explain
  required labels, branch protection, Tide, Prow, Konflux, Tekton, GitHub
  Actions, generated files, or conditional jobs.

Before the final report, re-read the PR head SHA and status if the review took
long enough for new commits or checks to arrive. If the head changed, either
review the new head or clearly scope the report to the earlier SHA and mark
merge readiness stale.

## Phase 2: Determine merge readiness

### Labels and reviews

Separate these categories in the report:

- **Required for this branch:** labels, approvals, or review states confirmed
  by the repository's current configuration or contributor guidance.
- **Present but not sufficient:** labels or approvals that exist but do not
  satisfy every merge gate.
- **Blocking labels:** current `do-not-merge/*`, `needs-rebase`,
  `tide/merge-blocker`, invalid-reference/bug/owners labels, or other blockers
  confirmed for this repository and branch.
- **Informational labels:** labels that describe ownership, area, severity, or
  automation but are not independently proven to block merging.

For Prow/Tide repositories, `approved` and `lgtm` are common gates, but do not
call them required without checking the repo/branch configuration. A pending
Tide context may be expected before `lgtm` and `approved` are present; do not
report that pending context as a failing pre-review check without evidence.

Do not infer that a GitHub approval is equivalent to `lgtm` unless the current
repo configuration says reviews act as LGTM. Report missing approver,
reviewer, or owner coverage separately from missing labels.

### Checks and status contexts

Classify every relevant check as **pass**, **fail**, **pending**, **skipped**,
or **unknown**. Use the exact context name and head SHA. Apply these rules:

- A required-if-present check matters when it is present; its absence alone is
  not proof that the PR is blocked.
- A conditional check may be required only when the changed paths or branch
  activate it. Verify that trigger before calling it missing.
- A check for an older head SHA is stale evidence, not a pass for the current
  PR.
- A skipped or neutral result is not automatically a pass; explain the
  repository's semantics when they are known.
- A successful build/test check does not prove approval, generated-file
  consistency, Jira validity, mergeability, or runtime behavior.
- Treat `/pj-rehearse` acknowledgement, queued CI, a pinned rehearsal, or a
  dry run as an action/status signal, not as a passed result. A rehearsal is
  passed only when the final result succeeded for the current tested commit.

When the repository exposes a Tide status, use its details to identify the
next missing gate. Do not replace Tide's current explanation with a generic
checklist.

### Jira and PR conventions

Check the repository's actual lifecycle configuration and contributor guidance.
For HyperShift, the title must contain a valid Jira reference or a deliberate
`NO-JIRA:` prefix, and the PR normally needs `jira/valid-reference`; bugs and
backports add conditional requirements. Do not apply this exact rule to every
repository: several GCP HCP team repositories use different lifecycle or
release automation.

## Phase 3: Repository-specific review focus

Use the matching profile to decide what to inspect. Verify current details
from the PR and repository configuration before reporting a gate.

### `openshift/hypershift`

- Check Prow and GitHub Actions results, plus labels commonly used here
  (`approved`, `lgtm`, `verified`, `jira/valid-reference`, and an applicable
  `area/*` label). Confirm which labels are required for the PR's base branch
  against the current repository/Tide configuration before reporting one as a
  missing gate. Also check current blockers such as
  `do-not-merge/needs-area` or `needs-rebase`.
- HyperShift contributor guidance says cloud-consuming E2E jobs are gated
  behind `/lgtm`; verify the current job configuration and status before
  treating a pending E2E job before LGTM as a failure or merge blocker.
- Check the Jira-in-title/`NO-JIRA:` convention, imperative conventional commit
  subjects, reviewer/approver coverage, and the `make pre-commit` expectation.
- For changes to pull secrets, ignition, MachineConfigs, or NodePool config
  hashes, review fleet-wide rollout and existing-cluster migration impact.
- Read the closest nested guidance for API, control-plane, E2E, or other
  affected subsystems. Do not assume a root-level review covers those rules.

### `openshift/release`

- Check the generated-source relationship for CI configuration, job
  definitions, OWNERS, and related generated files. Do not accept hand-edited
  generated output without the corresponding source change.
- For changes to ci-operator or Prow job definitions, check the repository's
  required regeneration/validation commands (commonly `make ci-operator-config`
  and `make jobs`) and whether the affected job received a PJ rehearsal.
- Treat `pj-rehearse` as relevant to changed job definitions, not as a blanket
  requirement for unrelated documentation or tooling changes.
- Read the current Prow/Tide configuration for required contexts and whether
  unknown contexts are skipped. Report exact job results for the current SHA.

### `openshift-online/gcp-hcp-infra`

- Read the closest `CLAUDE.md` for `terraform/`, `argocd/`, `secrets/`, or
  other affected areas.
- For Terraform changes, inspect formatting/validation, native Terraform tests,
  affected speculative plans, IAM/WIF behavior, and destructive or
  `prevent_destroy` implications.
- For ArgoCD changes, require rendered-output consistency and inspect sync
  ordering, ownership, and environment overlays. For source-template changes,
  use the repository renderer and review the generated output.
- The repository's common validation surface includes generated-file checks,
  orphan-module checks, GitOps promoter tests, image checks, Terraform tests,
  and Terraform validation. Treat each as path- or configuration-dependent.
  For `ci/prow/terraform-plan`, inspect the current Prow/Tide configuration
  and changed paths before treating it as required. An unchecked implementation
  plan or stale configuration is not evidence that the context is active.
- Treat `e2e-platform` as a separate platform-E2E signal. A requested or
  accepted rehearsal is not a completed passing run; match any success to the
  exact PR head SHA.

### `openshift-online/gecko`

- This repository contains multiple Go modules. Identify the owning module and
  read its nested guidance before reviewing tests or generation.
- Check the repository-level commands and affected-module tests/lint rather than
  running or expecting a root `go test ./...` where no root module exists.
- Its current PR automation is Konflux/Tekton Pipelines-as-Code. Inspect the
  actual build, image, scan, and validation contexts for the changed component;
  do not assume generic `ci/prow/lint` or `ci/prow/test` contexts.
- For platform API or generated artifacts, check the source schema, generation
  command, generated public API, and compatibility implications together.
- For controller or orchestration changes, review idempotency, ownership,
  status/error handling, and cleanup of stale resources.

### `openshift-online/gcp-hcp-ctl`

- Check the affected Go package with the repository's build, race-enabled unit
  test, and lint targets (`make build`, `make test`, and `make lint` where
  applicable).
- Preserve the Zero Operator Access boundary: cluster and control-plane
  operations go through Cloud Workflows. Read-only workflows may be permanently
  available, but sensitive or destructive operations require PAM-approved,
  time-bounded grants. Verify auditable identity/action and Workload Identity
  authentication, with no direct `kubectl`, SSH, pod exec, or long-lived
  credentials. A new destructive CLI/API operation must map to a PAM-gated
  workflow rather than creating a direct-access bypass.
- Review CLI/API compatibility, config precedence, error output, and tests for
  both success and failure paths. Do not infer a live GCP or workflow result
  from a local compile.

### `openshift-online/kube-applier-gcp`

- Read its README and local guidance, then check the affected Go and Helm
  behavior, generated manifests, tests, and deployment assumptions.
- Review reconciliation idempotency, desired-state key lifecycle, stale
  controller cleanup, and least-privilege service-account behavior when those
  areas are touched.

### `openshift-online/gcp-hcp`

- This is primarily a documentation, architecture, planning, and study
  repository; do not invent application CI gates when none are configured.
- For design decisions, check the template, `design-decisions/INDEX.md`, and
  the architecture skill topic index. For architecture or platform changes,
  apply the GCP HCP invariants below and check linked implementation plans.
- Review documents for source-backed claims, consistent terminology, explicit
  scope/non-goals, security implications, and links that resolve within the
  repository.

### Other repositories

If the repository is not listed, identify its local instructions and current
branch/CI configuration first. Report the repository-specific requirements you
could verify and explicitly list the requirements that remain unknown. Do not
copy a profile from a similarly named repository.

## Architecture review

When a PR touches GCP HCP platform code, infrastructure, control planes, or
architecture documentation, use the `gcp-hcp-architecture` skill when the
runtime supports skill invocation. If the lightweight channel runtime cannot
invoke another skill, apply these invariants directly and state that deeper
workspace review may be needed:

- No direct cross-cluster connectivity, except the documented PSC data-plane
  exception.
- Regional independence and data-residency boundaries are preserved.
- Workload Identity Federation is used instead of long-lived service-account
  keys.
- Google Managed Prometheus remains the observability target where applicable.
- ArgoCD manages production deployments; avoid direct production `helm install`
  or `kubectl apply` workflows.
- Terraform manages GCP infrastructure.

Flag violations with the affected file/line, the operational or security
impact, and the smallest safe correction. Do not claim that invoking a skill is
automatic in a channel workflow; only report what the current runtime actually
loaded.

## Phase 4: Code review method

Review the changed code and its surrounding call sites in this order:

1. **Behavior and correctness:** Does the implementation satisfy the PR's
   stated contract? Check error paths, retries, idempotency, concurrency,
   cleanup, resource ownership, API compatibility, and boundary conditions.
2. **Security and reliability:** Check authentication/authorization, secret
   handling, input validation, least privilege, failure isolation, logging,
   data loss, and operational rollback.
3. **Tests and generated artifacts:** Check whether tests exercise the changed
   behavior and failure modes, and whether generated files are reproducible and
   up to date.
4. **Repository conventions:** Apply the closest local guidance, naming,
   formatting, API, deployment, and release conventions.
5. **Architecture:** Apply the architecture section when the change crosses a
   GCP HCP boundary or changes a documented contract.

Report only actionable findings. For each finding include:

- severity (`P0` through `P3`),
- exact file and line or a tight line range,
- the concrete failure mode and impact,
- the smallest fix or decision needed, and
- evidence from the diff or relevant source.

Do not report a style preference as a blocker. If no substantive findings are
found, say so explicitly and state the review scope and remaining validation
limits.

## Required response format

Keep the channel response concise and put evidence behind links where the
runtime supports them.

```markdown
## Merge readiness
- Repository/base/head: `owner/repo`, `base`, `<head SHA>`
- Verdict: `Ready`, `Blocked`, or `Unknown` (first-pass assessment only)
- Required labels/reviews: present, missing, or unverified
- CI/status: exact failing, pending, skipped, and passing contexts for the head SHA
- Blocking conditions: concrete blockers, or `none found`
- Next steps: ordered actions for the author/reviewer

## Code review
- [P1] `path/to/file:line` — failure mode, impact, and smallest fix.
- `No substantive findings in the reviewed diff.`

## Limits
- State anything not verified: inaccessible configuration, pending E2E, stale
  status, unavailable workspace test, or missing runtime/architecture context.
```

Use `Unknown` instead of `Ready` whenever a required source of truth could not
be inspected. “No substantive findings” means only that no actionable defect
was found in the reviewed diff; it does not mean the PR is approved or safe to
merge.

## Integration note

Adding this file to the `gcp-hcp` plugin makes the skill available to runtimes
that install or load this plugin. It does not by itself change a Slack/Chai
workflow. The bot integration must explicitly register or include the skill;
until that is verified in the bot deployment, describe the result as a
repository skill being prepared for integration, not as a live behavior change.
