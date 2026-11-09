# Resume Bullets

Short, medium, and long versions for five target roles. Every number is taken from the repository or from its local test runs.

> **Wording rule.** The workflows and scripts are built and tested locally, but haven't run against a Salesforce org yet. Use verbs like *designed, built, implemented, automated, authored,* and *validated locally*. Don't write *deployed to production*, *managed releases for*, *reduced failures by X%*, *sandbox*, *Enterprise org*, or *Flosum implementation*. If asked, say: "Built for Developer Edition orgs. Locally validated. The live org run is the next step."

## Project header (use once per resume)

**Salesforce DevOps Pipeline** · Personal project · Salesforce DX (API 68.0), Apex, Flow, LWC, Salesforce CLI, Code Analyzer, GitHub Actions, Python, Bash · Developer Edition orgs

## Verified numbers you can use

| Metric | Value |
|---|---|
| GitHub Actions workflows | 3 (PR validate, staging deploy, manual production stand-in deploy) |
| Coverage gate | 85% per Apex class and per trigger (platform production minimum is 75%) |
| Local tests | 83 Python + 8 Node |
| Operational scripts | 12 (7 Bash, 5 Python) |
| Smoke checks per deploy | 5 |
| Apex tests | 7 methods, including a 200-record bulk insert |
| Triage | 8 failure categories, 15/15 fixtures correct (accuracy 1.0, 2 expected Unknown) |
| Quick-deploy refusal patterns | 9 |
| Runbooks and reference docs | 19 (~25,500 words) |
| Backup catalog | 3 SOQL queries → CSV + JSON + manifest |
| Milestones | 6 |

---

## 1. Salesforce Technical Operations

**Short**
- Built a Salesforce DX release and operations toolkit (3 GitHub Actions workflows, 12 scripts, 19 runbooks) covering scratch orgs, gated validation, deploys, rollback, and data backup on Developer Edition orgs.

**Medium**
- Built a Salesforce DX release pipeline with GitHub Actions: Code Analyzer, check-only validation with `RunLocalTests`, an 85% per-class coverage gate, a staging deploy, and a manually approved production stand-in deploy with 5 automated smoke checks.
- Wrote operational runbooks (19 docs) with expected results and recovery procedures for Dev Hub and scratch-org quotas, source-tracking conflicts, failed deploys, and Developer Edition limits (API, storage, licenses).
- Automated synthetic data operations (seed, Bulk API load with an error file, CSV and JSON backups with a manifest, and a restore plan that strips IDs and owners and never inserts users), validated by 26 local tests.

**Long**
- Designed an end-to-end Salesforce release path for Developer Edition orgs: source-tracked scratch-org development, a check-only PR validation, an automatic staging deploy, and a typed-approval production stand-in release.
- Enforced quality gates in CI: Salesforce Code Analyzer (fails on Critical and High), `sf project deploy validate` with `RunLocalTests`, and a Python coverage gate requiring 85% per Apex class and trigger that fails closed on malformed or incomplete results.
- Implemented release controls: a 0Af job-ID check, quick deploy with a narrowly scoped fallback that is recorded when Developer Edition refuses it, 5 smoke checks, and immutable UTC `release-YYYYMMDD-HHMM` tags.
- Built a metadata rollback by release tag (`git archive`, redeploy, an optional destructive manifest, smoke) that prints its data limitation before acting.
- Automated data operations with Salesforce CLI and Bulk API scripts: synthetic seeding with license checks, a bulk load with success and error files, dated backups from 3 SOQL queries with a manifest, and a checklist-only restore plan.
- Authored 19 runbooks and reference docs covering scratch-org quotas (3 active, 6 per day), storage (5 MB), API limits (15,000 per day), and recovery procedures R1–R8. Validated all tooling locally with 83 Python tests and 8 Node tests.

## 2. Salesforce DevOps Engineer

**Short**
- Built a GitHub Actions CI/CD pipeline for Salesforce DX with Code Analyzer, check-only validation, an 85% Apex coverage gate, staging and manually approved production stand-in deploys, release tagging, and tag-based rollback.

