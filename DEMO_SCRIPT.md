# 5-Minute Interview Demo Script

A timed walkthrough of the Salesforce DevOps Pipeline that runs entirely on a laptop, with no Salesforce org, no secrets, and no network. Every command and output below was run against this repository.

> **Say this up front:** "Everything I'll run is local. The workflows and scripts are built for Developer Edition orgs, but I haven't run them against an org yet. I'll show the gates and safety logic directly, and walk through the org steps from the runbooks."

---

## Before the interview (5 minutes of prep)

```bash
cd sf-devops-pipeline
python -m unittest tests.test_triage tests.test_coverage_gate tests.test_deployment tests.test_data_operations   # expect: Ran 83 tests … OK
python triage/triage.py --eval | head -6                                                                          # expect: 15 / 15 / 0 / 2 / 1.0
```

Open these in editor tabs, in this order:

1. `README.md` (Architecture section)
2. `config/project-scratch-def.json`
3. `.github/workflows/pr-validate.yml`
4. `.github/workflows/deploy-prod.yml`
5. `docs/deployment_runbook.md` (Quick Deploy section)

Use Git Bash or WSL on Windows, because the commands use Bash process substitution.

---

## 0:00 – 0:30 · Frame the problem

**Show:** the README Architecture diagram and org table.

**Say:**
> "Salesforce releases fail in predictable ways: manual moves, skipped tests, no known-good point to roll back to. I built one release path with gates, a manual production approval, tags, rollback, data operations, and a triage tool, using only Developer Edition orgs. Org B is my staging stand-in. Org A is both my Dev Hub and my production stand-in, and I'll be upfront that that's a trade-off."

## 0:30 – 1:15 · Scratch org workflow

**Show:** `config/project-scratch-def.json`, then the six steps in `runbooks/starting-new-development.md`.

**Say:**
> "Developer Edition orgs can't be source-tracked, so the scratch org is my only tracked environment. Before creating one I check `sf limits api display`. This Dev Hub allows 3 active and 6 creates per rolling 24 hours, and deleting an org gives back the active slot but not the daily one. I deploy `force-app`, make a Setup change, preview and retrieve it, run `RunLocalTests` (7 test methods, including a 200-record bulk insert), commit, and delete the org."

**Point at:** `sf org create scratch … --duration-days 7` and the recovery procedures R1–R8 in `docs/RUNBOOK.md`.

## 1:15 – 2:00 · PR validation and coverage gates

**Show:** `pr-validate.yml`. Name the order: Code Analyzer → login → check-only validate → coverage gate → artifact.

**Run:**

```bash
python scripts/coverage_gate.py tests/fixtures/validate-low-coverage.json --summary reports/coverage-summary.md; echo "exit=$?"
```

```text
Coverage gate failed.
Class coverage for ReleaseChecklistItemService is 80.00% (threshold 85%).
Trigger coverage for ReleaseChecklistItemTrigger is 25.00% (threshold 85%).
exit=1
```

```bash
python scripts/coverage_gate.py tests/fixtures/validate-exact-threshold.json --summary reports/coverage-summary.md; echo "exit=$?"
python scripts/coverage_gate.py tests/fixtures/validate-malformed.json --summary reports/coverage-summary.md 2>/dev/null; echo "exit=$?"
```

```text
Coverage gate passed. Summary: reports/coverage-summary.md
exit=0
exit=2
```

**Say:**
> "Developer Edition doesn't enforce the 75% production rule, so a weak validation would still pass on Org B. The repository enforces 85% per class and per trigger, rounded half-up, so 85.00 passes and 84.99 fails. It fails closed: malformed JSON is exit 2, and missing or empty coverage is a failure. Code Analyzer runs first, because it needs no org."

## 2:00 – 2:50 · Deployment process

**Show:** `deploy-prod.yml`. Point at `workflow_dispatch`, `confirm`, and the `"deploy"` check, then the step list: validate → gate → job ID → quick deploy → smoke → tag.

**Run:**

```bash
python scripts/parse_job_id.py tests/fixtures/validate-test-failure.json; echo "exit=$?"
```

```text
Validation did not succeed (status=Failed). Quick deploy was not started.
exit=1
```

```bash
python scripts/quick_deploy_decision.py assess --exit-code 1 --stderr-file <(echo "Quick deployments are only supported in production orgs.")
python scripts/quick_deploy_decision.py assess --exit-code 1 --stderr-file <(echo "INVALID_SESSION_ID: Session expired or invalid")
```

```text
unsupported
failed
```

**Say:**
> "Quick deploy is a production-org feature, and Org A is Developer Edition, so I expect a refusal. The decision script falls back to a normal deploy only when the CLI message matches one of nine known 'production only' refusals. An expired session is `failed`, and the release stops instead of deploying anyway. Either way, `quick-deploy.json` records the outcome and the reason. After five smoke checks pass, the job creates a UTC `release-YYYYMMDD-HHMM` tag and refuses to move one that already exists."

```bash
bash scripts/smoke_check.sh --list
```

```text
Custom object Release_Checklist_Item__c
Permission set Release_Checklist_Access
LWC checklistItemBadge
Flow Stamp_Ready_For_Review
Apex tests RunLocalTests
```

## 2:50 – 3:30 · Rollback

**Run:**

