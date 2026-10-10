# BiB (Business in a Box): developer onboarding

BiB is a multi-tenant SaaS for professional-services firms: accounting, tax and compliance
practices. A firm's staff use the **10X Workspace** to run the whole practice:

```
Lead → Client → Proposal → Engagement letter (signed) → Work (jobs, tasks, filings) → Invoice → Payment
```

Their clients use the **client portal** to see what's owed, answer requests, get documents and
read invoices.

This guide covers both repositories. Read it first. The deep rules live in each repo's
`CLAUDE.md`; see [Where to read next](#where-to-read-next).

---

## The two repositories

| | `bib-backend` | `bib-frontend` |
|---|---|---|
| Repo | `github.com/anishumar/bib-backend` | `github.com/anishumar/bib-frontend` |
| Stack | Node 22, TypeScript, Hono, Postgres 17 + row-level security, pg-boss, zod, pino | Next.js 16 (App Router), React 19, openapi-fetch, Tailwind 4 + shadcn/Radix beside an in-house design system |
| Runs as | API server (`:8080`), background worker, migrate job | Web app (`:3000`) |
| Architecture | Hexagonal: `domain` ← `usecase` ← `infrastructure`/`api`, wired only in `setup/`. The arch test enforces it. | Client components call the API through a client generated from the backend's OpenAPI spec |
| Gate | `npm run check` (typecheck, lint, arch, tests) | `npm run check` (typecheck, lint, copy check) **and** `npm run build` |

Clone both into **one parent folder**, side by side. Scripts assume `../bib-backend` and
`../bib-frontend`:

```
bib/                          ← any folder name; not a git repo
├── bib-backend/
│   └── docker-compose.yml    ← local Postgres only (see step 2)
└── bib-frontend/
```

---

## Prerequisites

- **Node 22.22.1** via nvm (`bib-backend/.nvmrc`). Node 22 is required, not suggested. Under
  Node 20, npm installs `pg-boss` as an **empty folder** without an error. The first symptom is
  `Cannot find module 'pg-boss'` from typecheck. Run `nvm use` before every `npm install`.
- **Postgres 17 or 18** with `pgcrypto` and `citext`: either native on `:5432` or Docker.
- git, plus `gh` if you work with PRs from the terminal.

---

## First-time setup

### 1. Clone

```bash
mkdir bib && cd bib
git clone https://github.com/anishumar/bib-backend.git
git clone https://github.com/anishumar/bib-frontend.git
```

### 2. Database

With Docker (from `bib-backend`):

```bash
(cd bib-backend && docker compose up -d --wait)   # Postgres 17 on :5432, container "bib-postgres"
```

Or with a native Postgres:

```bash
createdb bib
```

### 3. Backend

```bash
cd bib-backend
nvm use
cp .env.example .env
npm install
npm run migrate        # creates roles, applies migrations, turns on the RLS wall
npm run dev            # http://localhost:8080/health
```

`.env` holds **two** database URLs:

- `BIB_DATABASE_URL` is the migrator. Locally it must be a superuser, because it creates roles
  and extensions.
- `BIB_APP_DATABASE_URL` is `bib_app`, the `NOBYPASSRLS` login every request uses. That
  login is what keeps one firm from reading another firm's data.

### 4. Frontend

```bash
cd ../bib-frontend
cp .env.example .env.local     # NEXT_PUBLIC_API_BASE_URL=http://localhost:8080
npm install
npm run api:generate           # backend must be running; writes src/lib/api/schema.d.ts
npm run dev                    # http://localhost:3000
```

The API is a **different origin** from the app, in every environment. If calls fail with
CORS errors, check that `http://localhost:3000` is in the backend's `BIB_CORS_ORIGINS`.

### 5. Get a firm and logins

On an **empty** database, with the API running:

```bash
cd bib-backend
npm run demo           # seeds "10x Global": people, clients, processes, calendar, invoices
```

To wipe everything and seed again, run `npm run demo:fresh`. It **destroys the local database
volume** and needs Docker.

The seed's logins (local only; the password is committed in `scripts/demo-seed.sh`):

| Email | Role | Notes |
|---|---|---|
| `siddharth@10xglobal.in` | Partner | Sees fees. Start here. |
| `ravi@10xglobal.in` | Manager | Sees fees |
| `ananya@10xglobal.in` | Associate | No fees; use it to test permissions |
| `farhan@10xglobal.in` | Associate | No fees, reports to nobody |
| `meera@cobaltfoods.in` | Client portal | What a client sees |

Password for all of them: `10x Global demo 2026`.

`npm run smoke` is the other way to get logins. It provisions throwaway firms and prints the
logins. **Never run smoke against a deployed environment.**

---

## Keep local runs from emailing real people

`.env.example` ships safe values:

```
BIB_MAIL_TRANSPORT=log           # emails are written to the log, not sent
BIB_ESIGN_PROVIDER=attested      # no real DocuSign envelopes
BIB_STORAGE_BACKEND=filesystem
```

If your `.env` has `BIB_MAIL_TRANSPORT=smtp` or `BIB_ESIGN_PROVIDER=docusign`, then the API
**and the worker** send real Gmail and real DocuSign envelopes to whatever addresses are in
your local data. Override them for a walk-through:

```bash
BIB_MAIL_TRANSPORT=log BIB_ESIGN_PROVIDER=attested npm run dev
```

---

## Daily commands

**Backend** (`bib-backend`)

| Command | Does |
|---|---|
| `npm run dev` | API server with watch |
| `npm run worker` | The background worker, long-running, on its real schedule |
| `npm run worker:once` | Runs every sweep **once** and exits. Use it when you don't want to wait for the schedule. |
| `npm run migrate` | Applies pending migrations |
| `npm run check` | **The gate**: typecheck, lint, arch, unit tests |
| `npm test` / `npm run test:integration` | Unit tests / tests against a real database |
| `npm run demo` / `npm run demo:fresh` | Seed 10x Global / wipe and reseed |

**Frontend** (`bib-frontend`)

| Command | Does |
|---|---|
| `npm run dev` | Next.js dev server on `:3000` |
| `npm run api:generate` | Regenerates API types from the running backend. Run it after **any** backend API change. |
| `npm run check` | Typecheck, lint, placeholder-copy check |
| `npm run build` | Production build. Run it before every merge: some errors only appear here. |

---

## The worker, and why it isn't an API call

A lot of what BiB does is triggered by **the date**, with nobody clicking anything:

- reminders on overdue invoices
- filings landing on the compliance calendar
- escalations for late tasks
- new jobs at the start of each period
- the daily exchange rates

So a separate always-on process, the worker (`src/cmd/worker.ts`), runs these as scheduled
sweeps. It uses pg-boss on the same Postgres.

- **Schedules are in UTC.** They're set in `src/config/config.ts`, overridable with
  `BIB_WORKER_*_CRON`. For example, `0 7 * * *` is 12:30 pm in India.
- **Some sweeps are frequent:** notifications every minute, moving work to its next step every
  5 minutes, creating new jobs hourly.
- **The rest are daily:** filings, escalations, reminders, exchange rates.
- **Locally you usually run nothing continuously.** Use `npm run worker:once` after an action
  to see its effects immediately.

---

## Architecture in five minutes

**Backend**

- **The import wall.** `domain/` is pure: types, rules, errors, no I/O. `usecase/` holds
  services and the `ports.ts` interfaces they need. `infrastructure/` implements those ports
  (Postgres, SMTP, DocuSign, storage). `api/` is routes and handlers. `setup/` is the only
  place things are wired together. `npm run arch` fails the build if an import points the
  wrong way.
- **The tenant wall.** Every business table has `tenant_id` and row-level security. Requests
  run as `bib_app`, which cannot bypass RLS, and the server **refuses to boot** if the wall
  isn't intact. Tenant-scoped access goes through `withTenant` / `withIdentity`, never the
  raw pool.
- **The ledger** (`ledger.event`) is append-only: who did what, when. It's the audit trail and
  the source of notifications.
- **Migrations** live in `migrations/NNNN_name.{up,down}.sql`. An applied migration is never
  edited; you fix forward with a new one. Each `up` has a matching `down`.
- **Config** is read only in `src/config/` (zod-validated). Adding a key means editing
  `config.ts` **and** `.env.example`.
- **Errors** come back in one envelope: `{ error: { code, message, reason? } }`.

**Frontend**

- **Never hand-write an API type.** Import from `@/lib/api/client`; it re-exports the
  generated `schema.d.ts`. Never edit `schema.d.ts`.
- **The session is an httpOnly cookie** that no script can read. There's no token in
  localStorage. To know who's signed in, call `GET /api/v1/auth/me` (the shell already does,
  via `useSession()`). A 401 means "go to `/login`", not an error to show.