**Medium**
- Implemented 3 GitHub Actions workflows for Salesforce DX (API 68.0): PR validation (Code Analyzer severity ≤ 2, `sf project deploy validate`, `RunLocalTests`), a deploy on merge to a staging org, and a `workflow_dispatch` production stand-in release.
- Wrote a Python coverage gate enforcing 85% per class and per trigger, with a markdown summary artifact and fail-closed exit codes (0 pass, 1 gate failed, 2 malformed). Verified against 7 validation fixtures.
- Engineered a quick-deploy path that validates first, checks the 0Af job ID, falls back to a normal deploy only on 9 recognised edition refusals, and records each decision as a JSON artifact.

**Long**
- Designed a Salesforce DX CI/CD pipeline on GitHub Actions spanning PR validation, staging deployment, a manually approved production stand-in release, smoke testing, release tagging, and rollback, all built for Developer Edition orgs.
- Secured CI authentication with `sfdxAuthUrl` GitHub secrets written to `umask 077` temp files and deleted after login. Fork PRs fail closed. Concurrency groups queue deploys instead of cancelling them.
- Built quality gates: Salesforce Code Analyzer (`Recommended` rules, Critical and High fail), check-only `sf project deploy validate` with `RunLocalTests`, and a Python coverage gate (85% per class and trigger, `Decimal` half-up rounding, missing-row detection).
- Implemented quick deploy with a precise fallback: a normal deploy runs only when the CLI refusal matches 9 known "production only" messages. Auth or expired-job errors stop the release. The outcome is written to `quick-deploy.json`.
- Created a tag-based metadata rollback (`git archive` of a `release-YYYYMMDD-HHMM` tag, redeploy, a per-tag destructive manifest, 5 smoke checks) with dry-run support and guarded exit codes.
- Covered the pipeline logic with 83 Python tests and 8 Node tests that run without Salesforce credentials, and documented each workflow with negative-test runbooks.

## 3. Release Engineer

**Short**
- Engineered a gated Salesforce release process with check-only validation, typed manual approval, recorded quick-deploy decisions, immutable UTC release tags, and tag-based metadata rollback.

**Medium**
- Designed a release flow (PR validate → staging deploy → typed-approval production stand-in → smoke → tag) with GitHub Actions and the Salesforce CLI, producing a 14-day release artifact with validation, coverage, job-ID, and deploy records.
- Implemented immutable `release-YYYYMMDD-HHMM` UTC tags, created only after 5 smoke checks pass, and a rollback script that redeploys any release tag with dry-run, format, and existence guards.
- Wrote a deployment runbook with a failure-recovery matrix and expected outputs for failed validation, failed smoke, an unsupported quick deploy, and rollback failures.

**Long**
- Owned the release design for a Salesforce DX project: a PR gate on a staging stand-in, an automatic deploy on merge, and a manual production stand-in release that runs only after the operator types `deploy`.
- Built release safety controls: validate-before-deploy, a 0Af job-ID check that refuses failed validations, quick deploy with an edition-specific fallback, and a tag step that refuses to move an existing tag.
- Standardised release records: annotated UTC `release-YYYYMMDD-HHMM` tags after smoke, plus a `production-release` artifact (validate JSON, coverage summary, job ID, quick-deploy decision, fallback output) kept for 14 days.
- Delivered metadata rollback by release tag using `git archive`, a per-release destructive manifest, and post-rollback smoke, with explicit "metadata-only" output that separates rollback from data restore.
- Wrote release documentation: a deployment runbook with a 9-row failure-recovery matrix, a three-way comparison of the Salesforce CLI, Lifecycle Center, and Flosum (the latter two studied from documentation), and a tool selection guide.
- Validated release tooling with 23 deployment tests (tag format, job-ID parsing, quick-deploy decisions, rollback dry runs against temporary git repositories).

## 4. Platform Engineer

**Short**
- Built dependency-free Python and Bash platform tooling for a Salesforce delivery pipeline (coverage gate, deploy decisioning, rollback, backups, rules-based failure triage), covered by 83 Python tests and 8 Node tests.