```bash
bash scripts/rollback.sh --dry-run; echo "exit=$?"
bash scripts/rollback.sh --dry-run --tag v1.0; echo "exit=$?"
```

```text
No release tag found. Refusing to roll back without release-YYYYMMDD-HHMM.
exit=1
Tag v1.0 does not match release-YYYYMMDD-HHMM.
exit=2
```

**Say:**
> "No release has been tagged yet, so rollback refuses. It won't guess a target. With a release tag, it runs `git archive` on that tag, so my working tree is untouched. It redeploys that tag's `force-app`, applies the destructive manifest only if the tag's own copy lists members, and smoke-checks. Before it deploys it prints 'metadata-only rollback; org records are not restored'. That's the most common misunderstanding in Salesforce recovery."

*Optional, if there's time and you've prepared a throwaway clone with a demo tag:* the dry run prints `rollback-tag`, `checkout: git archive …`, `deploy: sf project deploy start …`, `data: metadata-only rollback; org records are not restored`, `destructive: skip`, and `smoke: scripts/smoke_check.sh prod`. Don't create demo tags in the real repository.

## 3:30 – 4:10 · Backup plans

**Run:**

```bash
bash scripts/backup.sh --dry-run --date 20261109 | head -4
bash scripts/restore-plan.sh | sed -n '4,8p'
bash scripts/restore-plan.sh --apply; echo "exit=$?"
```

```text
backup-date: 20261109
backup-dir: backups/20261109
target-org: staging
query: backup-queries/Release_Checklist_Item.soql
## Restore Order
1. Confirm Release_Checklist_Item__c and permission set Release_Checklist_Access are already deployed.
2. Users: do not insert backup users. Match Assignee.Username to users already in the org.
3. PermissionSetAssignment: recreate Release_Checklist_Access only for a user that already exists.
4. Release_Checklist_Item__c: insert business fields from the backup CSV. Id and OwnerId are removed.
Apply requires --backup and --target-org.
exit=2
```

**Say:**
> "Backups are three committed SOQL queries exported as CSV and JSON into a dated UTC folder, with a manifest of row counts. They're gitignored because the user export has usernames. Restore is a plan by default. Apply needs an explicit backup and target, strips IDs and owners, and never inserts users. Developer Edition has about two licenses, and usernames are globally unique."

**Optional (5 seconds):** `bash scripts/bulk-load.sh --preflight` shows `errors: 1`. That's the deliberate `NotAStatus` row, which proves the error-file path works.

## 4:10 – 4:50 · AI failure triage

**Run:**

```bash
python triage/triage.py triage/eval/coverage-82-tests-pass.json | sed -n '1,9p'
```

```text
# Deploy-failure triage

Human review required.

- Failure Category: Coverage Failure
- Confidence: 0.92 (high)
- Likely Cause: Apex tests passed, and coverage for ReleaseChecklistItemService fell below 85% (reported 82.00%).
- Suggested Next Step: Read the coverage rows for ReleaseChecklistItemService, add or extend an Apex test that executes the uncovered lines, and re-run check-only validation locally.
- Reference Artifact: triage/eval/coverage-82-tests-pass.json#result.details.runTestResult.codeCoverage[ReleaseChecklistItemService]
```

```bash
python triage/triage.py --eval | head -6
```

```text
{
  "cases_evaluated": 15,
  "correct_classifications": 15,
  "incorrect_classifications": 0,
  "unknown_classifications": 2,
  "accuracy": 1.0,
```

**Say:**
> "Stage 1 is a rules file with eight categories in priority order. An auth failure outranks test noise, and a test failure outranks low coverage. Stage 2 is a local template. There's no model call and no network. It can't deploy, merge, or post. It even strips deploy and merge commands from echoed errors and refuses the report if any remain. Every report starts with 'Human review required.' It gets 15 of 15 on my fixtures, two of which are correctly Unknown. Those are my fixtures, not production data."

## 4:50 – 5:00 · Close

**Say:**
> "What's next is evidence, not features: add the two auth-URL secrets and run each workflow once against Org A and Org B, then replace my 'not executed' notes with the artifacts."

---

## If something goes wrong live

| Symptom | Recovery line |
|---|---|
| `<(echo …)` errors on Windows | "Process substitution needs Bash. I'll use Git Bash." Or describe the two outcomes from `docs/deployment_runbook.md`. |
| `python` isn't found | Use `python3`, or `py -3` on Windows |
| The Node suite shows 1 failure on macOS or Linux | "Test 4 shells out to Windows PowerShell for XML parsing. It's a known portability issue on my roadmap. The other 7 pass." |
| Running out of time | Skip rollback and backup output. Say the one-line summary of each, and finish with triage. |

## Likely follow-up questions

- *Why not a scratch org per PR?* That costs 1 of 6 daily creates per PR. A check-only validation on Org B costs none.
- *Why 85% and not 75%?* Developer Edition doesn't enforce 75% here, and a per-row 85% catches a weak class hidden in a good average.
- *What would you change for a real production org?* GitHub Environments with required reviewers, pinned CLI versions, a dedicated Dev Hub, and quick deploy would then actually be supported.
- *Is the triage really "AI"?* It's AI-assisted triage, not an AI agent. It's rules plus templates by design, so it can be tested and audited. A model could rephrase Stage 2 later, behind the same human gate.

More questions with full answers are in [INTERVIEW_GUIDE.md](INTERVIEW_GUIDE.md).
