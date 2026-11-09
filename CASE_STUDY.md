# Case Study: A Salesforce Release Pipeline Built for Developer Edition

**How one engineer designed gated validation, controlled releases, rollback, data operations, and advisory failure triage for Salesforce, using only free Developer Edition orgs and documenting exactly where the evidence stops.**

| | |
|---|---|
| **Role** | Sole designer and implementer |
| **Domain** | Salesforce technical operations and release engineering |
| **Stack** | Salesforce DX (API 68.0), Salesforce CLI, Code Analyzer, GitHub Actions, Apex, Flow, LWC, Python 3.11+, Bash, Node.js 20+ |
| **Environment** | Developer Edition Org A (Dev Hub and production stand-in), Developer Edition Org B (staging stand-in), and a Developer scratch org |
| **Outcome** | 6 milestones delivered as repository artifacts. 83 Python tests and 8 Node tests passing locally. Triage classifies 15 of 15 fixtures correctly. Not yet run against an org. |

---

## Background

Salesforce release work sits between administration and software engineering. Changes arrive both as code and as clicks in Setup. Orgs have hard platform limits. The usual enterprise answers, sandboxes, a production org, and a commercial release tool, come with a license.

This project asked a practical question: **how much of a professional Salesforce release discipline can be built and demonstrated with nothing but Developer Edition orgs, and how do you present it without overstating what was proven?**

The scope was set across six milestones:

| Milestone | Goal |
|---|---|
| M1 | A DX project, Dev Hub, scratch org workflow, source tracking, and Apex tests, all with runbooks |
| M2 | A pull-request gate: static analysis, check-only validation, and a coverage threshold |
| M3 | A release path: staging deploy, manual production stand-in release, quick deploy, tags, and rollback |
| M4 | Operational data handling: seeding, bulk loads, backups, and a restore plan |
| M5 | A written comparison of the CLI path with Salesforce Lifecycle Center and Flosum |
| M6 | Advisory triage of failed validations, with a mandatory human review gate |

The deployable payload was kept deliberately small: a **Release Checklist Item** app with one object, four fields, an Apex service and trigger, a test class, a before-save flow, an LWC, and a permission set. That gives the pipeline every common metadata type to move, without turning the project into an application build.

## Challenges

| Challenge | Why it was hard |
|---|---|
| **No sandboxes** | Developer Edition orgs can't create them, so there's no conventional dev → UAT → prod sandbox chain and no refresh |
| **No production org** | Quick deploy, the 75% coverage enforcement, and production approvals all assume a production org |
| **Only one tracked environment** | Source tracking works on scratch orgs, not on Developer Edition orgs |
| **Tight quotas** | 3 active and 6 daily scratch orgs, 15,000 API requests per day, and 5 MB of data storage per long-lived org |
| **Two orgs, three roles** | Org A has to be both the Dev Hub and the production stand-in |
| **Unrefreshable staging** | `RunLocalTests` on Org B runs every local test in the org, forever |
| **Data versus metadata recovery** | Redeploying a tag doesn't bring back records, and restoring records across orgs breaks IDs, owners, and users |
| **Tool comparison without tools** | Lifecycle Center's documented path needs a sandbox, and Flosum needs a tenant. Neither was available. |
| **"AI" without hype** | Failure triage had to be useful without a model call and without any ability to act |
| **Credibility** | The authoring machine had no Salesforce CLI, so nothing could be run against an org from it |

## Design decisions

1. **Name the stand-ins honestly.** Org B is the *staging stand-in*. Org A is the *production stand-in* and is also the Dev Hub. Every document repeats this, and the trade-off of releasing into the Dev Hub org is written down rather than hidden.
2. **Make the scratch org the only tracked environment.** Development happens there. The result is retrieved and committed to Git, which is the only thing that survives the 7-day org.
3. **Validate on PRs without spending scratch quota.** A check-only `sf project deploy validate` against Org B proves deployability without saving metadata and without using one of the 6 daily scratch creates.
4. **Enforce coverage in the repository, not the platform.** Developer Edition doesn't apply the 75% rule, so `scripts/coverage_gate.py` enforces **85% per class and per trigger** and fails closed on malformed or incomplete input.
5. **Keep quick deploy, and handle its refusal precisely.** The production stand-in path validates, captures the `0Af…` job ID, and tries quick deploy. Only a recognised "production only" refusal (9 phrases) triggers a normal deploy. That decision is recorded in `quick-deploy.json`. Every other error stops the release.
6. **Tag only what passed smoke.** `release-YYYYMMDD-HHMM` in UTC is created after five smoke checks pass, and it's never moved, so every tag is a safe rollback target.
7. **Separate metadata rollback from data restore.** Rollback redeploys a tag's `force-app/` and says plainly that it doesn't restore records. Data restore is its own script, inserts checklist rows only, and never inserts users.
8. **Make triage advisory by construction.** Rules plus templates, standard library only, no network. It refuses to emit deploy or merge commands. Every output starts with "Human review required."
9. **Label every claim.** Each capability is Implemented, Studied, or Not available, and the docs carry a "What was not executed" list.

## Release engineering

```text
PR ──► Code Analyzer (sev 1–2 fail) ──► check-only validate on Org B (RunLocalTests) ──► 85% coverage gate
merge ──► deploy start to Org B ──► smoke (5 checks)
manual, type "deploy" ──► validate Org A ──► coverage gate ──► 0Af job id ──► quick deploy
                                                     └─ refused by edition? deploy start + record reason
                       ──► smoke ──► release-YYYYMMDD-HHMM ──► artifact (14 days)
undo ──► rollback.sh: git archive tag ──► redeploy ──► optional destructive manifest ──► smoke
```

