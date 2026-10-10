# Opening the QA database on your laptop

This lets you browse the QA database in pgAdmin (or psql) from your own machine. There is no
password to share. You log in with your own Google account, through Google's Cloud SQL proxy.

```
pgAdmin → 127.0.0.1:5433 → proxy on your laptop → (Google checks it's you) → QA database
```

Setup takes about 10 minutes, once. After that it's one command.

---

## 0. Ask for access first

Send Anish the **Google account email** you'll use and say whether you need **read** or **write**.

- **read**: you can look at every table. You cannot change anything.
- **write**: you can change or delete anything in QA. Only ask for this if you need it.

Wait until Anish confirms you've been added. Until then, the steps below end in a login error.

## 1. Install the tools (macOS)

You need three tools. The bib-backend repo is optional: it adds the `npm run db:qa` shortcut.

| Tool | Install | Check it worked |
|---|---|---|
| Google Cloud CLI | https://cloud.google.com/sdk/docs/install (or `brew install --cask google-cloud-sdk`) | `gcloud --version` |
| Cloud SQL proxy | `brew install cloud-sql-proxy` (or `gcloud components install cloud-sql-proxy`) | `cloud-sql-proxy --version` |
| pgAdmin 4 | https://www.pgadmin.org/download/ | opens |

## 2. Sign in to Google (once per laptop)

Run these one at a time. The first and third open a browser window. Sign in with the **same
email** you sent Anish both times.

```
gcloud auth login
```
```
gcloud config set project bib-qa
```
```
gcloud auth application-default login
```

You need both logins. The first is for `gcloud` itself. The third command signs in the proxy,
which is how it logs you in to the database without a password.

## 3. Open the tunnel

If you have the bib-backend repo, run this from its folder:

```
npm run db:qa
```

It checks your sign-in, prints the pgAdmin settings, then stays running. **Leave this terminal
open** while you use pgAdmin. Press Ctrl+C to close the connection.

Without the repo, or if `npm run db:qa` isn't in your copy yet, this command does the same thing:

```
cloud-sql-proxy bib-qa:asia-south1:bib-production --port 5433 --auto-iam-authn
```

## 4. Connect pgAdmin (once)

In pgAdmin, right-click **Servers → Register → Server…**

