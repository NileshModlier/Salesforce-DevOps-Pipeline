# Interview Guide: 50 Questions with Detailed Answers

These questions are ones an interviewer is likely to ask about this project. Each answer combines the general Salesforce concept with exactly what this repository does. The answers stay inside the evidence boundary: the workflows and scripts exist and are tested locally, but they haven't been run against an org yet.

**How to use it:** Read the **bold** first sentence of each answer as the short version. The rest is the follow-up if they dig in.

| Topic | Questions |
|---|---|
| [Project framing](#project-framing) | 1–2 |
| [Salesforce DX](#salesforce-dx) | 3–6 |
| [Scratch orgs](#scratch-orgs) | 7–10 |
| [Source tracking](#source-tracking) | 11–14 |
| [Apex testing](#apex-testing) | 15–19 |
| [Code Analyzer](#code-analyzer) | 20–23 |
| [Validation deployments](#validation-deployments) | 24–30 |
| [Rollback](#rollback) | 31–34 |
| [Data operations](#data-operations) | 35–39 |
| [Lifecycle Center](#lifecycle-center) | 40–42 |
| [Flosum](#flosum) | 43–45 |
| [AI failure analysis](#ai-failure-analysis) | 46–50 |

---

## Project framing

### 1. Walk me through this project in two minutes.
**It's one controlled release path for Salesforce metadata, built for Developer Edition orgs, with the operational work around it.** A developer works in a source-tracked scratch org and commits to Git. A pull request runs Code Analyzer, a check-only validation against Org B (my staging stand-in), and an 85% coverage gate. A merge deploys to Org B and smoke-checks it. A production stand-in release to Org A runs only when an operator types `deploy`. It validates, tries quick deploy, records a fallback if Developer Edition refuses, smoke-checks, and creates a UTC `release-YYYYMMDD-HHMM` tag. Rollback redeploys a tag. Alongside the release path there are scripts for synthetic data, backups, and a restore plan, and a rules-based triage tool that reads failed reports and drafts advice for a person. It's six milestones, with 83 Python tests and 8 Node tests passing locally.

### 2. What can't you claim about this project?
**I can't claim any of it has run against a Salesforce org yet, and I can't claim enterprise environments.** The machine I built it on didn't have the Salesforce CLI, so no scratch org, deploy, validation, tag, backup, or restore has happened. The orgs in the design are Developer Edition. There's no Enterprise org, no sandbox, no production tenant, and no Flosum tenant. Lifecycle Center and Flosum are studied from documentation, not operated. The triage tool doesn't call a language model. Org A is also both the Dev Hub and the production stand-in, which means releases aren't isolated from scratch-org administration. I document that trade-off rather than hide it. The next step is adding the two auth-URL secrets and running each workflow once.

## Salesforce DX

### 3. What is Salesforce DX, and why use source format?
**Salesforce DX is the source-driven development model: Git as the source of truth, the `sf` CLI, scratch orgs, and source format.** Source format decomposes metadata into small files, for example each custom field in its own `.field-meta.xml` under the object folder. That makes diffs and merges reviewable. The older Metadata API format puts a whole object in one file. Here, `force-app/main/default/` holds the object, four fields, two classes, a trigger, a flow, an LWC, and a permission set in source format. The CLI deploys source format directly, so no conversion step is needed in the pipeline.

### 4. What's in your `sfdx-project.json`, and why?
**One default package directory (`force-app`), source API version 68.0, no namespace, and `login.salesforce.com` as the login host.** Pinning `sourceApiVersion` keeps metadata behaviour consistent across orgs, and every `-meta.xml` and `package.xml` uses 68.0, which a Node test checks. No namespace keeps it simple because I'm not building a managed package. `login.salesforce.com` is the Developer Edition login host. It's also the production host, but I document that it isn't evidence of production access here. `test.salesforce.com`, the sandbox host, isn't used because there are no sandboxes.

### 5. Why didn't you use unlocked or second-generation packages?
**Packaging would add versioning and dependency machinery that doesn't serve this project's goal, and it isn't in scope.** The goal is a release path for org-based metadata: validate, deploy, tag, roll back. Unlocked packages would change the release unit to a package version and the rollback model to installing an older version, which is a different design. With one small app and no namespace, source-format deploys of `force-app` are the clearer demonstration. If the app grew into modules with separate release cadences, unlocked packages would be the next thing to evaluate.

### 6. What do `.forceignore` and `manifest/package.xml` do here?
**`.forceignore` keeps non-metadata out of deploys, and `package.xml` names the exact components for manifest-based operations, mainly destructive rollback.** `.forceignore` excludes `docs/**`, `runbooks/**`, `tests/**`, `package.json`, and Node tooling, so a broad deploy never tries to treat them as metadata. `package.xml` lists 7 members across 6 types (ApexClass, ApexTrigger, CustomObject, Flow, LightningComponentBundle, PermissionSet). Rollback uses it together with `destructiveChangesPost.xml` when a release needs to delete components. Normal deploys use `--source-dir force-app`.

## Scratch orgs

### 7. What is a scratch org, and why is it central here?
**A scratch org is a short-lived, configurable org created from a Dev Hub, and it's the only source-tracked environment this project has.** Developer Edition orgs can't be source-tracked and can't create sandboxes, so the scratch org is where development and change capture happen. Mine is created from `config/project-scratch-def.json` for 7 days, gets the full `force-app` deploy and the permission set, and is deleted when the work is committed. Git is what survives. The scratch org is a working copy with a clock on it.

### 8. What's in your scratch org definition?
**Edition `Developer`, no sample data, Lightning Experience on, persistent browser caching off, and the `EnableSetPasswordInApi` feature.** The edition matches the Dev Hub's edition and allocation, so no paid features are assumed. `hasSampleData: false` means tests see only their own rows. `EnableSetPasswordInApi` lets an operator set a scratch user's password through the API if needed. A Node test verifies the definition parses and matches what `docs/scratch_orgs.md` documents.

### 9. What scratch org limits did you design around?
**A Developer Edition Dev Hub allows 3 active scratch orgs and 6 successful creates per rolling 24 hours.** Deleting an org frees an active slot but doesn't give back the daily create, and the window slides rather than resetting at midnight. So the runbook checks `sf limits api display --target-org devhub-org-a` before creating. Pull-request validation deliberately doesn't create scratch orgs, because a check-only deploy to Org B costs no scratch quota. The runbook's recovery steps R2 and R3 cover freeing a slot and waiting out the daily window.

### 10. How do scratch orgs, sandboxes, and Developer Edition orgs differ?
**Scratch orgs are disposable and source-tracked. Sandboxes are copies of a production org. Developer Edition orgs are free, long-lived, standalone orgs.** Scratch orgs last 1 to 30 days and are built from a definition file. Sandboxes need a production parent org and can be refreshed, and Developer and Developer Pro sandboxes can be source-tracked. Developer Edition orgs have no sandbox capability, no source tracking, and small limits: 5 MB of data, 20 MB of files, and 15,000 API requests per day. This project uses one Developer Edition org as a Dev Hub and production stand-in, another as a staging stand-in, and scratch orgs for development.

## Source tracking

### 11. How does source tracking work?
**The CLI records which metadata revisions it has already synced with a tracked org, so later deploys and retrieves only exchange what changed.** After the first full deploy, `sf project retrieve preview` shows Setup changes made in the org, and `sf project retrieve start` writes them into `force-app/`. The runbook demonstrates this by adding a checkbox field `Needs_Follow_Up__c` in Setup. Preview lists it, retrieve writes the `.field-meta.xml`, and a second preview is empty. Then it's committed to Git.

### 12. Why can't you use source tracking on Org A and Org B?
**Salesforce supports source tracking on scratch orgs and on Developer and Developer Pro sandboxes, not on Developer Edition orgs.** `sf org enable tracking` fails against them. You can still deploy to them by naming the source explicitly with `--source-dir force-app` or `--manifest`, which is what the workflows do. You just don't get a tracking diff. That's why every retrieve in the runbook targets `m1-scratch` only.

### 13. How do you handle a source tracking conflict?
**Preview both directions first, compare, decide which side wins, and only then use `--ignore-conflicts` in that direction.** A conflict means the same component changed both locally and in the org since the last sync. The runbook's recovery step R5 is to run `sf project retrieve preview` and `sf project deploy preview`, compare the local file with the org version, and then retrieve with `--ignore-conflicts` if the org wins or deploy with it if Git wins. If unsure, copy the local file aside, retrieve, and diff. Don't delete `.sf/` as a first response, because that drops the baseline and makes everything look new.

### 14. Where is tracking state stored, and what must never be committed?
**Tracking and auth state live in `.sf/` and `.sfdx/`, which are gitignored along with test output, reports, and backups.** That state is machine-local. Copying the project to another computer doesn't carry a usable baseline, so the new machine creates its own scratch org and deploys. The `.gitignore` also excludes `.artifacts/`, every runtime report under `reports/` (validate JSON, coverage summary, quick-deploy records), and `backups/**`, because those can contain usernames or org identifiers.

## Apex testing

### 15. What does `RunLocalTests` do, and why use it everywhere?
**It runs every Apex test in the org except managed-package tests.** It's used for the scratch-org test step, for the PR check-only validation, for the staging and production deploys, and in smoke checks. Running all local tests rather than a hand-picked list means a new class can't escape testing because someone forgot to add it. The catch, which I document, is that on an org you can't refresh, like Org B, `RunLocalTests` also runs any leftover tests already in that org.

### 16. Describe your Apex test design.
**Seven test methods covering defaults on insert and update, provided values, null and empty input, a 200-record bulk insert, and both directions of the flow.** `ReleaseChecklistItemServiceTest` doesn't use `SeeAllData`, so each test sees only its own rows. The bulk test inserts 200 records in one transaction to prove the trigger and service are bulk-safe. One test calls the service directly with `null`, an empty list, and a list containing a null element, which hits the guard clauses. A Node test asserts there are exactly 7 test methods and no `SeeAllData`.

### 17. How do you test a flow from Apex?
**You perform the DML that triggers the flow and assert on the field the flow sets.** `Stamp_Ready_For_Review` is a record-triggered before-save flow that sets `Ready_For_Review__c` when Status is Ready. Two tests insert a record, update Status to Ready or away from Ready, re-query, and assert the checkbox is set or cleared. That only works if the flow is deployed and Active, so the runbook says to deploy all of `force-app` rather than just the classes.

### 18. Why an 85% gate when Salesforce requires 75%?
**The 75% rule is enforced on production deployments. Developer Edition Org B doesn't enforce it on these validations, so without my own gate, weak coverage would pass.** `scripts/coverage_gate.py` sets 85% for each non-test class and each trigger, not just the org-wide average, so one poorly tested class can't hide behind good ones. Test classes ending in `Test` or `_Test` are excluded. The threshold is configurable, but the troubleshooting guide says to fix findings rather than lower it.

### 19. How exactly does the coverage gate compute and decide?
**Covered lines are `numLocations − numLocationsNotCovered`, the percentage uses `Decimal` with half-up rounding to two places, and the gate fails closed on anything unusable.** 85.00% passes and 84.99% fails. It also fails on a Failed, Error, or Canceled validation status, any test failure, any component failure, empty coverage, or a deployed class or trigger with no coverage row. Malformed JSON exits 2, and gate failures exit 1. Against the seven committed fixtures it returns 0 for pass and exact threshold, 1 for low, missing, empty, and test-failure, and 2 for malformed. It always writes `coverage-summary.md`, so the PR artifact explains the failure.

## Code Analyzer

### 20. What is Salesforce Code Analyzer, and how do you run it?
**It's Salesforce's static-analysis CLI plugin. It bundles engines like PMD and ESLint and reports violations with severities from 1 (Critical) to 5 (Info).** The PR workflow installs `code-analyzer@latest` and runs `sf code-analyzer run --workspace force-app --rule-selector Recommended --severity-threshold 2 --output-file reports/code-analyzer.html`. `Recommended` uses Salesforce's recommended rule set rather than a custom list. The HTML report goes into the `pr-validation` artifact.

### 21. What does `--severity-threshold 2` mean?
**The command fails if any violation is severity 2 (High) or more severe (1, Critical).** Severity 3 (Moderate), 4 (Low), and 5 (Info) are reported but don't fail the PR. The triage engine mirrors exactly this boundary: severity 1 and 2 are a Code Analyzer Violation, while a file with only a severity-3 finding classifies as Unknown. One evaluation fixture tests that case.

### 22. Why does Code Analyzer run before the org steps, and why is Java installed?
**It's the cheapest gate, because it needs no org and no API calls, and PMD needs Java.** Failing on a Critical finding before logging in to Org B saves API calls on an org with a 15,000-per-day allocation. The workflow installs Temurin 17 because PMD needs Java 11 or later. The troubleshooting table covers "Code Analyzer cannot find Java" and the missing `reports/` folder, which is why `mkdir -p reports` runs first.

### 23. What would you do with a false positive?
**Fix the code if reasonable, otherwise suppress that specific finding with justification, and never raise the threshold.** The PR validation doc says explicitly not to raise the threshold to hide a finding. A targeted suppression is visible in code review and doesn't weaken the gate for everyone else. In this project the sample code is small and no suppression exists.

## Validation deployments

### 24. What is a check-only validation?
**`sf project deploy validate` compiles and deploys the metadata, runs the requested tests, then rolls everything back, so nothing is saved.** It answers "would this deploy succeed?" without changing the org. It returns a deploy request ID (`0Af…`), which on a production org can be used for a quick deploy within 10 days. On pull requests, I use it against Org B purely as a check. The PR workflow never deploys, never quick-deploys, and never reuses that job ID.

### 25. Walk me through the PR workflow's steps and why they're ordered that way.
**Setup (Java 17, Node 20, CLI), then Code Analyzer, Org Login, Deploy Validation, Coverage Gate, and Artifact Upload.** Static analysis first, because it's free. Login only once that passes. Validation writes `reports/validate.json`. The coverage gate runs `if: always()` whenever that JSON exists, so even a failed validation produces a readable summary. The artifact upload also runs `if: always()` with `if-no-files-found: warn`, so a missing file never masks the real error. Artifacts are kept for 14 days.

### 26. How do you authenticate CI to Salesforce securely?
**Each org's `sfdxAuthUrl` is stored as a GitHub secret. At run time it's written to a `umask 077` temp file, used with `sf org login sfdx-url`, and deleted immediately.** The value comes from `sf org display --verbose --json` on a trusted machine and is never committed. That output contains a refresh token. The workflow fails clearly if the secret is empty. GitHub doesn't pass secrets to pull requests from forks, so fork PRs fail at login by design, and the docs say to open PRs from a branch in the repository.

### 27. How does quick deploy work, and what happens on Developer Edition?
**Quick deploy promotes a validation that already ran its tests, using the validation's job ID, so tests don't run again. Salesforce documents it as a production-org operation.** The production workflow validates Org A, `parse_job_id.py` extracts the ID, but only if the status is Succeeded and the ID matches the `0Af…` pattern, and `production_deploy.sh` runs `sf project deploy quick --job-id`. Org A is Developer Edition, so the expected result is a refusal, which triggers the fallback.

### 28. Explain your quick-deploy fallback logic.
**A normal deploy runs only if the CLI's message matches one of nine specific "production only" refusals. Any other failure stops the release.** `quick_deploy_decision.py assess` returns `succeeded` for exit 0, `unsupported` for a recognised refusal, and `failed` otherwise. For `unsupported`, the script runs `sf project deploy start` with `RunLocalTests`, so tests run again because the validated job wasn't accepted. It writes `quick-deploy.json` with the job ID, `quickDeploySupported`, `fallbackUsed`, `outcome`, the reason including the CLI text, and the parsed outputs. For an expired job ID or `INVALID_SESSION_ID`, it records `failed` and doesn't deploy. A broad fallback would turn real failures into silent deploys.

### 29. What do your smoke checks verify, and when do they run?
**Five checks after every staging deploy, production deploy, and rollback: the object describes, the permission set, LWC, and flow are listed in metadata, and `RunLocalTests` passes.** `smoke_check.sh <alias>` uses `sf sobject describe`, three `sf org list metadata` calls parsed with a small Python snippet, and `sf apex run test`. Any miss prints `SMOKE FAIL: …` and exits 1. In production, the tag step runs only after smoke exits 0, so a failed smoke never produces a release tag.

### 30. How are staging and production deploys different, and how are releases recorded?
**Staging deploys automatically on every push to `main`. Production runs only from a manual dispatch with a typed `deploy`, validates first, and produces a tag and an artifact.** Both use concurrency groups with `cancel-in-progress: false`, so a second run queues rather than interrupting a deploy. Staging has `contents: read` permissions. Production has `contents: write`, which it needs to push the tag. The tag is `release-YYYYMMDD-HHMM` in UTC, annotated, created only after smoke passes, and never moved. If the tag already exists, the job fails. The `production-release` artifact keeps the validation JSON, the coverage summary, the job ID, the quick-deploy record, and any fallback output for 14 days.

## Rollback

### 31. How does your rollback work?
**`scripts/rollback.sh` takes the latest `release-*` tag (or `--tag`), runs `git archive` on it into a temp directory, redeploys that tag's `force-app` with `RunLocalTests`, optionally applies a destructive manifest, and smoke-checks.** Using `git archive` means the operator's working tree and branch are never touched. Zero-padded tag names sort chronologically, which is how "latest" is found. The default target is `prod`, and `--target-org staging` rolls back Org B. `--dry-run` prints the plan and calls nothing.

### 32. How do you remove components that were added after the release you're rolling back to?
**With `destructiveChangesPost.xml`, but only the copy inside the tag being restored, and only if it lists members.** Redeploying an older `force-app` doesn't delete newer components, so the script checks that tag's manifest. If it contains `<members>`, it runs a second deploy with `--manifest manifest/package.xml --post-destructive-changes …`. The committed file lists none, so today a rollback deletes nothing and prints `destructive: skip`. Making the delete list explicit for each release prevents accidental deletions.

### 33. Why doesn't rollback restore data?
**Metadata rollback and data recovery are different problems, and combining them would over-promise.** Redeploying metadata doesn't bring back records, and a destructive change that removes a field drops its values. Developer Edition has no sandbox copy to recover from. The script prints `data: metadata-only rollback; org records are not restored` before deploying. Records are handled by a separate restore plan with its own rules.

### 34. What are rollback's failure modes, and how do you handle them?
**It refuses unsafe input, stops on deploy errors, and never pretends a partial rollback succeeded.** With no `release-*` tag it exits 1 and won't guess. A badly named tag exits 2. A tag missing from the clone exits 1, and you run `git fetch --tags`. If the source deploy fails, smoke doesn't run and the org may still be on the newer metadata. If the destructive deploy fails after the source deploy, the org has the old source but the extra components remain. The runbook gives the operator action for each and says not to start a new release from a half-rolled-back state.

## Data operations

### 35. How do you seed a scratch org?
**`scripts/seed.sh` bulk-imports 3 synthetic `Seed:` checklist rows, verifies at least 3 exist, then creates one synthetic user and a permission set assignment only if a Salesforce license is free.** The user's username is rewritten per alias (`synthetic.seed.<alias>@example.com`), because usernames are globally unique. If no license is free, it skips the user and assignment but keeps the rows. A `DUPLICATE_VALUE` on the assignment is treated as already present. The script warns if pointed at `prod` or `devhub-org-a`, because that's the Dev Hub.

### 36. How do you handle bulk loads and their errors?
**`scripts/bulk-load.sh` uses `sf data import bulk` and always produces success and error CSV files. The sample includes one deliberately invalid row.** The bulk CSV has 3 rows, and `NotAStatus` violates the restricted picklist on `Status__c`. `--preflight` classifies the rows locally into `bulk-success.csv` and `bulk-errors.csv` without calling Salesforce. A real run replaces those files with the Bulk API's `successfulResults` and `failedResults` for the job. A completed job with exactly one failed row is the expected outcome. Including a failure means the error path is always exercised.

### 37. Describe your backup design.
**Three committed SOQL queries export to CSV and JSON in a dated UTC folder, with a manifest recording row counts. The folder is never overwritten and never committed.** The queries cover checklist items, active users, and permission set assignments that aren't profile-owned. `backup.sh` writes six files plus `backup-manifest.json` (date, target org, created time, `syntheticDataOnly`, `orgClass`, and per-object query, CSV, JSON, and row count). It refuses to overwrite a day's folder without `--force`. `backups/` is gitignored because the user export contains usernames and emails. The nightly routine keeps 7 folders locally. If a query hits the row limit, the runbook says to switch that object to `sf data export bulk`.

### 38. How does the restore work, and what gets stripped?
**The restore prints a plan by default. With `--apply --backup <dir> --target-org <alias>` it re-inserts checklist rows only, after removing IDs, owners, audit fields, and relationship columns.** IDs belong to one org, so a row from Org B gets a new ID in a scratch org. `OwnerId` is removed, so the importing user becomes the owner. It drops system columns, any dotted relationship column, and any `__r` column, requires `Name`, and skips blank rows. Applying twice duplicates rows, which is documented as a known risk with the fix: delete the extras or refresh the scratch org.

### 39. Why don't you restore users, and how do Developer Edition limits shape data work?
**Licenses are scarce, usernames are globally unique, and backups contain no passwords or sessions, so restoring users would fail or collide.** Developer Edition orgs typically have 2 Salesforce licenses and the admin uses one. The plan says to match existing users by username and recreate assignments manually, outside `--apply`. More generally, Org A and Org B have 5 MB of data storage and 15,000 API requests per day, and a backup is six queries. That's why data stays synthetic and small, and why backups run against Org B rather than the Dev Hub.

## Lifecycle Center

### 40. What is Salesforce Lifecycle Center?
**According to my study notes, it's Salesforce's platform-native change and release management feature, described as the current form of what was called DevOps Center.** The notes, from Trailhead and Salesforce Help and marked Studied, say it's turned on in an org rather than installed as a package, keeps Git as the source of truth while Salesforce creates and manages work-item branches, uses DX Inspector in the development org to track changes for commit, and lets a release manager promote from the UI or the CLI. They also record that new installs of the older DevOps Center managed package stopped being supported in April 2026. I didn't verify any of this hands-on.

### 41. When would you choose Lifecycle Center over your CLI pipeline?
**When a team wants Salesforce to manage work items and branches, admins build in sandboxes and commit through clicks, and the org can enable the feature.** It suits mixed admin and developer teams who need one visible pipeline. My guide also notes what it doesn't replace, based on the pages I read: there's no documented one-click undo to replace metadata rollback, and no backup product. A team adopting it still needs decisions for both. The CLI path fits when Git and `sf` in CI are already the norm and there's no sandbox pipeline.

### 42. Why didn't you implement Lifecycle Center?
**Its documented declarative path develops in a sandbox, and Developer Edition orgs can't create sandboxes.** I also didn't want to rename my CLI jobs and call them Lifecycle Center. So M5 delivered a comparison table, a selection guide, and notes, all labelled Studied, plus an explicit list of what I didn't do: open it, connect a repo, create a work item, or promote. Points from a secondary 2026 summary, such as edition eligibility and testing gates, are marked as unverified.

## Flosum

### 43. What is Flosum?
**Flosum is a commercial, Salesforce-specific release platform. Its documentation describes pipelines that bind a branch to orgs with validation, test, and deploy steps, plus separate Backup & Archive and Data Migrator modules.** My notes separate the product documentation from marketing claims. Metadata-type-aware merge, one-click rollback with impact analysis, drift detection, and optional Git are vendor-stated on the product page. One detail I noted from their docs is that "Pipeline ran successfully" means the job started, not that every step passed.

### 44. When would a team choose Flosum?
**When they need vendor-hosted release management with deployment rollback, drift detection across many orgs, compliance-grade approval and audit trails, or real backup and org-to-org data migration, and can procure it.** My selection guide lists those conditions and adds that vendor-stated behaviours should be proven in the team's own tenant. It also says Flosum would be a poor fit as a hidden dependency of this project: there's no production org to protect and no sandbox topology for drift detection.

### 45. How did you handle vendor claims in your comparison?
**I labelled every cell by evidence: Implemented for my CLI path, Studied for documented behaviour, and "vendor-stated" for marketing claims, and I didn't use competitor comparison tables as facts.** For example, Flosum's one-click rollback is marked vendor-stated and not executed. Lifecycle Center's testing gates come from a secondary summary and are marked as not re-proven. The comparison also has a "What this comparison refuses to claim" section. It doesn't claim a Flosum pipeline ran, a work item was promoted, or that Org A is a production org.

## AI failure analysis

### 46. What does your AI-assisted triage do?
**It reads a saved failure report, classifies it into one of eight categories with a fixed confidence, and drafts a likely cause, a next step, and a suggested PR comment for a person to review.** Inputs are files a validation already wrote: `validate.json`, a coverage summary, Code Analyzer JSON, Apex failure lists, or raw error text. Stage 1 is a rules file. Stage 2 is a local template that writes operator-friendly text. It's AI-assisted in the sense of automating diagnosis. It isn't an autonomous agent, and it doesn't call a model.

### 47. How does the classifier decide between overlapping signals?
**A fixed priority order, where the first match wins: Authentication, Permission, Code Analyzer (severity 1–2), Apex Test, Coverage, Metadata Conflict, Missing Component, then Unknown.** Auth and permission errors outrank test and coverage noise, because a deploy that never authenticated never reached those gates. A test failure outranks low coverage on the same run, and coverage is mentioned only when tests passed. Metadata Conflict beats Missing Component when both phrases appear. The coverage logic mirrors the gate: 85%, excluding `Test` and `_Test` classes, so 85.00% isn't a failure.

### 48. Why rules and templates instead of an LLM?
**Determinism, testability, auditability, and no data egress, which matter more than fluent prose in a release pipeline.** The same input always gives the same category, the logic is reviewable in `rules.yml`, it runs offline on the Python standard library, and no org data leaves the machine. `explain()` even accepts a "model configured" flag and ignores it, and a test confirms the output stays rule-based. A local model could rephrase Stage 2 later, but Stage 1 would still decide the category and the human gate would stay.

### 49. What stops the triage tool from doing something harmful?
**It has no write path, and it actively blocks dangerous suggestions.** It makes no Salesforce, GitHub, or network calls and has no credentials. It can't deploy, roll back, merge, approve, change metadata, or post a comment. If an echoed error contains `sf project deploy start`, `sf project deploy quick`, or `git merge`, those strings are redacted, and if any remain the report is refused with an error. Every report and suggested comment includes "Human review required.", and a test confirms the engine source doesn't shell out or call a model.

### 50. How did you evaluate it, and what do the numbers mean?
**Fifteen committed fixtures with expected labels, scored by the same `classify_path` the CLI uses: 15 correct, 0 incorrect, 2 expected Unknown, accuracy 1.0.** The fixtures include precedence traps (auth plus test failure, permission plus 82% coverage, test failure plus low coverage) and two true negatives, a disk-space error and a severity-3-only finding, so the score rewards not guessing. Confidence values are fixed per category (0.88 to 0.97 for known categories, 0.34 for Unknown), not learned probabilities. The fixtures were written for this project, so the result shows the rules behave as designed. It's not a measure of performance on real production failures. Running it on real artifacts from a first live run is the next step.
