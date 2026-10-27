# zod 3 → zod 4: the migration plan, and what the advisory reaches today

Written 2026-10-28. Planning only: nothing in `bib-backend` or `bib-frontend` was changed.

Everything below was read from, or measured on, a scratch copy of **bib-backend `main` at
`001ec90`** and **bib-frontend `main` at `5fc9d2e`**. "Measured" means a real run on an M1 laptop
(8 GB), through the real router (`buildRouter` with the in-memory fakes that
`tests/api/openapi.test.ts:781-797` uses), unless it says otherwise. Cloud Run's API instance is
1 vCPU and 1 GiB (`infra/terraform/variables.tf:259-276`), so expect it to be slower, not faster.

---

## 1. The answer first

**Is BiB exposed today? Yes, to any signed-in staff member, and not to anyone else.** None of
the 9 public routes, the 18 routes a client-portal login can reach or the 6 `/internal` routes
takes an array or a record in its body or query, so nothing they parse can multiply. 21 staff
routes can. The worst needs no permission at all: one 256 KB `PUT /api/v1/notifications/preferences`
from any staff login costs about 1.2 s of CPU, takes the heap from about 90 MB to about 550 MB and
answers with a 9 MB error. Three at once took the heap to 1.3 GB, which is more than an API instance has.

**A worse hole sits beside it, and it is not zod.** A JSON body sent to one of the four upload
paths is held to the 25 MB upload limit. `boundedStrings()` then parses it before the session
guard and the rate limiter run. One **anonymous** 24 MB JSON body to `POST /api/v1/documents`
took 2.9 s and 574 MB of heap before it got its 401.

**Ship this before the migration (half a day, no dependency change):**

1. **Body limit by content type, not just path.** The upload limit applies only when the path
   is an upload path **and** the body is `multipart/form-data`. Every other body is held to the
   256 KB JSON limit (`src/api/router.ts:448-449`).
2. **A count ceiling in the walk that already runs.** In `firstOffence` (`src/api/bounds.ts:96-122`),
   refuse a body with a list over 1,000 items, an object over 1,000 fields, or more than
   10,000 list items in total. The answer is a 400 `VALIDATION`, sent before zod sees the body.
3. **Put the first 20 issues in `reason`, not all of them** (`src/api/openapi.ts:51-53`), and
   say how many more there were.

I built all three in a scratch copy. The same 256 KB attack then got its 400 in 31 ms, with 9 MB
of heap and a 95-byte answer. The anonymous 24 MB JSON got a 413 in 31 ms. The full unit suite
still passed (§2.6).

**Then migrate.** It is small. Bumping to zod 4.6.7 and @hono/zod-openapi 1.6.3 gave
**10 compile errors** (§4.1). With those fixed, `tsc`, `eslint` and all **4,049 unit tests
passed on Node 22**. The OpenAPI spec changed only in how it writes things, so the frontend's
generated types are the same (§5). The upgrade fixes the advisory only for arrays that carry a
`.max()`. Six request arrays have none, and on zod 4 a large one turns into a **500** (§2.4). So the
migration has to add those six `.max()`s.

**Size:** about 2 days in total: half a day for the mitigation and about 1.5 days for the
migration, including verification. Detail in §6.

---

## 2. The advisory

### 2.1 What it is

| | |
|---|---|
| Advisory | GHSA-mrhx-c7xc-m3w5 / CVE-2023-54404, "Zod can exhaust memory while validating oversized constrained collections", high (CVSS 3.1 7.5), CWE-770 |
| Root report | colinhacks/zod#1872, filed 2023-01-12 **against zod 3.20.2**: `z.number().max(1).array()` given 1,000,000 elements built 1,000,000 issues in about 3 s, and 10,000,000 elements ran out of memory |
| Fix | colinhacks/zod#6652, merged 2026-10-09, released in 4.6.6. `.max()`, `.length()` and `.size()` on arrays, sets and maps now refuse an oversized input **before any element is parsed**. "Unbounded containers are unchanged." Records are not covered. |
| Not merged | colinhacks/zod#6475 (an `abortEarly` parse option) was closed unmerged |
| zod 3 | No fix. npm's range `<4.6.6` covers every 3.x |

`npm audit --omit=dev` lists three high entries, but they are one issue. zod is the vulnerable
package. `@hono/zod-openapi <=0.19.10` and `@asteasolutions/zod-to-openapi <=8.0.0-beta.4` are
flagged only because they need zod 3. Audit's suggested fix is `@hono/zod-openapi@1.6.3`, a major bump.

