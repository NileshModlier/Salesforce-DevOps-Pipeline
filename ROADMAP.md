# Roadmap

> **Status of everything on this page: not started.** These are proposals. None of them exists in the repository. The project's stated position is that later work stays documentation and that no M7 code is in the repository. Items that need new code are listed so the next step is clear, not to suggest it's underway.

## Where the project stands

| Milestone | Scope | State |
|---|---|---|
| M1 | DX project, Dev Hub, scratch orgs, source tracking, Apex tests, runbooks | Complete |
| M2 | PR validation: Code Analyzer, check-only validate, 85% coverage gate | Complete |
| M3 | Staging deploy, production stand-in deploy, quick-deploy fallback, tags, rollback | Complete |
| M4 | Seed, bulk load, CSV and JSON backups, backup catalog, restore plan | Complete |
| M5 | Lifecycle Center and Flosum comparison | Complete (Studied) |
| M6 | Advisory deploy-failure triage | Complete |

"Complete" means delivered as files and validated locally (83 Python tests, 8 Node tests). No milestone has been run against a Salesforce org yet.

## Guiding principles for anything next

1. **Evidence before features.** A live run of what exists is worth more than a new capability.
2. **Keep the evidence labels.** Implemented, Studied, and Not available stay accurate, and new claims need artifacts.
3. **Never widen authority quietly.** Triage stays advisory. Production stays manual.
4. **No paid or unavailable environments by implication.** Enterprise orgs, sandboxes, production tenants, and vendor tenants stay out of scope unless they're actually provided.

---

## Horizon 1 · Prove the existing pipeline live

Needs: GitHub secrets `STAGING_AUTH_URL` (Org B) and `PROD_AUTH_URL` (Org A), and a machine with the Salesforce CLI. No code changes.

| # | Item | Exit criteria |
|:-:|---|---|
| 1.1 | Run the M1 scratch-org runbook end to end | The scratch org is created. The deploy succeeds. The `Needs_Follow_Up__c` retrieve is committed. 7 Apex tests pass, with recorded coverage. |
| 1.2 | First **PR Validate** run on a real pull request | A green check, with the `pr-validation` artifact saved |
| 1.3 | Negative PR runs: a failing test, low coverage, a Code Analyzer severity 1 | Each fails at the documented step with the documented message |
| 1.4 | First **Deploy Staging** run | The deploy succeeds and all 5 smoke checks pass on Org B |
| 1.5 | First **Deploy Production** run | `quick-deploy.json` records the real outcome (expected `unsupported`). Smoke passes and the first `release-*` tag exists. |
| 1.6 | Rollback drill to that tag on Org B | `rollback.sh --target-org staging` redeploys, and smoke passes |
| 1.7 | Data drill on a scratch org | Seed (≥ 3 rows), bulk load (2 succeed, 1 expected error), backup manifest with 3 objects, restore `--apply` |
| 1.8 | Triage on real artifacts | Classify the saved negative-run artifacts from 1.3, and compare with the fixture expectations |
| 1.9 | Update the evidence | Replace "not executed" statements with dated, linked artifacts. Leave the fixture-based metrics labelled as such. |

## Horizon 2 · Hardening (small, local changes)

| # | Item | Why | Size |
|:-:|---|---|:-:|
| 2.1 | Make Node foundation test 4 portable, parsing XML in Node or Python instead of `powershell` | It currently fails on macOS and Linux runners with `ENOENT` | S |
| 2.2 | Add a CI job that runs the 83 Python tests and the Node suite on each PR | Repository validation is currently manual | S |
| 2.3 | Pin `@salesforce/cli` and `code-analyzer` versions instead of `@latest` | Reproducible validation results | S |
| 2.4 | Use GitHub **Environments** with required reviewers for `deploy-prod.yml` | Enforces *who* approves, not just that someone typed `deploy` | S |
| 2.5 | Upload a smoke and deploy artifact from `deploy-staging.yml` | Staging currently leaves only the job log | S |
| 2.6 | Add a docs-consistency check | Would catch stale lines such as the M3 runbook's "There is no data script in this milestone." | S |
| 2.7 | Record CLI refusal text from the first real quick-deploy attempt as a fixture | Keeps the 9 refusal phrases honest | S |
| 2.8 | Move the Release Approval step before CLI install in `deploy-prod.yml` | A rejected approval wouldn't spend runner time | XS |

## Horizon 3 · Optional extensions (each needs a decision)

| # | Item | Needs | Constraint that stays |
|:-:|---|---|---|
| 3.1 | A local language model to *rephrase* Stage 2 triage text | A model client, which the repository doesn't have | Stage 1 rules still decide the category. "Human review required." stays. No write path. |
| 3.2 | Post the suggested comment as a PR comment, behind a manual trigger | A GitHub token with comment scope | Advisory text only. No approval or merge authority. |
| 3.3 | A third Developer Edition org as a dedicated Dev Hub | Another sign-up | Org A becomes a cleaner production stand-in |
| 3.4 | Hands-on Lifecycle Center evaluation | An org allowed to enable it, plus the sandbox its declarative path uses | Stays separate from the CLI path. Results labelled by evidence. |
| 3.5 | Hands-on Flosum evaluation | A Flosum tenant | Vendor-stated features are proven or disproven in that tenant only |
| 3.6 | Scheduled nightly backup in CI | A secret-handling decision for user exports | `backups/` must never be committed |

## Explicitly not planned

- Calling a model with authority to deploy, roll back, merge, or approve
- Representing Org A as a real production org
- Describing Lifecycle Center or Flosum as implemented
- Restoring users from backups by default
- Adding a sandbox refresh process, because there's no sandbox to refresh

## How to read progress

| Label | Meaning |
|---|---|
| Not started | Listed here only |
| In progress | A branch or a pull request exists |
| Proven live | An artifact from a real org run is linked |
| Done | Merged, documented, and the evidence label updated |
