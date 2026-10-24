# Global search — plan and tracker

Started 2026-10-24. One search box, opened with **⌘K / Ctrl+K** from any staff screen, that
finds pages, actions and records across the app.

## Decisions (and why)

| Decision | Why |
|---|---|
| **Postgres, not Algolia / Elasticsearch** | A firm has thousands of records, not millions. Search stays inside the tenant wall: RLS applies to every query. A copy in a search engine would need the wall rebuilt by hand, and client names / PAN / GSTIN would leave our infrastructure. |
| **Behind a `Searcher` port** (`usecase/search/ports.ts`) | If Postgres is ever outgrown, a Typesense / OpenSearch adapter replaces the Postgres one. The API and the screens do not change. |
| **One query per record type**, not a copied search table | Nothing to keep in sync, and each type's permission filters stay the same as its own list screen. |
| **Staff only** | The portal gets its own search later, behind gate 4. |
| **No fee amounts in results** | Fee visibility is its own axis. Leaving amounts out means a result line can never leak them. |
| **Fixed group order**: clients, proposals, engagement letters, invoices | Predictable. People learn where to look. |

## The API contract

`GET /api/v1/search?q=<text>` (staff audience, signed in)

- `q` is trimmed. 2–80 characters after trimming, else `400` with the house error envelope.
- Response body (inside the usual envelope):

```jsonc
{
  "query": "sharma",
  "groups": [
    {
      "kind": "client",            // "client" | "proposal" | "engagement_letter" | "invoice"
      "items": [
        {
          "kind": "client",
          "id": "uuid",
          "title": "Sharma & Co",                 // name, or the document number
          "subtitle": "GSTIN 27AAACS1234F1Z5",    // short context; null when there is none
          "match": "identifier"                   // "identifier" | "prefix" | "fuzzy"
        }
      ]
    }
  ]
}
```

- Only groups with at least one item are returned, in the fixed order above.
- A group is searched only if the caller may see that list at all (same role / capability
  check as the list endpoint). Inside it, the same record-scope rules as the list (client
  assignment, hidden "deleted by mistake" records excluded).
- At most **5 items per group**.
- The frontend maps `kind` + `id` to a route: `client → /clients/{id}`,
  `proposal → /proposals/{id}`, `engagement_letter → /engagement-letters/{id}`,
  `invoice → /invoices/{id}`. The backend never knows frontend routes.

### What each type matches on

| Kind | Matches on |
|---|---|
| client | display name; PAN / GSTIN and other identifiers (exact or prefix, case-insensitive) |
| proposal | proposal number; client name |
| engagement_letter | letter number; client name |
| invoice | invoice number; client name |

### Ranking (inside a group)

1. exact identifier / number match
2. identifier / number / name **starts with** the query
3. name **contains** the query, or a close spelling (trigram similarity, so `sharam` finds `Sharma`)