### 2.2 Why BiB is exposed on zod 3

- zod 3's array does check `.max()` first, but it only marks the result dirty and then walks
  every element anyway (`node_modules/zod/v3/types.js:1788-1805`). So the `.max(54)` on
  notification settings bounds nothing.
- `@hono/zod-validator` calls `schema.safeParseAsync` (`node_modules/@hono/zod-validator/dist/index.js:12`).
  On zod 3 that builds one promise per element, even for elements that are valid (`types.js:1801-1805`).
- The shared hook joins **every** issue into the `reason` string (`src/api/openapi.ts:51-53`),
  so the error response grows with the attack.

Every handler gets `z` from one seam (`src/api/openapi.ts:9`, re-exported at `:14`). All 44 files in `src/api/` (42 handlers, `router.ts`, `paging.ts`)
import it from there, and only `src/config/config.ts:12` imports `zod` directly.

### 2.3 Who can reach it

I walked every request schema of all 373 registered operations (body, form and query) and found
each array or record a caller controls the length of.

| Surface | Operations | Arrays / records in the request | Can multiply? |
|---|---|---|---|
| Public, no session (`src/api/guards.ts:92-189`) | 9 | none | no |
| Client portal (`PORTAL_PATHS`, `src/api/guards.ts:246`) | 18 | none | no |
| `/internal` (service key) | 6 | none | no |
| Staff session | 340 | 23 arrays + 7 records, on 26 routes | **21 routes** |

An object without arrays costs a bounded amount whatever the caller sends: zod 3's default
`strip` mode does not even loop over unknown keys. The 21 routes:

| Route (under `/api/v1`) | Field | Bound today | Permission | Issues a 256 KB body builds (estimate) |
|---|---|---|---|---|
| `PUT /notifications/preferences` | `settings` | .max(54) | **none, any staff login** (`src/api/permissions.ts:73`) | 262,582 |
| `PUT /notifications/firm-defaults` | `settings` | .max(54) | `tenant:admin` | 262,582 |
| `PUT /engagement-letters/{letterId}/advances-received` | `advances` | .max(20) | `party:write` | 175,492 |
| `PUT /billing/settings` | `taxDefaults` | .max(50) | `tenant:admin` | 175,492 |
| `POST /advances/{advanceId}/allocations` | `targets` | .max(20) | `advance:manage` | 175,054 |
| `POST /billing/raise` | `lines` | **none** (`src/api/handler/billing.ts:185`) | `party:write` | 175,054 |
| `POST /billing/raise-batch` | `cards`, `cards[].lines` | .max(50), **none** | `party:write` | 175,054 |
| `POST /leads` | `identifiers` | **none** (`src/api/handler/lead.ts:289-292`) | `party:write` | 174,617 |
| `POST /tenancy/members` | `deptIds` | .max(50) | `tenant:admin` | 132,058 |
| `POST /obligation-rules` | `serviceEntryIds` | .max(50) | `tenant:admin` | 132,058 |
| `PATCH /parties/{partyId}` | `billingAddress.lines` | .max(8) | `party:write` | 132,058 |
| `POST /firm-entities`, `PATCH /firm-entities/{entityPartyId}` | `registeredAddress.lines` | .max(8) | `tenant:admin` | 132,058 |
| `POST /invoices/{invoiceId}/email` | `to`, `cc` | .max(10) | `party:write` | 132,058 |
| `PUT /tenancy/mail-senders` | `sendFromDomains` | .max(50) | `tenant:admin` | 132,058 |
| `PUT /tenancy/members/{membershipId}/depts` | `deptIds` | **none** (`src/api/handler/tenancy.ts:222`, used at `:374`) | `tenant:admin` | 131,401 |
| `POST /invoices/issue-batch` | `invoiceIds` | .max(50) | `invoice:issue` | 131,401 |
| `POST /billing/raise`, `/billing/raise-batch` | `dueIds` | .max(50) | `party:write` | 131,401 |
| `POST /gst-certificate-readings/{readingId}/apply` | `fields` | **none** (`src/api/handler/gstcert.ts:137`) | `party:write` | 130,743 |
| `POST /trade-licence-readings/{readingId}/apply` | `fields` | **none** (`src/api/handler/tradelicence.ts:148`) | `party:write` | 130,743 |
| `POST /work-items/{workItemId}/state`, `/work-item-checks/{approvalId}/decide` | `filedOn` (record) | **none** (`src/api/handler/workitem.ts:521`, `:580`) | `work:write` | 62,083 |

