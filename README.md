# bib docs

Docs for developers working on bib (Business in a Box). The code itself is in
`bib-backend` and `bib-frontend`. Clone this repo beside them, in the same parent folder,
so paths like `bib-docs/backend/DEPLOY.md` in the code's comments resolve.

| Guide | What it's for |
|---|---|
| [Developer onboarding](developer-onboarding.md) | Start here: both repos, running locally, how the code is laid out, where to read next |
| [QA database access](qa-database-access.md) | Open the QA database in pgAdmin or psql on your laptop, logging in with your Google account |

## Backend

| Doc | What it's for |
|---|---|
| [HANDOFF](backend/HANDOFF.md) | Why we're building this and the founding decisions |
| [DEPLOY](backend/DEPLOY.md) | Deploying, step by step, and known failures |
| [Deploy runbook](backend/deploy/runbook.md) | Routine deploys, rollback, boot-gate reference |
| [Database](backend/deploy/database.md) | Cloud SQL roles and the migration contract |
| [Release notes](backend/deploy/) | One note per release, plus the client answers of 2026-10-05 |
| [MERGE_BUILD-PLAN](backend/MERGE_BUILD-PLAN.md) | The live task board |
| [BUILD-PLAN](backend/BUILD-PLAN.md) | Superseded history of the board |
| [build2](backend/build2.md) | The demo vs the rebuild |
| [DEMO](backend/DEMO.md) | The live demo runbook for 10x Global |
| [Security audit 2026-09-05](backend/SECURITY-AUDIT-2026-09-05.md) | Read-only security and bug audit; findings go to the board |
| [India compliance calendar](backend/India_Compliance_Calendar_Applicability_Business_in_a_Box.xlsx) | The firm's sheet the shipped rule pack was built from |

Docs that stay in the code repos, because tools or tests read them there: each repo's
`README.md` and `CLAUDE.md`, `bib-backend/CONTRIBUTING.md`, `bib-backend/docs/EVENT-CONTRACT.md`
(a test parses it), and the READMEs inside code folders (`migrations/`, `infra/`,
`src/design/`).

This repo is private. Ask Anish if you need access to it or to anything a guide mentions.
