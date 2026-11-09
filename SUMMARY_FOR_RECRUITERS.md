# Summary for Recruiters

## Salesforce DevOps Pipeline, in one minute

**What it is:** A portfolio project showing how to release Salesforce changes safely. Changes are checked automatically, moved through a test org, released only with a manual approval, recorded with a release tag, and rolled back if needed. It also covers backing up and reloading test data, and a tool that explains why a release check failed.

**Who built it:** One engineer, end to end: design, automation, testing, and documentation.

**Honest scope:** It's built for free Salesforce Developer Edition orgs. Everything is in the repository and tested on a laptop (91 automated tests), but it hasn't been run against a live Salesforce org yet. It doesn't involve Enterprise orgs, sandboxes, a production org, or a paid Flosum account.

---

## Role fit

| Role | Fit | Why |
|---|:-:|---|
| Salesforce Technical Operations | Strong | Runbooks, org limits, scratch orgs, data backup and restore, recovery procedures |
| Salesforce DevOps Engineer | Strong | CI/CD with GitHub Actions and the Salesforce CLI, quality gates, deploys, rollback |
| Release Engineer | Strong | Approval step, release tags, release records, rollback, comparison of release tools |
| Platform Engineer | Good | Python and Bash tooling, CI security, testing, safety-first scripting |
| Salesforce Application Support | Good | Failure triage tool, troubleshooting runbooks, smoke checks |

## Keywords

**Salesforce:** Salesforce DX, Salesforce CLI, scratch orgs, Dev Hub, source tracking, Apex, Apex testing, code coverage, Flow, Lightning Web Components (LWC), permission sets, Metadata API, Bulk API, SOQL, Salesforce Code Analyzer, quick deploy, Developer Edition

**DevOps:** GitHub Actions, CI/CD, release management, release tagging, rollback, smoke testing, deployment validation, secrets management

**Tools and languages:** Python, Bash, Git, Node.js, JSON, YAML

**Studied (not hands-on):** Salesforce Lifecycle Center, Flosum

## Numbers at a glance

| | |
|---|---|
| Automated workflows | 3 |
| Automated tests | 91 (83 Python + 8 Node) |
| Code coverage rule | 85% for every Apex class (Salesforce's production minimum is 75%) |
| Failure types the triage tool recognises | 8 |
| Triage accuracy on test cases | 15 of 15 |
| Runbooks and guides | 19 |
| Project milestones | 6 |

## Screening questions this project answers

| If you ask... | Short answer |
|---|---|
| "Have you used Salesforce DX and scratch orgs?" | Yes. The project uses a DX project, a Dev Hub, a scratch-org definition, source tracking, and runbooks with recovery steps. |
| "Have you built CI/CD for Salesforce?" | Yes. 3 GitHub Actions workflows: PR validation, staging deploy, and a manually approved production stand-in release. |
| "Do you know Apex testing and coverage?" | Yes. 7 Apex tests, including a 200-record bulk test, and an 85% coverage gate. |
| "Have you done rollbacks?" | Built a rollback-by-release-tag script, tested locally. It hasn't been run against a live org yet. |
| "Have you used Flosum or DevOps Center / Lifecycle Center?" | Studied both and wrote a comparison. Not used hands-on. |
| "Have you worked with production orgs or sandboxes?" | Not in this project. It uses Developer Edition orgs only. |
| "Does it use AI?" | The triage tool is AI-assisted, using rules plus templates. It doesn't call an AI model, and it always requires human review. |

## Where to look

- [README.md](README.md): the full project page
- [SUMMARY_FOR_HIRING_MANAGERS.md](SUMMARY_FOR_HIRING_MANAGERS.md): the technical version
- [RESUME_BULLETS.md](RESUME_BULLETS.md): bullets by role