The estimate comes from the element shape that builds the most issues per byte, measured on 200
elements and scaled to 256 KB. Five more records (`config.ts:80`, `:122`, `party.ts:132`, `:167`,
`orchestrator.ts:122`) hold `z.unknown()` values, which never fail, so they build nothing.

### 2.4 Measured cost

**Today, zod 3.25.76:** `PUT /api/v1/notifications/preferences` with body `{"settings":[{},{},…]}`,
which is 87,314 empty objects in 262 KB, sent with a staff session through the real router.

| Requests at once | Answer | Time | Heap | Response |
|---|---|---|---|---|
| 1 | 400 | 1.2–1.3 s | ~90 → ~530–560 MB | 9.0 MB |
| 3 | 400 ×3 | 5.3 s | peak 1.3 GB | 9.0 MB each |

The cost grows with the element count (one request each, same route):

| Elements | Body | Time | Heap after | Response |
|---|---|---|---|---|
| 1,000 | 3 KB | 20 ms | +16 MB | 97 KB |
| 5,000 | 15 KB | 66 ms | +29 MB | 0.5 MB |
| 10,000 | 30 KB | 111 ms | +61 MB | 1.0 MB |
| 87,000 | 261 KB | 3.1 s | +426 MB | 8.9 MB |

The 87,000 row ran after five others in the same process. On its own it took 1.2–1.3 s, as in
the first table.

The general rate limit is 300 requests a minute per session (`src/config/config.ts:250`), and the
API runs on one vCPU. One staff session can therefore keep an instance busy, and a few
concurrent requests can push it out of memory.

**After the upgrade, zod 4.6.7** (scratch copy):

| Case | Result |
|---|---|
| The same 87k body to the bounded `settings` (`.max(54)`, `src/api/handler/notification.ts:202-205`) | 400 in **43 ms**, +16 MB, a 133-byte answer: `settings: Too big: expected array to have <=54 items` |
| 87k `{}` into the unbounded `billing/raise` `lines`, through `safeParseAsync` | **throws `RangeError: Maximum call stack size exceeded`** |
| The same through sync `safeParse` | 174,721 issues, +190 MB |
| The smallest unbounded array that throws | 64,000 objects (`billing/raise` `lines`, `leads` `identifiers`), 128,000 numbers (`gst-certificate-readings/…/apply` `fields`) |

The validator uses `safeParseAsync`, so on zod 4 an unbounded array that large is not a slow 400.
It is a thrown error, and `app.onError` (`src/api/router.ts:280-286`) maps it to a **500 with an
`unhandled error` log line**. **The upgrade alone is therefore not enough. The six unbounded
arrays need a `.max()`.**

### 2.5 The pre-auth hole beside it (not zod)

- The body limit picks the 25 MB upload allowance by **path alone** (`src/api/router.ts:448-449`,
  paths at `:142-148`, allowance `src/config/config.ts:267` plus 64 KB at `router.ts:132`).
- `boundedStrings()` parses any `application/json` body (`src/api/bounds.ts:66-80`).
- It runs before the CSRF origin check, the rate limiter and both guards
  (`router.ts:450`, `:455`, `:460`, `:470`, `:478-479`).

Measured, with no credentials:

| Request | Answer | Time | Peak heap |
|---|---|---|---|
| 24 MB JSON of `{}` to `POST /api/v1/documents` | 401 | 2.9 s | +574 MB |
| 24 MB JSON of `0` to `POST /api/v1/documents` | 401 | 0.4 s | +135 MB |
| 24 MB JSON to `POST /api/v1/leads` (control) | 413 | 51 ms | +30 MB |

Nobody needs to sign in for this, and no rate limit applies. Two such requests at once exceed an
instance's memory.

**Found while checking this, and the opposite problem.** Four multipart routes are not on the
upload list, so they get the 256 KB JSON limit:

- `POST /api/v1/gst-certificates/read` and `POST /api/v1/parties/{partyId}/gst-certificate`
  (`src/api/handler/gstcert.ts:161`, `:182`, multipart at `:168`, `:191`)
- `POST /api/v1/trade-licences/read` and `POST /api/v1/parties/{partyId}/trade-licence`
  (`src/api/handler/tradelicence.ts:176`, `:197`, multipart at `:183`, `:205`)