What makes this release engineering rather than scripting:

- **Gates fail closed.** A missing coverage row, empty coverage, a malformed file, or a failed validation status all fail the gate. The job-ID parser refuses anything but a Succeeded validation.
- **Recorded decisions.** The production artifact carries the validation JSON, the coverage summary, the job ID, the quick-deploy record, and any fallback output.
- **Non-destructive defaults.** The committed destructive manifest is empty, rollback uses `git archive` rather than checkout, and tags can't be overwritten.
- **Concurrency control.** Both deploy workflows queue rather than cancel an in-flight deploy.
- **Secret handling.** Auth URLs live in GitHub secrets, pass through a `umask 077` temp file, and are deleted immediately after login.
- **Runbooks with negative tests.** For failed validation, failed smoke, an unsupported quick deploy, and rollback failures, the runbook gives the expected output and the operator action.

## Data operations

Developer Edition storage (5 MB of data) and licenses (typically 2) shaped every choice.

| Operation | Design |
|---|---|
| Seed | 3 synthetic `Seed:` rows to the scratch org, verified by a `COUNT()` query (≥ 3). The synthetic user is created only if a Salesforce license is free, with its username rewritten per alias to avoid global collisions. |
| Bulk load | 3 rows, 1 deliberately invalid (`NotAStatus` against a restricted picklist), so the error-file path is always exercised. `--preflight` produces the success and error CSVs locally. |
| Backup | 3 committed SOQL queries → CSV and JSON → `backup-manifest.json` with row counts, in a dated UTC folder that's never overwritten without `--force`, and gitignored because it contains usernames |
| Restore | A plan by default. `--apply` needs an explicit backup and target, strips `Id`, `OwnerId`, and audit columns, re-inserts checklist rows only, and never restores users. Re-applying duplicates rows, which is documented as a risk. |
| Nightly routine | An operator checklist, not a workflow: confirm the alias, back up Org B, check the manifest, keep 7 folders, never commit |

## AI triage

The question was what an "AI-assisted" step should do in a release pipeline when it isn't allowed to act.

The answer was a two-stage advisory reader:

1. **Rules.** Eight categories in a fixed priority. Authentication and permission errors outrank test noise, because those deploys never reached the tests. A test failure outranks low coverage on the same run. Code Analyzer severity 3 and above is deliberately ignored, matching the PR gate.
2. **Template.** Each category has a fixed explanation and next step. A "model configured" flag is accepted and ignored, so the text stays rule-based.

Guardrails: no network, no Salesforce, no GitHub, and no command suggestions to deploy, quick-deploy, or merge, which are redacted and then checked again. Confidence is a fixed value per category: 0.88 to 0.97 for known categories and 0.34 for Unknown. It isn't approval.

Evaluation used 15 committed fixtures, including precedence traps and two true negatives. **Result: 15 of 15 correct, 2 expected Unknown, accuracy 1.0.** These are fixtures written for the project. They aren't production failures, and they aren't evidence of a model's accuracy.

## Results

**Delivered (in the repository):**

| Deliverable | Count |
|---|---|
| GitHub Actions workflows | 3 |
| Operational scripts | 12 (7 Bash, 5 Python) |
| Sample metadata components | 7 members across 6 metadata types, plus 4 fields |
| Apex test methods | 7, including a 200-record bulk test |
| Runbooks and reference docs | 19 (~25,500 words) |
| Triage categories and fixtures | 8 and 15 |
| Backup queries | 3 |

**Verified locally (no Salesforce calls):**

| Check | Result |
|---|---|
| Python test suite | 83 tests, OK (23 triage, 11 coverage gate, 23 deployment, 26 data operations) |
| Node foundation tests | 8 tests. They pass on Windows. Test 4 needs `powershell`. |
| Coverage gate on 7 fixtures | Correct exit codes: 0 for pass and exact 85%, 1 for four failure types, 2 for malformed |
| Triage evaluation | 15/15, accuracy 1.0 |
| Dry runs | Rollback, production deploy, seed, bulk-load preflight, backup, and restore plan all print the intended plan or refuse unsafe input |

**Not yet demonstrated (documented as such):** a green GitHub Actions run, an org login, Apex executing in an org, a quick-deploy refusal from Org A, a release tag, a backup from Org B, and any Lifecycle Center or Flosum operation.

## Lessons learned

1. **Developer Edition is enough to show discipline, not to prove scale.** Quotas and missing sandboxes force a smaller topology. The controls (gates, records, tags, rollback) carry over. Production throughput claims don't.
2. **A precise fallback beats a generous one.** Falling back to a normal deploy on *any* quick-deploy error would hide expired job IDs and auth failures. Matching the specific refusal keeps failures visible.
3. **Rollback language matters.** Saying "metadata only" in the script output prevents the most expensive misunderstanding in Salesforce recovery.
4. **Restoring users is a trap.** Licenses, global username uniqueness, and missing credentials make user restoration the wrong default.
5. **Advisory AI needs structural limits, not just a disclaimer.** Redaction, refusal, and the absence of any write path do more than a warning label.
6. **Evidence labels build trust.** Writing down what *wasn't* executed makes everything else credible.
7. **Portability is a test requirement too.** One foundation test relies on Windows PowerShell, which a Linux CI runner would surface immediately.

The full retrospective is in [LESSONS_LEARNED.md](LESSONS_LEARNED.md).