**Medium**
- Wrote 12 operational scripts (5 Python, 7 Bash) on the standard library only, with explicit exit-code contracts, `--dry-run` modes, and fail-closed input validation for CI and operator use.
- Built a rules-based deploy-failure triage engine (8 categories, priority-ordered precedence, a built-in YAML-subset parser) that scored 15/15 on a fixture suite and blocks deploy and merge commands from its output.
- Designed a CI security posture for org credentials: secrets in temp files with `umask 077`, least-privilege workflow permissions (`contents: read`, with write only for tagging), and gitignored auth state, reports, and backups.

**Long**
- Engineered platform tooling for a Salesforce DX pipeline: a coverage gate, a job-ID parser, a quick-deploy decision engine, a release tagger, a smoke checker, a rollback, and data scripts, all on the Python standard library and Bash, with no package installs.
- Defined consistent contracts across tools: exit codes (0 success, 1 rule failure, 2 bad input), `--dry-run` planning output, and refusal of unsafe defaults (overwriting backups, an untagged rollback, applying a restore without a target).
- Built a two-stage advisory triage engine: priority-ordered rules over JSON fields and text patterns, then template explanations, with redaction and refusal of deploy and merge commands. Evaluated at 15/15 fixtures, including precedence and true-negative cases.
- Implemented least-privilege CI: per-workflow `contents` permissions, deploy concurrency groups, secrets handled through temporary files, and artifacts kept for 14 days.
- Made the scripts cross-platform, with LF line endings enforced through `.gitattributes` and WSL-aware path handling in tests, so the same tooling runs on Linux CI runners and on Windows with WSL.
- Achieved 83 Python tests and 8 Node tests across triage (23), coverage gate (11), deployment (23), data operations (26), and repository structure, all runnable offline.

## 5. Salesforce Application Support

**Short**
- Built runbooks and tooling to diagnose Salesforce deployment failures, including a rules-based triage tool that classifies 8 failure types (auth, permissions, Apex tests, coverage, missing components) and drafts next steps for human review.

**Medium**
- Created a deploy-failure triage tool that reads saved Salesforce validation results and classifies them as authentication, permission, Code Analyzer, Apex test, coverage, metadata conflict, missing component, or unknown, with a confidence level and a suggested next step.
- Wrote support runbooks with expected results and recovery steps for common Salesforce issues: expired sessions, scratch-org limits, source-tracking conflicts, failed tests, `REQUEST_LIMIT_EXCEEDED`, and storage limits.
- Built a sample Salesforce app (custom object, Apex trigger and service, record-triggered flow, LWC, permission set) with 7 Apex tests, including a 200-record bulk test.

**Long**
- Developed a Salesforce deploy-failure triage tool that turns raw validation JSON and error text into a category, a likely cause, a next step, and a reference to the exact report section. Every report requires human review.
- Encoded support knowledge as priority rules: authentication and permission errors before test noise, test failures before coverage, Code Analyzer Critical and High only. Verified on 15 fixtures with 15/15 correct, including 2 correctly Unknown cases.
- Authored troubleshooting and recovery runbooks (19 docs) for Dev Hub logins, scratch-org quotas, source-tracking conflicts, failed deploys and tests, API and storage limits, and data reload procedures.
- Built smoke checks that confirm a release is present and healthy (object, permission set, LWC, flow, Apex tests) after deploys and rollbacks.
- Created data support tooling: seeding with a license check, a bulk load with an error file, dated backups with a row-count manifest, and a restore plan that avoids duplicate users and foreign IDs.
- Built and tested the sample application: a custom object with a restricted picklist, an Apex service and trigger, a before-save flow, an LWC, and a permission set, with 7 Apex test methods.

---

## Skills line (ATS)

Salesforce DX, SFDX, Salesforce CLI (`sf`), scratch orgs, Dev Hub, source tracking, Apex, Apex triggers, Apex testing, code coverage, Flow, Lightning Web Components (LWC), permission sets, Metadata API, `package.xml`, destructive changes, Salesforce Code Analyzer, PMD, GitHub Actions, CI/CD, release management, quick deploy, release tagging, rollback, Bulk API, SOQL, data backup and restore, Python, Bash, Git, runbooks, incident triage