Their services accept files up to 20 MB (`src/usecase/gstcert/service.ts:85`,
`src/usecase/tradelicence/service.ts:85`). Measured: a 300 KB certificate gets a 413 at the edge,
and a 100 KB one gets through. Mitigation #1 below adds these four paths to `UPLOAD_PATHS`.

### 2.6 The mitigation, in detail

| # | Change | Where | What it costs a real request |
|---|---|---|---|
| 1 | Upload limit only for `multipart/form-data` on an upload path; every other body gets `jsonLimit`. Add the four certificate and licence paths above to `UPLOAD_PATHS` | `src/api/router.ts:448-449`, `:142-148` | nothing: every upload route is multipart (`filestore.ts:288`, `:302`, `clientimport.ts:353`, `portalask.ts:148`, plus the four above) |
| 2 | In `firstOffence`, refuse a list over **1,000** items, an object over **1,000** fields, or more than **10,000** list items in the whole body, with a 400 `VALIDATION` naming which | `src/api/bounds.ts:96-122` (it already visits every value, so this is a counter) | nothing legitimate (see below) |
| 3 | `reason` carries the first **20** issues, then "and N more" | `src/api/openapi.ts:51-53` | bounds the reply. It does not reduce what zod allocates, which is what #2 is for |

Why those numbers:

- The largest bounded request list is 54 (`notification.ts:205`).
- The largest list cap in the config validators is 40 checklist lines (`src/domain/config/model.ts:2642`).
- The heaviest legitimate body is a raise-batch at its schema maximum: 50 cards
  (`src/usecase/billing/to-raise.ts:30`), each with 50 dues (`billing.ts:184`) and 50 lines.
  That is 266 KB, so it is **already** refused by the 256 KB JSON limit (`router.ts:127`).
- At 10,000 empty objects zod 3 costs 111 ms and 61 MB (§2.4), about a tenth of today's worst case.
  At 1,000 it costs 20 ms.

One check before choosing the number: the config registry's `body` is `z.record(z.string(), z.unknown())`
(`src/api/handler/config.ts:80`), and its validators in `src/domain/config/model.ts` are not zod.
Count the values in the largest stored body on the demo database. QA was wiped on 2026-10-21, so
QA would say little.

```sql
SELECT e.kind, e.key, v.version,
       (SELECT count(*) FROM jsonb_path_query(v.body, 'strict $.**')) AS json_values
FROM config.version v JOIN config.entry e ON e.id = v.entry_id
ORDER BY json_values DESC LIMIT 5;
```

**Scratch prototype, all three changes, zod 3, Node 22:**

| Request | Before | After |
|---|---|---|
| 87k `settings`, staff | 400, 1.2 s, +450 MB, 9 MB answer | 400, **31 ms**, +9 MB, 95 bytes ("a list in the request body has more than 1000 items") |
| 1,000 `settings`, staff | 400, 20 ms, 97 KB answer | 400, 36 ms, **723-byte** answer |
| 24 MB anonymous JSON to `/documents` | 401 after 2.9 s, +574 MB | **413 in 31 ms** |
| Full unit suite (Node 22) | 4,049 pass | **4,049 pass** |

The prototype did not include the four extra upload paths; that is a one-line addition.

Tests to add with it, in `tests/api/`:

- JSON to each upload path answers 413.
- A 300 KB multipart certificate is not refused at the edge.
- A list of 1,001 answers 400 before validation.
- 10,001 items spread over two lists answer 400.
- A 30-issue validation failure keeps 20 issues in `reason`.

---

## 3. Target versions

| Package | Now | Target | Published | Requires |
|---|---|---|---|---|
| `zod` | 3.25.76 (`^3.24.1`) | **4.6.7** | 2026-10-12 | none |
| `@hono/zod-openapi` | 0.19.10 | **1.6.3** | 2026-09-04 | peer `zod ^4.0.0`, `hono >=4.10.0` |
| `@asteasolutions/zod-to-openapi` (transitive) | 7.3.4 | **9.1.0** | 2026-07-19 | peer `zod ^4.0.0` |
| `@hono/zod-validator` (transitive) | 0.7.6 | 0.9.1 | | peer `zod ^3.25 \|\| ^4`, `hono >=4.11.2` |
| `hono` | 4.13.13 | unchanged | | meets both peers |
| `@anthropic-ai/sdk` | 0.128.0 | unchanged | | peer `zod ^3.25 \|\| ^4`. BiB uses none of its zod helpers |

