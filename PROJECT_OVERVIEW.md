# Project Overview: Salesforce DevOps Pipeline

A portfolio project that documents and implements one controlled release path for Salesforce metadata, plus the operational work around it: data handling, rollback, and failure triage. It is built for Developer Edition orgs and is honest about what that does and doesn't allow.

| | |
|---|---|
| **Type** | Technical portfolio repository. Not a product and not a client engagement. |
| **Org model** | Developer Edition Org A (Dev Hub and production stand-in), Developer Edition Org B (staging stand-in), and a Developer scratch org |
| **Status** | Six milestones complete as repository deliverables. Locally validated by 83 Python tests and 8 Node tests. Not yet run against a Salesforce org. |
| **Related** | [README](README.md) · [Architecture](ARCHITECTURE.md) · [Technical design](TECHNICAL_DESIGN.md) · [Case study](CASE_STUDY.md) |

---

## 1. Business problem

Salesforce orgs change constantly, and much of that change is configuration rather than code. An admin adds a field in Setup, a developer edits a trigger, and someone else updates a flow. Without a release discipline, those changes reach the org that matters through manual steps that nobody can repeat exactly.

The problem chain this project addresses is the one the project names: **manual releases → human errors → failed deployments → slow recovery.**

- **Manual releases.** Changes are moved by hand, so the same release can't be repeated or audited.
- **Human errors.** A missing component, a skipped test, or the wrong target org is caught late, or not at all.
- **Failed deployments.** A deploy fails partway or lands with untested code, and nobody knows which commit was live.
- **Slow recovery.** There's no known-good point to return to, and data and metadata recovery get confused with each other.

## 2. Salesforce release risks

These risks shaped specific controls in the repository.

| Risk | Why it matters on Salesforce | Control in this repository |
|---|---|---|
| Untracked Setup changes | Declarative edits live in the org, not in Git | Source tracking on a scratch org, then retrieve and commit (`docs/source_tracking.md`) |
| Low or missing test coverage | The platform enforces 75% only on production deploys, and other orgs accept less | An 85% per-class and per-trigger gate on every pull request (`scripts/coverage_gate.py`) |
| Static-analysis findings shipped | Critical or High issues reach the org unnoticed | Code Analyzer `Recommended` rules, failing on severity 1 and 2 |
| Deploying unvalidated metadata | A failed deploy can leave an org half-changed | A check-only `sf project deploy validate` on every PR, with nothing saved |
| Accidental production deploys | A push or a mis-click releases code | The production workflow runs only from `workflow_dispatch`, and only after the operator types `deploy` |
| No known-good restore point | Rollback becomes guesswork | An annotated `release-YYYYMMDD-HHMM` tag after each smoke-checked release, and `scripts/rollback.sh` |
| Confusing data and metadata recovery | Redeploying metadata doesn't bring back records | Rollback is metadata-only and says so. Data restore is a separate script that re-inserts checklist rows only. |
| Restoring the wrong data | User records, owners, and IDs don't transfer safely between orgs | The restore never inserts users and strips `Id`, `OwnerId`, and other system columns |
| Slow diagnosis of failures | Raw deploy JSON is long and noisy | Rules-based triage that names the category, the cause, and a next step, behind a human review gate |
| Credential leakage | Auth URLs carry refresh tokens | Auth URLs are GitHub secrets, written to a `umask 077` temp file and deleted after login. `.sf/`, `.sfdx/`, and backups are gitignored. |
| Edition surprises | Developer Edition refuses some operations, such as quick deploy | The quick-deploy refusal is detected, a normal deploy runs instead, and the reason is recorded in `reports/quick-deploy.json` |

## 3. Technical operations responsibilities

The repository is organised around what a Salesforce technical operations or release engineer is accountable for day to day.

| Responsibility | What it means in practice | Where it lives |
|---|---|---|
| Environment management | Dev Hub setup, scratch org creation and quotas, org aliases, limits checks | `docs/devhub_setup.md`, `docs/scratch_orgs.md`, `docs/RUNBOOK.md` |
| Change capture | Retrieving declarative changes into source control | `docs/source_tracking.md` |
| Quality gates | Static analysis, Apex tests, coverage | `.github/workflows/pr-validate.yml`, `scripts/coverage_gate.py` |
| Release execution | Staging deploys, controlled production releases, smoke tests | `deploy-staging.yml`, `deploy-prod.yml`, `scripts/production_deploy.sh`, `scripts/smoke_check.sh` |
| Release records | Tags, artifacts, the record of why quick deploy didn't run | `scripts/release_tag.py`, the `production-release` artifact |
| Recovery | Metadata rollback and data restore, kept separate | `scripts/rollback.sh`, `scripts/restore-plan.sh` |
| Operational data | Seeding, bulk loads with error files, nightly backups, a backup catalog | `scripts/seed.sh`, `scripts/bulk-load.sh`, `scripts/backup.sh`, `docs/data_operations.md` |
| Tool evaluation | Comparing the CLI path with Lifecycle Center and Flosum without overstating either | `docs/release-comparison.md`, `docs/tool-selection-guide.md` |
| Incident support | Classifying a failed validation and proposing a next step | `triage/triage.py`, `docs/ai_triage.md` |
| Runbooks | Expected results, failure modes, and recovery procedures | 19 docs and runbooks |