Ties go to the most recently updated record. LIKE wildcards in `q` (`%`, `_`, `\`) are escaped.

## Guards (so search cannot hurt the rest of the app)

| Guard | Where |
|---|---|
| Btree indexes on `(tenant_id, column COLLATE "C")` for identifiers and document numbers, used through the leakproof "starts with" operator `^@`; 5 results per group | migration `0089`, adapter |
| A 2 s statement timeout inside the search transaction (`set_config('statement_timeout', '2000ms', true)`, the same as `SET LOCAL`; the app default stays 30 s) | adapter |
| All four groups are branches of **one** SQL statement in one transaction on one connection: one pool slot per search | adapter |
| When the browser cancels, the API cancels the running query | handler (`c.req.raw.signal`) + postgres.js `cancel()`. Still to confirm on Cloud Run; the 2 s timeout covers it if the cancel does not get through |
| Rate limit | already in place: every signed-in `/api/v1` request counts against the per-session bucket (`router.ts:472`) |
| Browser: wait 200 ms after typing stops, at least 2 characters, cancel stale requests, remember recent answers | frontend |

## What the build found (2026-10-24)

- **A trigram index can never serve a search under row-level security.** Postgres applies
  a condition through an index before the tenant policy only if its operator is
  "leakproof". `ILIKE`, pg_trgm's `%` / `<%` and every citext operator are not, so the app
  role always scans the tenant's rows for them. The client-name index from 0002
  (`idx_party_name_trgm`) has never been used. Checked as `bib_app` with sequential scans
  disabled.
- **So 0089 adds btrees instead**, and "starts with" is written as `^@` (leakproof) on the
  stored forms of identifiers and numbers. "Contains" and close spellings scan the
  tenant's own rows: about 17 ms at 5,000 clients, growing linearly (about 70 ms at 20,000).
- **Close spelling** is `strict_word_similarity >= 0.4`: "sharam" finds Sharma, "guptha"
  finds Gupta, "kapor" finds Kapoor. A one-letter change in a five-letter name (mehra / Mehta)
  is missed. This threshold is a judgement call.
- **Measured** (3 firms × 5,000 clients, 45,000 documents, PG 18 locally): a whole search
  takes 16–40 ms in Postgres and 19–45 ms through the API. QA runs PG 17; re-check the plans there.
- **Scoping as built**: all four groups need `party:read`, the same permission as their
  lists. None of the four lists scopes by client assignment, so search does not either.
  Clients exclude archived clients and the firm's own companies; proposals and letters
  exclude deleted ones; invoices include drafts (titled "Draft invoice") and cancelled.
- **Cancel** works locally: an aborted request leaves no statement running. It is still
  unconfirmed on Cloud Run.

## Phases

**Redesign, 2026-10-24 (FE `7328dc3`):** at the user's request the modal palette became an
inline pill in the top bar that expands in place into the results panel: no dialog, no
backdrop. Clicking anywhere on the pill or ⌘K / Ctrl+K puts the caret straight in. It never
autofocuses on page load. cmdk was replaced by a small hand-written combobox so the pill can
live in the shell; the panel still loads lazily (+1.4 KB per route). Walk 26/26.

### Phase 1 — ⌘K palette: pages, actions, recents (frontend only)
Done in FE `f89f9e6`:
- [x] ⌘K / Ctrl+K opens the palette on every staff screen; a visible "Search ⌘K" button in the top bar opens it too
- [x] Pages: every link in the sidebar map and the settings map, filtered by the same `visible()` role check (the maps moved to `nav-map.ts`)
- [x] Actions: New proposal, New engagement letter, Agreed by email, Import from Excel, Add a service, each with its button's own check
- [x] Recents: palette picks, kept in the browser per user + firm, at most 8
- [x] Not shown in the client portal or on signed-out screens
- [x] The palette's inside loads lazily: +1.2 KB per route instead of +27 KB

### Phase 2 — record search
Backend (`worktrees/search-be`, branch `feat/global-search`, commits `e5acb58`…`0d8fb00`)
- [x] `usecase/search`: ports + service (validates `q`, decides which groups the caller may search)
- [x] `infrastructure/postgres/searcher.ts`: one statement, one transaction, 2 s timeout, cancellable
- [x] `GET /api/v1/search` + zod/OpenAPI schema, wired in `setup/`
- [x] Migration `0089_global_search_indexes` (up + down)
- [x] Unit tests + 22 integration tests: ranking, typo match, identifier match, **tenant isolation**, role gating, deleted / archived / own-company excluded, no amounts in any result, timeout, cancel, literal wildcards
- [x] `npm run check` (3,963 tests) + `npm run test:integration` (1,263 tests) green

Frontend (`worktrees/search-fe`, branch `feat/global-search`, commits `7fc1974`, `5c1bd85`)
- [x] Types generated from the backend branch (only the search additions changed); the call uses the generated client
- [x] Record groups in the palette under the pages / actions; cmdk's own filtering off
- [x] 200 ms debounce, 2-character minimum, AbortController, in-memory cache of the last 20 answers (per login + firm, cleared on firm switch and sign-out)
- [x] Loading, empty, too-short and timed-out states
- [x] Sign-out clears this login's palette recents
- [x] `npm run check` + `npm run build` green; browser walk 29/29 (fast typing of 14 keys sent 1 request; a repeat search sent 0)

### Phase 3 — later, only when needed
- Leads, work items, staff, client requests, payments, compliance filings (about half a day each)
- Document file names; then text inside documents (`files.document_extraction`)
- Portal search (behind gate 4)
- If name search gets slow for very large firms (it scans the firm's own rows, see "What the build found"): a stored, normalised name column with a btree for "starts with", or a search engine. A trigram or `btree_gin` index will not help under RLS.
- A dedicated engine, only if Postgres measurably cannot keep up

## Separate from search, before production
The API can scale to 10 instances × a 10-connection pool = 100 connections, which is about
Cloud SQL's default limit for the current instance size (`db-custom-1-3840`). The worker's
connections come on top. Fix with a smaller pool per instance or Cloud SQL managed
connection pooling. Not part of this build.

## Rollout
1. Backend PR first (migration `0089` + route), then frontend PR.
2. Both go through the CI gate and the "production" environment approval, backend first.
3. On QA: try ⌘K, a misspelled client name, a GSTIN prefix, an invoice number; confirm the cancel reaches Cloud Run.