- **zod-to-openapi goes to 9.x, not 8.x.** @hono/zod-openapi 1.6.0 moved to v9 for the HTTP QUERY
  method. v8.0.0 is where zod 4 support arrived. v9's notes say runtime output for the 3.0 and 3.1
  generators is unchanged and its breaking changes are TypeScript-only. BiB generates 3.0
  (`router.ts:530-531`).
- **@hono/zod-openapi 1.x is zod 4 only.** Its peer is `zod ^4.0.0`, and the 1.0.0 entry in its
  changelog says it "migrated the Zod version from v3 to v4". 0.19.x is zod 3.
- **Can v3 and v4 coexist?** Yes as packages: zod 3.25 already ships zod 4 at `zod/v4`, and zod 4
  ships zod 3 at `zod/v3`. No as route schemas: there is one `OpenAPIHono` and one `z`
  (`src/api/openapi.ts:9`, re-exported at `:14`), and a schema built with the other major is not validated by the
  other's validator. The API layer moves in one commit. `src/config/config.ts` could stay on
  `zod/v3`, but it needs a single change (§4.1), so there is no point.
- **npm refuses the bump in place** (`ERESOLVE`), because the locked zod-to-openapi 7.3.4 has peer
  `zod ^3.20.2`, even with both packages named on one command line. What worked:
  1. Set `zod: ^4.6.7` and `@hono/zod-openapi: ^1.6.3` in `package.json`.
  2. Delete the lock entries for `zod`, `@hono/zod-openapi`, `@hono/zod-validator`,
     `@asteasolutions/zod-to-openapi` and `openapi3-ts`.
  3. Run `npm install`.
  The lock diff is six packages: the four in the table, plus `openapi3-ts` 4.6.1 → 4.6.2 and its
  `yaml` 2.9.0 → 2.9.1. After that, `npm audit --omit=dev` showed **only** nodemailer, which is the
  other agent's work.
- Type-checking got cheaper: instantiations went from 5.67M to 2.49M and check time from 48 s to
  36 s (one `--extendedDiagnostics` run each).

---

## 4. Breaking changes that hit this codebase

Counts are over `src/` with comments stripped, at `001ec90`. "Measured" means I ran that case on
both zod 3.25.76 and zod 4.6.7.

### 4.1 What the compiler catches (10 errors in total)

| Change | Sites | Where | Fix |
|---|---|---|---|
| `z.record(value)` single-argument form removed | **9** | `src/api/handler/config.ts:80`, `:122`; `ledger.ts:48`; `orchestrator.ts:122`, `:401`; `party.ts:132`, `:167`, `:295`, `:325` | `z.record(z.string(), z.unknown())`. The spec does not change |
| `required_error` dropped | **1** | `src/config/config.ts:33` | `error: (iss) => iss.input === undefined ? "BIB_ENVIRONMENT is not set. …" : undefined`. **Not** `error: "…"`: a plain string also replaces the wrong-value message, so `BIB_ENVIRONMENT=prod` would say "is not set" (measured) |

After these ten: `tsc` 0 errors, `eslint src tests` 0, unit 4,049 / 4,049 on Node 22.

### 4.2 What the compiler does not catch