| Tab | Field | Value |
|---|---|---|
| General | Name | `bib-qa` |
| Connection | Host name/address | `127.0.0.1` |
| | Port | `5433` |
| | Maintenance database | `bib` |
| | Username | your Google email, e.g. `priya@gmail.com` |
| | Password | leave **empty** |
| Parameters | SSL mode | `disable` (the proxy already encrypts the connection) |
| Advanced | DB restriction | `bib` (hides Google's internal databases) |

Click **Save**. From now on, connecting is: run `npm run db:qa`, then click **bib-qa** in pgAdmin.

## 5. Find your way around

Go to **bib-qa → Databases → bib → Schemas → (schema) → Tables**.

- **See rows:** right-click a table → **View/Edit Data → First 100 Rows**
- **See columns:** expand the table → **Columns**
- **Write a query:** select `bib`, then **Tools → Query Tool**. Type the query and press F5.
- **Diagram of the tables:** right-click `bib` → **ERD For Database**

| Schema | What's in it |
|---|---|
| `tenancy` | firms, staff, logins, roles |
| `core` | clients (parties), companies, PAN / GSTIN |
| `crm` | leads, proposals, engagement letters, engagements, invoices, payments |
| `delivery` | jobs, tasks, filings (the compliance calendar), client asks |
| `files` | uploaded documents |
| `config` | services, roles and other firm settings |
| `ledger` | the audit log |

### "read" access: why most tables look empty

Every firm's data is walled off per firm. The database only returns a firm's rows once you say
which firm you're looking at. In the Query Tool, run:

```sql
SELECT id, name FROM tenancy.tenant;
```

Copy the firm's `id`, then run this once per Query Tool tab:

```sql
SET app.tenant_id = '<the id>';
```

Queries in that tab now show the firm's rows. "View/Edit Data" opens its own connection, so it
still looks empty. Use the Query Tool instead.

**write** users see everything without this step.

## Rules

- This is **QA**, a real environment the client tests on. With write access, a saved edit or a
  delete is real and immediate.
- Don't copy client data (names, PAN, GSTIN, emails) into chats, tickets or your own files.
- Close the tunnel (Ctrl+C) when you're done.

## When something goes wrong

| You see | It means / do this |
|---|---|
| `One-time setup missing. Run: gcloud auth application-default login` | Step 2's third command wasn't done or didn't finish. Run it again. |
| `Not signed in to Google Cloud` | Run `gcloud auth login`. |
| `cloud-sql-proxy is not installed` | Step 1. |
| In the proxy terminal: `403` / `not authorized` / `cloudsql.instances.get` | You haven't been given access yet, or you signed in with a different email. Check with `gcloud config get account`. |
| In pgAdmin: `password authentication failed` / `role "…" does not exist` | Your database login doesn't exist yet. Ask Anish. |
| `permission denied for table …` | Your database login exists but has no read/write access yet. Ask Anish. |
| `pg_hba.conf rejects connection … database "cloudsqladmin"` | You clicked Google's internal database. Ignore it, and set **DB restriction** to `bib` (step 4). |
| `quota project` / `serviceusage` error | Ask Anish to give you the **Service Usage Consumer** role. |
| `address already in use` | Something else is on port 5433. Run `BIB_DB_PORT=5434 npm run db:qa` (or `--port 5434` on the plain command) and use port 5434 in pgAdmin. |
| `connection refused` in pgAdmin | The tunnel isn't running. Start it again (step 3). |

---

## For the admin: adding and removing people

Everything is done in the Cloud console, in the **bib-qa** project.

**Add someone**

1. **IAM & Admin → IAM → Grant access**: enter their Google email and give it two roles,
   **Cloud SQL Client** and **Cloud SQL Instance User**. Save.
2. **SQL → bib-production → Users → Add user account → Cloud IAM**: add the same email.
3. **SQL → bib-production → Cloud SQL Studio**: log in to database `bib` as user `postgres`. The
   password is in **Secret Manager → bib-owner-database-url**, between `postgres:` and `@`. Then
   run **one** of these:
   ```sql
   GRANT bib_dev_read TO "their.email@gmail.com";
   ```
   ```sql
   GRANT bib_owner TO "their.email@gmail.com";
   ```
   The first is read, the second is write. Keep the double quotes around the email.

**Change read ↔ write:** in Cloud SQL Studio, `REVOKE` one role and `GRANT` the other, e.g.
`REVOKE bib_owner FROM "their.email@gmail.com";` then the read `GRANT` above.

**Remove someone:** delete their user under **SQL → bib-production → Users**, then remove their
two roles under **IAM**.

**Check who has access:** **SQL → bib-production → Users** lists every login. Run this in Cloud
SQL Studio to see which login can do what:
```sql
SELECT m.rolname AS person, r.rolname AS access
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member
WHERE r.rolname IN ('bib_dev_read', 'bib_owner') AND m.rolname LIKE '%@%';
```

**If `bib_dev_read` does not exist** (a new environment), create it once in Cloud SQL Studio as
`postgres`. This lets it read every table, including tables added later:
```sql
CREATE ROLE bib_dev_read NOLOGIN;
DO $$
DECLARE s text; o text;
BEGIN
  FOR s IN SELECT nspname FROM pg_namespace
            WHERE nspname NOT LIKE 'pg\_%' AND nspname <> 'information_schema' LOOP
    BEGIN
      EXECUTE format('GRANT USAGE ON SCHEMA %I TO bib_dev_read', s);
      EXECUTE format('GRANT SELECT ON ALL TABLES IN SCHEMA %I TO bib_dev_read', s);
      FOREACH o IN ARRAY ARRAY['bib_owner', 'bib_app'] LOOP
        EXECUTE format('ALTER DEFAULT PRIVILEGES FOR ROLE %I IN SCHEMA %I GRANT SELECT ON TABLES TO bib_dev_read', o, s);
      END LOOP;
    EXCEPTION WHEN insufficient_privilege THEN
      RAISE NOTICE 'schema %: skipped (%)', s, SQLERRM;
    END;
  END LOOP;
END $$;
```
On QA it already exists. If a migration adds a new **schema**, run the `DO` block again.

How it's built (in bib-backend): `infra/terraform/sql.tf` gives the database a public IP with no
networks allowed in and turns on Google-account logins; `scripts/db.sh` is `npm run db:qa`.
