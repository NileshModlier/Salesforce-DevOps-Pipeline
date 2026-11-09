# Summary for Hiring Managers

**Salesforce DevOps Pipeline**: a release-engineering and technical-operations portfolio built for Developer Edition orgs.

## The short version

I built a complete Salesforce DX release path: a gated pull request, a staging deploy, a manually approved production stand-in release with quick deploy and a recorded fallback, release tags, and tag-based rollback. Around it sit operational data tooling and an advisory failure-triage engine. The code, workflows, and 19 runbooks are in the repository and validated by 83 Python tests and 8 Node tests that run without Salesforce. It hasn't yet been run against an org. The two GitHub secrets and one run of each workflow are the remaining step. All orgs are Developer Edition. There's no sandbox, Enterprise org, or production tenant.

## What it shows about how I work

| Signal | Evidence in the repository |
|---|---|
| **Safety by default** | Gates fail closed: the coverage gate exits 2 on malformed input, the job-ID parser refuses failed validations, rollback refuses unknown tags, backups refuse to overwrite, and restore refuses to apply without a target |
| **Precise error handling** | The quick-deploy fallback triggers only on 9 recognised edition refusals. Expired jobs and auth errors stop the release instead of deploying anyway. |
| **Separation of concerns** | Metadata rollback and data restore are separate scripts with separate promises. Triage can read but can't act. |
| **Operational empathy** | Runbooks give expected output, negative tests, and recovery steps (R1–R8, plus a 9-row failure matrix for releases) |
| **Security hygiene** | Auth URLs only in secrets, `umask 077` temp files, least-privilege workflow permissions, and gitignored auth state, reports, and backups |
| **Testability** | Pure-function scripts with exit-code contracts, 7 coverage-gate fixtures, and 15 triage fixtures including precedence traps and true negatives |
| **Honesty about evidence** | Every capability is labelled Implemented, Studied, or Not available, with a written list of what wasn't executed |

## Technical highlights

1. **PR gate.** Code Analyzer (`Recommended`, severity ≤ 2 fails) → check-only `sf project deploy validate` with `RunLocalTests` on Org B → a Python gate at **85% per class and per trigger**, with `Decimal` half-up rounding, missing-row detection, and a markdown summary. It runs even after a failed validation, so the PR artifact always explains the failure.
2. **Production stand-in release.** `workflow_dispatch` plus a typed `deploy` → validate → coverage gate → 0Af job ID → quick deploy → classified outcome (`succeeded`, `unsupported`, or `failed`) written to `quick-deploy.json` → 5 smoke checks → an annotated UTC tag that's never moved → a 14-day artifact.
3. **Rollback.** `git archive` of a release tag (the working tree is untouched) → redeploy → that tag's own destructive manifest, applied only if it lists members → smoke. It prints "metadata-only rollback; org records are not restored".
4. **Data operations.** A seed with license detection and alias-scoped synthetic usernames. A bulk load with a deliberate failing row, so the error path is always exercised. Dated backups from 3 SOQL queries with a row-count manifest. A restore that strips IDs, owners, audit, and relationship columns and never inserts users.
5. **Triage.** 8 priority-ordered categories (auth > permission > analyzer > Apex test > coverage > conflict > missing component > unknown). It runs on the standard library only, redacts and refuses deploy and merge commands, and prints "Human review required." on every report. It scored 15/15 on its fixtures.

## Trade-offs I made deliberately

| Decision | Trade-off accepted |
|---|---|
| Org A is both the Dev Hub and the production stand-in | Releases aren't isolated from scratch-org administration. Documented, and there's a roadmap item for a third org. |
| Check-only validation on Org B, not a scratch org per PR | Uses no scratch quota (3 active, 6 per day). The cost is depending on a long-lived org that can't be refreshed. |
| A typed approval word, not GitHub Environments | Works on any plan, but doesn't restrict *who* approves. Environments are on the roadmap. |
| Rules plus templates, not an LLM | Deterministic, auditable, offline, and free of data egress, at the cost of less fluent prose |
| `@latest` CLI and Code Analyzer in CI | Always current, but less reproducible. Pinning is on the roadmap. |

## Gaps I know about

- **No live run yet.** No workflow has executed against Org A or Org B, and no tag, backup, or restore exists.
- **One Windows-only test.** Node foundation test 4 calls `powershell` for XML parsing and fails on macOS and Linux runners. The XML itself is well formed. A stand-in parser makes all 8 tests pass.
- **One stale doc line.** The deployment runbook still says there's no data script, which predates M4.
- **Studied, not used:** Lifecycle Center, which needs a sandbox, and Flosum, which needs a tenant.

## How to evaluate it in 15 minutes

```bash
python -m unittest tests.test_triage tests.test_coverage_gate tests.test_deployment tests.test_data_operations
python scripts/coverage_gate.py tests/fixtures/validate-low-coverage.json --summary reports/coverage-summary.md
python scripts/quick_deploy_decision.py assess --exit-code 1 --stderr-file <(echo "INVALID_SESSION_ID")
bash scripts/rollback.sh --dry-run
python triage/triage.py --eval
```

Then read `.github/workflows/deploy-prod.yml` and `docs/deployment_runbook.md` (the Failure Recovery section). Interview prompts are in [INTERVIEW_GUIDE.md](INTERVIEW_GUIDE.md), and a 5-minute walkthrough is in [DEMO_SCRIPT.md](DEMO_SCRIPT.md).

## Suitable for

Salesforce technical operations · Salesforce DevOps engineering · Release engineering · Platform engineering · Salesforce application support