| Change | Sites | Where | Effect here | Action |
|---|---|---|---|---|
| A `z.any()` / `z.unknown()` key is now **required** at parse time (since zod 4.4.0) | **6** object keys | `filestore.ts:288`, `:302`; `gstcert.ts:132`; `tradelicence.ts:143`; `clientimport.ts:353`; `portalask.ts:148` | A form without its file gets zod's 400 (`file: Invalid input: expected nonoptional, received undefined`) instead of the handler's own refusal (`filestore.ts:781-785`, and `ClientSheetRequired` for the import). The spec already marks none of these parts required, on both versions. **No unit test covers a missing file**, which is why all 4,049 passed | `.optional()` on each part, plus one test per upload route that a missing part still gets the handler's own answer |
| `z.string().uuid()` is RFC 9562 strict (version 1–8, variant 8–b) | **723** in 39 files | e.g. `src/api/handler/acceptance.ts:25` | Real ids are fine: `randomUUID` v4 (`src/platform/ids.ts:20`), uuid v5 (`:21`), `gen_random_uuid()`; there are no uuid literals in `migrations/`; all 127,307 ids in `demo-assets` pass. **One breaks:** `scripts/smoke.sh:796` sends `00000000-0000-0000-0000-0000000000ff` and expects 404 at `:802`, but gets 400. `DEV_SESSION_ID` (`src/api/principal.ts:58`) is not checked by zod. `tests/` hold 309 non-RFC literals (61 distinct), and none reaches a `.uuid()` over HTTP | Change the smoke id to `00000000-0000-4000-8000-0000000000ff`. Keep `.uuid()`; do not switch to `z.guid()` |
| Unbounded arrays throw on the async path (§2.4) | **6** fields, 5 definitions | `billing.ts:185` (also `cards[].lines`, the same schema); `lead.ts:289`; `tenancy.ts:222`; `gstcert.ts:137`; `tradelicence.ts:148` | A 500 plus an error log instead of a 400 | `.max()` on each (§6, step 3). `tenancy.ts:222` is also a response schema, so the spec gains a `maxItems` there |
| A body whose Content-Type matches none of the route's declared media types, or a body with no Content-Type, gets **415** (@hono/zod-openapi 1.6.3, honojs/middleware#2087). Before, it was validated as `{}` | every body route | the error is an `HTTPException(415)`, which `fromHttpException` maps to **400 `VALIDATION` "Unsupported Media Type"** (`src/api/domainerr.ts:1499-1522`) | The frontend sends JSON or `FormData` through openapi-fetch. Its raw `fetch`es are GETs (`bib-frontend` `src/lib/api/client.ts:1644`, `:1686`; `src/app/admin/brand/page.tsx:336`). The scripts send `-H "$H"` (`scripts/smoke.sh:51`) | Low risk. The smoke run and the upload walk cover it |
| Default error messages reworded | all | the `reason` string only (`openapi.ts:51-53`), and boot errors from `config.ts:474` | e.g. `Required` becomes `Invalid input: expected string, received undefined`; an enum now lists every option. The frontend shows `message`, never `reason`'s text (`bib-frontend` `src/app/catalogue/shared.tsx:100-117`). It compares `reason` only to the domain code `"PROSE_EDITED"` (`src/components/proposal-builder/client-step.tsx:588`, `src/components/letter-builder/client-step.tsx:319`). The one test on `reason`, `toContain("slug")` (`tests/api/openapi.test.ts:980`), still passes. The comment at `bib-frontend` `src/app/admin/obligation-rules/rule-editor.tsx:200` quotes the old sentence | None. Mention the wording change in the PR |
| `.int()` limited to ±(2^53−1) | **364** | e.g. `src/api/handler/advance.ts:106` | zod 3 accepted 2^60 (measured). No BiB integer comes near it; money travels as decimal strings (`src/api/handler/engagement.ts:80`) | None |
| `z.number()` refuses ±Infinity | 347 | | JSON cannot carry Infinity, and `?limit=Infinity` already failed `.int()` on zod 3 | None |

Before and after wording, measured:

| Input | zod 3 | zod 4 |
|---|---|---|
| missing string | `a: Required` | `a: Invalid input: expected string, received undefined` |
| `""` with `.min(1)` | `String must contain at least 1 character(s)` | `Too small: expected string to have >=1 characters` |
| 5 items, `.max(3)` | `Array must contain at most 3 element(s)` | `Too big: expected array to have <=3 items` |
| bad enum | `Invalid enum value. Expected 'x' \| 'y', received 'z'` | `Invalid option: expected one of "x"\|"y"` |
| bad uuid / email | `Invalid uuid` / `Invalid email` | `Invalid UUID` / `Invalid email address` |
| `1.5` with `.int()` | `Expected integer, received float` | `Invalid input: expected int, received number` |

### 4.3 Checked, and no change needed