## 4. How this project solves them

| Problem | Solution | Status |
|---|---|---|
| Manual releases | Three GitHub Actions workflows: validate on PR, deploy staging on merge, release production manually | Implemented. Not run against an org. |
| Human errors | Code Analyzer, check-only validation, an 85% coverage gate, five smoke checks, and a typed approval | Implemented. Gate logic tested locally. |
| Failed deployments | Validate before deploy. Quick deploy is attempted only after a successful validation. Other quick-deploy errors don't fall back to a deploy. | Implemented. Decision logic tested locally. |
| Slow recovery | Annotated release tags, a metadata rollback by tag, and a separate data restore plan | Implemented. Rollback dry-run tested locally. |
| Slow diagnosis | Eight-category rules triage with a mandatory human review sentence | Implemented. 15 of 15 fixtures correct. |
| Tool choice | A written three-way comparison and a selection guide | Documented. Lifecycle Center and Flosum are Studied, not operated. |

### What makes the approach credible

1. **Evidence labels everywhere.** Each capability is marked Implemented, Studied, or Not available. The docs list what was *not* executed as plainly as what was built.
2. **Fail-closed defaults.** The coverage gate fails on malformed or missing input. The job-ID parser refuses a failed validation. Rollback refuses an unknown tag. Backup refuses to overwrite a day's folder. Restore refuses to apply without a backup and a target.
3. **Separation of concerns.** Metadata and data recovery are different scripts with different promises. The triage tool can read, but it can't act.
4. **Local verifiability.** Every gate and decision script runs against committed fixtures with the Python standard library. A reviewer can check the logic in minutes without Salesforce access.

## 5. Expected audience

| Audience | What they should take from it | Start with |
|---|---|---|
| Recruiters | The role fit and the honest scope | [SUMMARY_FOR_RECRUITERS.md](SUMMARY_FOR_RECRUITERS.md) |
| Hiring managers | Engineering judgment, controls, and the evidence boundary | [SUMMARY_FOR_HIRING_MANAGERS.md](SUMMARY_FOR_HIRING_MANAGERS.md) |
| Salesforce architects | Org topology, DX choices, Developer Edition trade-offs | [ARCHITECTURE.md](ARCHITECTURE.md) |
| Technical operations engineers | Runbooks, recovery procedures, data handling | `docs/RUNBOOK.md`, `docs/data_operations.md` |
| DevOps and release engineers | Workflows, gates, tags, rollback | [TECHNICAL_DESIGN.md](TECHNICAL_DESIGN.md) |
| Release managers | Approval, release record, tool comparison | `docs/deployment_runbook.md`, `docs/release-comparison.md` |
| Platform engineers | Script design, idempotency, guardrails, testing | `scripts/`, `tests/` |
| Interviewers and candidates | Questions and walkthrough | [INTERVIEW_GUIDE.md](INTERVIEW_GUIDE.md), [DEMO_SCRIPT.md](DEMO_SCRIPT.md) |

## 6. Scope boundary

**In scope:** Salesforce DX source format, scratch orgs, source tracking, Apex tests, Code Analyzer, check-only validation, coverage gating, staging and production stand-in deploys, quick deploy with a fallback, tags, metadata rollback, synthetic data operations, a release-tool comparison, and advisory triage.

**Out of scope, with the reason:**

| Not included | Why |
|---|---|
| Enterprise, Unlimited, or Performance edition orgs | Not available. Both long-lived orgs are Developer Edition. |
| Sandboxes and sandbox refresh | Developer Edition orgs can't create sandboxes |
| A real production org | None exists. Org A is a stand-in. |
| Lifecycle Center hands-on | Not enabled. Its documented declarative path needs a sandbox. |
| Flosum hands-on | No tenant and no subscription |
| Unlocked or managed packages | Not part of the design. The sample has no namespace. |
| A language-model call | The triage explanation is a local template |
| Real customer data | All data is synthetic (`Seed:`, `Bulk:`, Synthetic Seed) |