- **Show the API's `message` as-is.** Some messages are deliberately vague so they don't
  reveal who has an account.
- **Use the design system.** Read `src/design/README.md` before building UI. Use the
  components (`<Button>`, `<Field>`, `<Table>`… from `@/components/ui`), never raw classes.
  There are no hex values, no inline font sizes or colours, and nothing off the spacing and
  type scale; lint enforces this. Colour means status: teal is done, amber is warning, red is
  wrong. The plum brand colour is used only for the mark, the sidebar and a screen's one
  primary action.
- **Two styling vocabularies coexist.** The design system is `@/components/ui` plus
  `src/design/*.css`. The Tailwind/shadcn layer is `@/components/tw` and
  `@/components/shadcn`, and a migrated screen starts with `<Page>`. Mix them per component,
  never on the same element. Frontend `CLAUDE.md` explains the CSS-layer rules behind this.
- **Navigation lives only in `AppShell`** (`src/components/layout/app-shell.tsx`). Pages don't
  build their own menus.

---

## Domain words you'll meet

| Word | Means |
|---|---|
| **Firm / tenant** | The practice using BiB. Everything is scoped to one. |
| **Party** | Any person or company. A **client** is a party the firm serves. |
| **Lead** | A prospective client (the sidebar calls the list "Pipeline"). |
| **Catalogue / service** | What the firm sells, with prices and the filings each service involves. |
| **Proposal** | A priced quote; the client accepts or declines it from an email link. |
| **Engagement letter** | The signed contract. Signing creates one **engagement** per line. |
| **Engagement** | One service the client is under contract for. Billing and work hang off it. |
| **Job** / **step** | One run of a firm's process for an engagement (`/work`), made of steps. |
| **Work item / task** | A unit of work on someone's desk (`/work-items`, the "Work board"). |
| **Obligation / filing** | What the law requires of a client company and by when, such as GSTR-3B for August due 20 Sept. **Generated, never typed in.** |
| **Compliance rule** | The due-date formula behind obligations. It must be confirmed and linked to a service in the Catalogue before it generates anything. |
| **Statutory vs internal date** | The law's deadline vs the firm's earlier target. Lists sort by the internal one. |
| **Advance** | Money collected before work starts. If one is asked, work waits until it's recorded. |
| **Dunning / chasing** | Reminders about unpaid invoices. |
| **SPOC / Key Manager / Dept** | The client's point of contact per department, its overall manager, and the firm's departments. These decide whose desk work lands on. |