| Change in zod 4 | Sites here | Why it does not bite |
|---|---|---|
| `.strict()` / `.passthrough()` / `.nonstrict()` / `.strip()` / `.catchall()` | **0** | none used |
| `errorMap` → `error`, `invalid_type_error` dropped | **0** | none used |
| `message` param deprecated (still accepted) | 11 refine options (`dashboard.ts:526`, `:534`, `:633`, `:641`, `:749`, `:757`; `document.ts:566`; `portalhome.ts:191`; `work-billing.ts:28`, `:243`, `:250`), `config.ts:293`, and 5 string shorthands (`config.ts:238`, `:243`, `:288`, `:612`, `:616`) | still work; renaming them is cosmetic |
| `.email()` / `.uuid()` / `.url()` / `.datetime()` / `.date()` become top-level formats | 16 / 723 / 2 / 10 / 11 | method forms still work. The email regex was rewritten but accepts the same addresses (`zod/v3/types.js:384`, `zod/v4/core/regexes.js:34`). Offsets and impossible dates are refused exactly as before (measured) |
| `.default()` no longer parses its value; defaults now apply inside `.optional()` | 16 in handlers (e.g. `invoice.ts:1074`, `proposal.ts:879`, `tenancy.ts:326`) + 37 in `config.ts` | every default is already the output type, none comes after a `.transform()`, and none is wrapped in `.optional()`, the one case whose result changes. `.prefault()` is not needed |
| `ZodError` `.flatten()` / `.format()` / `.formErrors` / `.errors` / `.addIssue()` | **0** | only `.issues` is read (`openapi.ts:51`, `config.ts:474`, `:634`), and it is unchanged. `path` is now `PropertyKey[]`; `join(".")` is fine for JSON paths |
| `ZodTypeAny`, `ZodEffects`, `ZodType<…, Def>`, `ZodObject` generics, `._def` | **0** in `src/` and `tests/` | `z.infer` (119 uses in 32 files) is unchanged |
| `.refine()` / `.superRefine()`: no `ctx.path`, no function-as-second-argument, predicates do not narrow | 11 / 0 | none uses any of the three |
| `.transform()` / `z.preprocess()` return `ZodPipe` | 2 (`escalation.ts:133`, `obligation.ts:717`) / 4 (`config.ts:216`, `:234`, `:284`, `:291`) | compile, and the spec is unchanged |
| `z.coerce.*` input type is `unknown` | 20 in 6 files (e.g. `dashboard.ts:867`) | compiles; missing-and-optional and missing-with-default behave as before (measured) |
| enum-keyed records are exhaustive | **0** | both two-argument records key on `z.string().uuid()` (`workitem.ts:521`, `:580`) |
| `.extend()` on an object that carries a refinement | 14 in 7 files (e.g. `billing.ts:295`, `tenancy.ts:194`) | every base is a plain object schema; none of the 11 refined schemas is extended, so how zod 4 carries a refinement through `.extend()` does not matter here (it does not throw in 4.6.7, checked) |
| `z.nativeEnum`, `.Enum` / `.Values`, `z.function`, `z.promise`, `z.lazy`, `z.intersection`, `.nonempty()`, `.deepPartial()`, `.merge()` | **0** | `.options` (`engagement.ts:2540-2542`) still exists; `z.discriminatedUnion` (`advance.ts:39`) has the same API |
| `.openapi()` metadata | 1,943 calls in 45 files (531 named, 1,039 objects, 2 `param:`) | same API in zod-to-openapi 8/9; output in §5. The tsup bundle (`tsup.config.ts`, ESM, split into chunks) still loads: the built `dist/cmd/server.js` runs every `.openapi()` at import and reaches config validation. @hono/zod-openapi 1.5.2 fixed `.openapi()` going missing under esbuild code splitting |
| `.describe()` | 29 in 11 files | descriptions are kept in the spec |

---

## 5. The OpenAPI spec and the frontend's types

**How it was proved.** I dumped `/openapi.json` from the same `buildRouter` and fakes that
`tests/api/openapi.test.ts:781-797` uses, once on zod 3 and once on zod 4. Then I ran
bib-frontend's own `openapi-typescript` 7.13.0 on each and diffed the results. Sanity check:
the file generated from main's spec is **byte-identical** to bib-frontend main's
`src/lib/api/schema.d.ts`, so the harness spec is the real one.

| | zod 3 | zod 4 |
|---|---|---|
| Paths / schemas | 319 / 531 | 319 / 531, same paths in the same order |
| Nullable enums | `nullable: true`, `enum: [...]` | the same, plus an explicit `null` in `enum`: **34** schemas (zod-to-openapi 8.3.3, "add null option for nullable enum"), e.g. `ProposalFeeGroup.billingTiming` |
| Parameters | description on the schema | also copied onto the parameter: **114** parameters |
| Key order | | 3 parameter schemas swap `example` and `description`; `components.schemas` comes out in a different order (259 of 531 move) |
| Types, `required`, formats, patterns, min/max, paths | | **no change** |

**Generated `schema.d.ts`:** with the schemas sorted, the diff is **114 added `/** @description … */`
lines and 0 changed type lines**. The nullable enums were already `"a" | "b" | null`. Unsorted, the
reordering adds about 3,000 diff lines that change nothing.

So the frontend needs no code change. Regenerate it once, in its own commit after the backend
deploys, so reviewers know the diff is reordering plus comments. Then run `npm run check` and
`npm run build` there. The frontend has no zod dependency (`bib-frontend/package.json`) and no
zod import in `src/`.

