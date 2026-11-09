# Release Notes · v1.0

**Salesforce DevOps Pipeline, portfolio release v1.0**

v1.0 is the first complete snapshot of the repository: milestones M1 through M6. It's a portfolio release, not a production software release. The orgs in the design are Developer Edition orgs, and there's no sandbox, no Enterprise org, and no production tenant.

> v1.0 is a documentation label for this snapshot. No `v1.0` git tag is defined by the pipeline, which tags only production stand-in releases, using the `release-YYYYMMDD-HHMM` format.

---

## Highlights

- **Pull-request gate.** Code Analyzer (severity 1 and 2 fail), a check-only `sf project deploy validate` with `RunLocalTests` against Org B, and an **85% per-class and per-trigger coverage gate**.
- **Controlled release path.** Staging deploys on merge. Production stand-in releases need a manual run where the operator types `deploy`. Quick deploy is tried, and a recorded fallback runs when Developer Edition refuses it.
- **Release records and rollback.** UTC `release-YYYYMMDD-HHMM` tags after smoke checks pass, and a metadata rollback by tag.
- **Operational data handling.** A synthetic seed, a bulk load with an error file, dated CSV and JSON backups with a manifest, and a checklist-only restore plan.
- **Release tool comparison.** Lifecycle Center and Flosum, labelled Studied.
- **Advisory failure triage.** 8 categories, a mandatory human review sentence, and 15/15 on the committed fixtures.

## What's included

### M1 · Salesforce DX foundation
- `sfdx-project.json` (API 68.0, no namespace, `force-app` default) and `config/project-scratch-def.json` (edition `Developer`)
- The Release Checklist Item sample: object, 4 fields, `ReleaseChecklistItemService`, `ReleaseChecklistItemTrigger`, `ReleaseChecklistItemServiceTest` (7 methods, including a 200-row bulk test), the `Stamp_Ready_For_Review` before-save flow, the `checklistItemBadge` LWC, and the `Release_Checklist_Access` permission set
- Runbooks: Dev Hub setup, scratch orgs, source tracking, Apex testing, Developer Edition limits, the main runbook with recovery procedures R1–R8, and the starting-new-development checklist

### M2 · Pull-request validation
- `.github/workflows/pr-validate.yml` on `pull_request`
- `scripts/coverage_gate.py` with exit codes `0`, `1`, and `2`, writing `reports/coverage-summary.md`
- The `pr-validation` artifact (14 days) and `docs/pr_validation.md`

### M3 · Release path
- `.github/workflows/deploy-staging.yml` (push to `main` → Org B) and `.github/workflows/deploy-prod.yml` (`workflow_dispatch` → Org A)
- `scripts/production_deploy.sh`, `quick_deploy_decision.py`, `parse_job_id.py`, `smoke_check.sh`, `release_tag.py`, and `rollback.sh`
- `manifest/package.xml` and an empty `manifest/destructiveChangesPost.xml`
- The `production-release` artifact (14 days) and `docs/deployment_runbook.md`

### M4 · Data operations
- `scripts/seed.sh`, `bulk-load.sh`, `backup.sh`, `restore-plan.sh`, and the `data_ops.py` helpers
- `data/seed/` and `data/bulk/` synthetic CSVs, and `backup-queries/` (3 SOQL files)
- Backup catalog `backups/YYYYMMDD/backup-manifest.json` (gitignored) and `docs/data_operations.md`

### M5 · Release tool comparison
- `docs/release-comparison.md`, `tool-selection-guide.md`, `lifecycle-center-notes.md`, and `flosum-notes.md`, all marked **Studied**
- No change to the M1–M4 workflows or scripts

### M6 · Advisory deploy-failure triage
- `triage/triage.py`, `triage/rules.yml`, and `triage/eval/` (15 fixtures + manifest)
- `docs/ai_triage.md` and `docs/ai_triage_evaluation.md`
- No new workflow, no model client, no network calls