---

## Making a change

1. **Branch** from `main`: `feat/…` or `fix/…`. Never commit straight to `main`.
2. **Backend API changed?** Run `npm run api:generate` in the frontend and commit the
   regenerated types alongside your frontend change.
3. **New migration?** Take the next free number, check nobody else has claimed it (two
   people working in parallel take separate number ranges), and include the `down`.
4. **Run the gates** before pushing:
   - backend: `npm run check`
   - frontend: `npm run check` and `npm run build`

   A husky pre-push hook runs `npm run check` using your shell's **default** Node. If that's
   Node 20, backend tests fail. Run `nvm use` first, or push with Node 22 on your `PATH`.
5. **Open a PR** and go through `bib-backend/CONTRIBUTING.md`'s checklist (boundaries,
   tenancy, migrations, money).
6. **When a rule seems wrong, say so in the PR.** Don't silence it with `eslint-disable` or an
   arch exception.

---

## Environments and deploys

- **QA** is the only deployed environment: Google Cloud project `bib-qa`, Cloud Run in
  `asia-south1`. It has four workloads: API, worker, migrate job and frontend. The database
  is Cloud SQL on a private IP only, so you can't connect from a laptop.
- **Deploys are manual** with `gcloud`, in a fixed order: build images, run migrations, then
  API, worker, frontend. Follow `bib-docs/backend/DEPLOY.md` exactly. Infrastructure is Terraform
  in `bib-backend/infra/terraform`.
- `gcloud builds submit` uploads your **working folder**, not the commit, so deploy from a
  clean tree.
- After a deploy, run `scripts/postdeploy-check.sh` (read-only). Never run smoke against QA.

---

## Gotchas that have cost people time

- **pg-boss missing under Node 20.** See Prerequisites: switch to Node 22 and run
  `npm install` again.
- **Real emails from local runs.** See [Keep local runs from emailing real people](#keep-local-runs-from-emailing-real-people).
- **Frontend: a Tailwind class on an element that has a design-system class may silently do
  nothing.** Design-system CSS wins on every property it declares.
- **Frontend: `.row` spreads children to opposite edges.** Use `.cluster` for a row of
  buttons. `.gap` doesn't exist.
- **Frontend: requests appear twice in dev.** React StrictMode runs effects twice; production
  doesn't.
- **Typed-route errors after a merge** are usually stale `.next/types`. `npx next typegen` or
  a build fixes them.
- **An empty compliance calendar is usually correct.** The calendar's second tab says which of
  these is missing: a confirmed rule, a Catalogue link, client details (country, kind, year
  end), a signed letter, or a run of `worker:once`.
- **Rate limit:** 300 requests a minute per session. Long scripted walks can hit `429`, so
  retry after a pause before deciding something is broken.

---

## Where to read next

| Doc | Why |
|---|---|
| `bib-backend/CLAUDE.md` | The backend's rules and the reasoning behind every module. It's long (5k+ lines), so skim the `##` headings and read the section for the area you're touching. |
| `bib-frontend/CLAUDE.md` | The same for the frontend: API, session, styling, every screen's decisions. |
| `bib-frontend/src/design/README.md` | The design system. Read it before building any UI. |
| `bib-backend/CONTRIBUTING.md` | The PR checklist. |
| `bib-docs/backend/HANDOFF.md` | The founding decisions (tenancy model, ADRs). |
| `bib-docs/backend/DEPLOY.md` | Deploying, step by step, and known failures. |
| `bib-docs/backend/MERGE_BUILD-PLAN.md` | The live task board. `bib-docs/backend/BUILD-PLAN.md` is superseded. |
| `bib-backend/docs/EVENT-CONTRACT.md` | The ledger's event types. |

Good first exercise: seed the demo, sign in as Siddharth, open a client, send a proposal,
accept it from the logged email, write up and sign the letter, run `npm run worker:once`, and
watch the jobs, tasks and calendar entries appear. That one walk passes through most of the
codebase.