To repeat the proof in the PR: a throwaway test that writes `await (await app().request("/openapi.json")).json()`
to a file, run once before and once after; then the frontend's `openapi-typescript` on both. Expect
exactly the three differences above. Anything else is a regression.

---

## 6. The plan, step by step

| Step | What | Mechanical? | `npm run check` | Hours |
|---|---|---|---|---|
| 0 | The mitigation (§2.6), its own PR, shipped now | by hand, ~25 lines + 5 tests | green: this changes no dependency | 3–4 |
| 1 | Branch `chore/zod4`. One commit: bump both packages (lock procedure in §3), fix the 10 compile errors (§4.1) | mechanical | green at the end of the commit. It cannot be split, because the bump breaks the build until the 10 are fixed | 1–2 |
| 2 | Keep behaviour: `.optional()` on the 6 file parts, the `error` function at `config.ts:33`, the smoke id at `smoke.sh:796`, tests for a missing file part on each upload route and for `BIB_ENVIRONMENT=prod` | by hand | green | 2 |
| 3 | `.max()` on the 6 unbounded arrays: `lines` 50 (as `dueIds`, `billing.ts:184`), `identifiers` 20, `deptIds` 50 (as `tenancy.ts:333`), `fields` `PROPOSAL_FIELDS.length` (`src/domain/gstcert/model.ts:213`, `src/domain/tradelicence/model.ts:185`); plus a regression test that a 256 KB oversized list answers one `too_big` issue in under 100 ms. The limits are proposals for whoever owns each screen to confirm | by hand | green | 1–2 |
| 4 | Verify (below) | | | 4–6 |
| 5 | Deploy through CI: merge, gate, approve "production". Then regenerate the frontend's types (§5) | | | 1 |
| later, optional | Rewrite the deprecated forms (`z.string().uuid()` → `z.uuid()` ×723, `.email()` ×16, `.datetime()` ×10, `.date()` ×11, `message` → `error` ×17). The community codemod `zod-v3-to-v4` (npm, 1.22.1) does this. I did not run it. Spec diff after: it must be empty | codemod | green | 3–4 |

**Verification (step 4), in this order:**

1. `npm run check` on **Node 22**. On Node 20, `tests/fx/feeds.test.ts` fails 5 tests on zod 3
   and zod 4 alike, because it needs the JSON reviver's `source` (Node 21+, `src/infrastructure/fx/feeds.ts:18`).
2. `npm run test:integration` on a migrated local database. **Not run in this trial**: it needs Postgres.
3. The spec diff (§5): exactly 34 nullable enums, 114 parameter descriptions and the reordering.
4. `npm run build`, then boot `dist/cmd/server.js`: it must reach config validation, not a `TypeError`.
5. `npm run smoke` against a local API started with `BIB_ESIGN_PROVIDER=attested
   BIB_MAIL_TRANSPORT=log`, because `.env` sends real DocuSign envelopes and real Gmail.
6. An upload walk in the UI: a document and a new version, a GST certificate, a trade licence,
   the client import, and a portal ask file. These exercise multipart, the 415 gate and the
   `z.any()` change, none of which a unit test does today.
7. The advisory, again: the 256 KB `settings` body answers 400 with one issue in under 100 ms.
8. `npm audit --omit=dev`: no zod, zod-openapi or zod-to-openapi entries.
9. Frontend: `npm run api:generate` against the local API, then `npm run check` and `npm run build`.

**Total:** step 0 is half a day; steps 1–5 are about 1.5 days, a third of it verification.
About 2 days in all, plus half a day if the cosmetic codemod is wanted.

### What could go wrong

| Risk | Likelihood | Guard |
|---|---|---|
| A path only the integration suite or smoke reaches sends a non-RFC uuid or asserts on zod's wording | low: one found (`smoke.sh:796`) | verification items 2 and 5 |
| Open branches rebase onto zod 4 with zod-3-only forms (a single-argument `z.record`, `required_error`) | likely, since several feature worktrees are open | the compiler catches these. A new `z.any()` file part does not, so every upload route needs a missing-part test (step 2) |
| Something sends JSON with no Content-Type or the wrong one | low (§4.2) | now a 400 "Unsupported Media Type"; smoke and the walk |
| Steps 1–2 ship without step 3 | | unbounded lists become 500s (§2.4). Steps 0 and 3 both close it; ship step 3 in the same release |
| zod's new wording reaches a reader | none in code (§4.2) | note it in the PR |
| Rollback | | revert the merge. There is no migration and no data change |