### Portfolio documentation (this release)
- `README.md`, plus [PROJECT_OVERVIEW](PROJECT_OVERVIEW.md), [ARCHITECTURE](ARCHITECTURE.md), [TECHNICAL_DESIGN](TECHNICAL_DESIGN.md), [CASE_STUDY](CASE_STUDY.md), [LESSONS_LEARNED](LESSONS_LEARNED.md), [ROADMAP](ROADMAP.md), [DEMO_SCRIPT](DEMO_SCRIPT.md), [INTERVIEW_GUIDE](INTERVIEW_GUIDE.md), [SUMMARY_FOR_RECRUITERS](SUMMARY_FOR_RECRUITERS.md), [SUMMARY_FOR_HIRING_MANAGERS](SUMMARY_FOR_HIRING_MANAGERS.md), [RESUME_BULLETS](RESUME_BULLETS.md), and [LINKEDIN_POST](LINKEDIN_POST.md)
- No source code, workflow, script, metadata, or test was changed for these documents. The README keeps every heading and link the repository's tests require.

## Verification

| Check | Result |
|---|---|
| `python -m unittest tests.test_triage tests.test_coverage_gate tests.test_deployment tests.test_data_operations` | **83 tests, OK** (23 + 11 + 23 + 26) |
| `node --test tests/m1-foundation.test.js` | **8 tests.** All 8 pass on Windows. See the known issues below for macOS and Linux. |
| `python triage/triage.py --eval` | 15 evaluated · 15 correct · 0 incorrect · 2 unknown · accuracy 1.0 |
| Coverage gate on the 7 fixtures | Exit 0 for pass and exact 85% · exit 1 for low, missing, empty, test failure · exit 2 for malformed |

None of these checks calls Salesforce.

## Not included in v1.0

- A run of any workflow against Org A or Org B
- A scratch org, an Apex test executed in an org, a quick-deploy attempt, or a release tag
- A seed, bulk load, backup, or restore against an org
- A language-model call, a posted PR comment, or an automated merge or approval
- A Lifecycle Center project, a Flosum tenant, an Enterprise org, a sandbox, or a production org

## Upgrade and setup notes

- Add the repository secrets `STAGING_AUTH_URL` (Org B) and `PROD_AUTH_URL` (Org A), each an `sfdxAuthUrl` from `sf org display --verbose --json`. Never commit that output.
- Open pull requests from a branch in this repository. Fork PRs don't receive secrets and fail at Org Login.
- Keep Org B free of unrelated local Apex tests. `RunLocalTests` runs all of them, and the org can't be refreshed.

## Known issues and limitations

| Item | Impact | Workaround |
|---|---|---|
| Node foundation test 4 calls `powershell` to parse XML | Fails with `spawnSync powershell ENOENT` on macOS and Linux | Run the Node suite on Windows. A portable rewrite is in the roadmap (2.1). |
| `docs/deployment_runbook.md` says "There is no data script in this milestone." | Stale since M4 | Data scripts are documented in `docs/data_operations.md` |
| Quick deploy on Org A | Expected to be refused, because Developer Edition isn't a production org | Handled. A normal deploy runs and the reason is recorded. |
| Org A is both the Dev Hub and the production stand-in | Releases aren't isolated from scratch-org administration | Documented trade-off. Roadmap 3.3. |
| Production approval is a typed word | Doesn't restrict *who* approves | Roadmap 2.4 (GitHub Environments) |
| A second `restore-plan.sh --apply` duplicates rows | Duplicate checklist rows | Delete the extras or refresh the scratch org |

## Compatibility

| Component | Version |
|---|---|
| Salesforce source API | 68.0 |
| Salesforce CLI | `@salesforce/cli@latest` in CI |
| Code Analyzer | `code-analyzer@latest` plugin, `Recommended` rules |
| Java (CI) | Temurin 17 |
| Node.js | 20+ (CI uses 20) |
| Python | 3.11+, standard library only |
| Orgs | Developer Edition (Org A, Org B) and a Developer scratch org |
