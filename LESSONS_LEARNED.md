# Lessons Learned

A retrospective on building the Salesforce DevOps Pipeline across six milestones on Developer Edition orgs. Each lesson names what happened, what it changed in the repository, and what I'd do next time.

---

## Platform and environment

### 1. Design around the edition you have, then say so
**Observation.** Developer Edition orgs can't create sandboxes, can't be source-tracked, and aren't production orgs. Quick deploy, sandbox refresh, and the 75% coverage enforcement all assume otherwise.
**What changed.** The topology became one scratch org (tracked), Org B (staging stand-in), and Org A (Dev Hub and production stand-in). Every limitation is listed in `docs/developer_edition_limits.md` with its source.
**Next time.** Write the edition constraints document *first*. It decided more of the design than any tool choice.

### 2. Quotas are a design input, not an operational nuisance
**Observation.** Org A allows 3 active and 6 daily scratch orgs. Deleting an org frees an active slot but not the daily create.
**What changed.** PR validation uses a check-only deploy to Org B rather than a scratch org per PR. The runbook checks `sf limits api display` before every create and has explicit recovery steps (R2, R3).
**Next time.** Treat each pipeline stage's API and quota cost as part of its design review.

### 3. A shared Dev Hub and production stand-in is a real trade-off
**Observation.** With two orgs, Org A had to be both. A production release therefore writes metadata into the Dev Hub.
**What changed.** The trade-off is stated in the README, the deployment runbook, and the architecture docs. The seed script warns if it's pointed at `prod` or `devhub-org-a`.
**Next time.** With a third Developer Edition org, separate the Dev Hub from the production stand-in.

### 4. Unrefreshable orgs accumulate state
**Observation.** `RunLocalTests` runs every local test already in Org B, and Org B can't be refreshed.
**What changed.** The runbooks tell operators to keep Org B limited to this project's tests, and they treat a failure in a test you didn't change as a symptom of leftover state.

## Release engineering

### 5. Fallbacks should be narrow
**Observation.** Quick deploy is expected to be refused on a Developer Edition stand-in. The easy fix, falling back to a normal deploy on any error, would also hide expired job IDs and auth failures.
**What changed.** `quick_deploy_decision.py` returns `unsupported` only for 9 specific refusal phrases. Everything else is `failed`, and no deploy runs. The decision and the CLI text are recorded in `quick-deploy.json`.
**Next time.** Keep the refusal phrases under test against real CLI output once a live run exists.

### 6. Gates should fail closed
**Observation.** A coverage gate that treats "no data" as "no problem" is worse than no gate.
**What changed.** `coverage_gate.py` fails on empty coverage, a missing coverage row for deployed Apex, unsupported types, and invalid counts, and exits 2 on malformed JSON. `parse_job_id.py` refuses any validation that isn't Succeeded.

### 7. Per-row thresholds catch what averages hide
**Observation.** An aggregate of 85% can conceal one class at 40%.
**What changed.** The gate checks 85% for each non-test class and each trigger, and also reports the aggregates.

### 8. Tag only known-good states, and never move a tag
**Observation.** A tag is only useful as a rollback target if it means "this passed smoke".
**What changed.** The tag step runs after the smoke check, refuses an existing tag, and uses zero-padded UTC names so that sort order equals time order.

### 9. Rollback output must state its limits
**Observation.** The most expensive Salesforce recovery mistake is believing a metadata rollback restored data.
**What changed.** `rollback.sh` prints `data: metadata-only rollback; org records are not restored` before it deploys. It uses the *tag's own* destructive manifest and extracts with `git archive`, so the operator's working tree is untouched.

### 10. Typed approval is a minimum, not an ideal
**Observation.** Typing `deploy` on a `workflow_dispatch` prevents accidents. It doesn't enforce who may approve.
**Next time.** Use GitHub Environments with required reviewers for the production job. This is listed in [ROADMAP.md](ROADMAP.md).

## Data operations

### 11. Don't restore users by default
**Observation.** Developer Edition orgs typically have 2 Salesforce licenses, usernames are globally unique, and exports contain no passwords or sessions.
**What changed.** The restore never inserts users or permission set assignments. The seed creates a user only when a license is free, with an alias-specific synthetic username.

### 12. IDs and owners don't travel between orgs
**What changed.** The restore strips `Id`, `OwnerId`, audit columns, and relationship columns before inserting. The importing user becomes the owner.

### 13. Make the failure path part of the sample
**Observation.** Error handling that never runs is never verified.
**What changed.** The bulk sample includes one invalid picklist value (`NotAStatus`), so every bulk run, including `--preflight`, produces an error file.

### 14. Backups contain personal data even in a demo
**What changed.** `backups/` is gitignored, folders are dated and never overwritten without `--force`, and the runbook says to keep 7 locally and never commit them.

## AI-assisted triage

### 15. Structural limits beat disclaimers
**Observation.** A "suggestion only" label doesn't stop a tool from emitting a command someone pastes.
**What changed.** The engine has no network, Salesforce, or GitHub calls. It redacts `sf project deploy start`, `sf project deploy quick`, and `git merge`, and refuses any report that still contains one. A test confirms the engine source doesn't shell out or call a model.

### 16. Precedence is the hard part of classification
**Observation.** Real failure reports contain several signals. An auth failure often appears next to test noise.
**What changed.** There's an explicit priority order and fixtures that test it: auth outranks tests, permission outranks coverage, and a test failure outranks low coverage.

### 17. Unknown is a correct answer
**What changed.** Two fixtures are expected to be Unknown, a disk-space message and a severity-3 finding, so the evaluation rewards not guessing.

### 18. Report what the score means
**What changed.** Every place that states "15/15, accuracy 1.0" also says it's local fixtures, not production failures and not model accuracy.

## Documentation and evidence

### 19. Write the "not executed" list
**Observation.** The authoring machine had no Salesforce CLI, so no org run was possible from it.
**What changed.** Each milestone records what wasn't run. That's why the rest of the claims can be trusted.

### 20. Keep docs in step with later milestones
**Observation.** `docs/deployment_runbook.md` still says "There is no data script in this milestone." That line was written in M3, before M4 added the data scripts.
**Next time.** Re-read earlier runbooks at each milestone close. A docs-consistency check would catch statements like this.

### 21. Tests must be portable to where CI runs
**Observation.** Node foundation test 4 validates XML by shelling out to `powershell`. It passes on Windows and fails with `ENOENT` on macOS and Linux runners, even though the XML itself is well formed.
**Next time.** Parse XML in Node or Python so the same suite runs on any runner.

### 22. Repository tests can protect documentation
**Observation.** The foundation and triage tests assert README headings and links.
**Effect.** Documentation rewrites have to keep those headings (Project Purpose, Architecture, Current Milestone Status, Roadmap, Developer Edition Constraints, AI Deploy-Failure Triage, Failure Classification, Human Review Gate). That's a useful guard against silently deleting operational content.
