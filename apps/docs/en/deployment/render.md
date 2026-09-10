# Render Deployment (Free: CPA + Supabase + Lightweight Panel)

This page explains how to deploy **CPA (the CLIProxyAPI gateway itself)** on a Render **Free instance**, using free **Supabase Postgres** for persistence — no paid Render Disk required. The admin UI is the CPAMP **Lightweight Panel** (hosted directly by CPA; no separate Manager Server service needed).

If you need request history, usage/cost analytics, or account automation, those only come from CPAMP **Full Mode** (Manager Server), and its SQLite storage has no equivalent free hosting option — it needs a paid Render Disk; see [Docker Deployment](./docker.md). This page covers only the free path: CPA itself plus the Lightweight Panel.

## Why This Combination

- CPA ships with a pluggable remote storage backend (`PGSTORE_DSN`). When set, it persists the full `config.yaml` content and account auth tokens to Postgres; the container's local disk is just a disposable cache that gets rehydrated from the database on every restart. That matches exactly how a Render Free instance behaves — no persistent Disk, filesystem can be rebuilt at any time — with zero code changes required.
- The CPAMP Lightweight Panel needs no separate service, database, or extra port; it's just a different UI CPA hosts itself, which makes it free by nature.

The trade-off: the Lightweight Panel has no persisted request history, cost analytics, or server-side automation, and Free instances spin down when idle, adding cold-start latency. See [Known Limitations Of The Free Plan](#known-limitations-of-the-free-plan) below.

## Architecture Overview

Just one Render service:

| Service | Purpose | Port | Source |
| --- | --- | --- | --- |
| `cli-proxy-api` | The gateway plus the lightweight admin panel; both clients and admins connect here directly | `8317` | Docker image `eceasy/cli-proxy-api:latest` |

Persistence: Supabase Postgres (`config.yaml` + account auth tokens).

The [`render.yaml`](https://github.com/seakee/CPA-Manager-Plus/blob/main/render.yaml) Blueprint at the repo root is written for exactly this setup — use Render Dashboard -> New -> Blueprint to create it in one shot. Double-check field names against Render's current docs; if the Blueprint fails, follow the manual steps below instead.

## Step 1: Create A Supabase Postgres Database

1. Create a free project on [Supabase](https://supabase.com/).
2. Go to Project Settings -> Database -> **Connection pooling**, pick **Session** mode, and copy the connection string.

   **Do not use Transaction mode.** CPA's Postgres driver caches prepared statements. Transaction-mode pooling can route different requests on the same logical connection to different backend Postgres connections, which collides with that cache and fails startup with something like `prepared statement "stmtcache_..." already exists (SQLSTATE 42P05)`. Session mode pins each connection to one backend connection for its lifetime, behaving like a direct connection, so it doesn't hit this problem — and for this setup's single Render instance, Session mode is plenty.
3. Replace the password placeholder in the connection string with the database password you set when creating the project, and keep the full DSN handy — you'll paste it into Render next.

## Step 2: Deploy CPA On Render

1. Render Dashboard -> **New** -> **Web Service** -> choose "Existing Image" and enter:

   ```text
   docker.io/eceasy/cli-proxy-api:latest
   ```

2. Pick **Free** as the instance type (this setup needs no Disk).
3. Environment variables:

   | Key | Value |
   | --- | --- |
   | `PORT` | `8317` (tells Render which port to route to; CPA itself doesn't read this variable) |
   | `PGSTORE_DSN` | the Supabase connection string from step 1 |
   | `TZ` | optional, e.g. `Asia/Shanghai` |

4. Once deployed, open the Render-assigned URL (e.g. `https://cli-proxy-api-xxxx.onrender.com`) and check the Logs: CPA should start with no Postgres connection errors.

On the very first boot, Postgres has no stored configuration yet, so CPA seeds a default config from its bundled `config.example.yaml` template and writes it to Supabase. Every subsequent cold start rehydrates that config from Supabase instead of depending on the Render container's local disk.

## Step 3: Configure config.yaml

Render's interactive **Shell** is a paid-instance feature and isn't available on Free, so you can't open a terminal into the container. You don't need it — write the config directly into the `config_store` table through **Supabase's SQL Editor** instead (a free Supabase feature, unrelated to your Render plan). CPA reads it back the next time it loads config from the database.

CPA creates the `config_store` / `auth_store` / `cooldown_store` tables the first time it successfully connects to Postgres (even while still running the default template config), so complete Step 2 first and confirm in the Logs that CPA has started successfully at least once before doing this.

1. Generate two random strings — one for `secret-key` (used to log into the panel), one for `api-keys` (used by clients to call CPA):

   ```bash
   openssl rand -hex 32
   ```

2. Supabase Dashboard -> **SQL Editor** -> New query, paste (replacing the two placeholder strings with the ones you just generated):

   ```sql
   INSERT INTO config_store (id, content, created_at, updated_at)
   VALUES (
     'config',
     $cfg$
   host: ""
   port: 8317
   auth-dir: "~/.cli-proxy-api"
   debug: false

   remote-management:
     allow-remote: true
     secret-key: "replace-with-your-first-random-string"
     disable-control-panel: false
     disable-auto-update-panel: false
     panel-github-repository: "https://github.com/seakee/CPA-Manager-Plus"

   usage-statistics-enabled: false

   api-keys:
     - "sk-replace-with-your-second-random-string"
   $cfg$,
     NOW(),
     NOW()
   )
   ON CONFLICT (id) DO UPDATE SET content = EXCLUDED.content, updated_at = NOW();
   ```

   `id` is fixed as `'config'` — that matches the key CPA itself uses for this row. The `$cfg$ ... $cfg$` dollar-quoting lets you paste the whole YAML block as a string without worrying about quotes or colons inside it. This `INSERT ... ON CONFLICT DO UPDATE` is the exact statement CPA uses internally, so it stays compatible when CPA later saves config on its own.

   If this errors with `relation "config_store" does not exist`, CPA hasn't connected to the database successfully yet — check `PGSTORE_DSN` and the Logs, confirm it has started at least once, then re-run this query.

3. Confirm it saved:

   ```sql
   select id, left(content, 50), updated_at from config_store;
   ```

4. Back on Render, use **Manual Deploy -> Restart service** (the basic restart button, available on Free — different from the Shell). CPA finds the `config` row already in `config_store` on restart and loads it.

You don't need `usage-statistics-enabled`: the Lightweight Panel doesn't consume the usage queue, so it doesn't matter either way.

## Step 4: Open The Lightweight Panel

```text
https://<your-cpa-service>.onrender.com/management.html
```

Log in with the CPA Management Key you set in the previous step. On first visit, CPA fetches `management.html` from the latest Release of the repository named in `panel-github-repository` and caches it, so it needs access to GitHub.

Confirm the Dashboard, Configuration, Providers, Accounts, OAuth, and Logs pages all load correctly.

## Remote OAuth Login

Render only exposes CPA's `8317` port publicly. The extra ports CPA uses during login to receive OAuth callbacks from each provider (`8085` / `1455` / `54545` / `51121` / `11451`, etc.) are never exposed, and don't need to be:

1. Start the login from the Lightweight Panel's OAuth Login page.
2. After completing authorization with the provider, the browser will try to redirect to `http://localhost:<port>/...`, which will always fail — that port only exists inside the Render container and your browser can't reach it.
3. Copy the **full** callback URL from the browser's address bar and paste it into the panel's callback URL field, then submit to finish login.

See [OAuth Login](../manual/oauth.md) for details. Vertex service account import doesn't use a browser callback and is unaffected.

## Known Limitations Of The Free Plan

- **Free instances spin down.** After a period without requests, Render lets the container sleep; the next request triggers a cold start (spin the container back up, rehydrate config from Supabase), adding a few seconds to tens of seconds of latency. If first-request latency matters to you, upgrade to a paid instance.
- **Supabase free projects pause too.** After a period of no database activity (currently 7 days on Supabase), a free project auto-pauses and the next connection attempt fails until you manually resume it from the Supabase Dashboard. If CPA itself also sits unused for a while, both sleeps can stack — the first request failing afterward is expected; retry, or resume the Supabase project manually first.
- **Render's free tier has its own limits** (instance count, monthly running hours, etc.) — check Render's current pricing page rather than relying on numbers here, since they change over time.
- **The Lightweight Panel has no request history, cost analytics, or account automation** — that's by design, not a bug. If you need those, look at CPAMP Full Mode ([Docker Deployment](./docker.md)), which requires a paid Render Disk since this repo's SQLite storage has no free hosting equivalent.

## Troubleshooting

- **Logs show `failed to bootstrap postgres-backed config` / `prepared statement "stmtcache_..." already exists (SQLSTATE 42P05)`** — `PGSTORE_DSN` is using Supabase's **Transaction** pooling mode, which is incompatible with the driver's prepared-statement cache. Switch the pooling mode to **Session** in Supabase, update the `PGSTORE_DSN` env var on Render, and restart — see [Step 1](#step-1-create-a-supabase-postgres-database). This error happens while reading the database and doesn't damage anything you already wrote into `config_store`.
- **The panel won't open** — confirm `secret-key` is non-empty, `disable-control-panel: false`, the service actually restarted, and the logs show no config parsing errors.
- **Can't connect to Supabase / startup shows a database error (not the 42P05 one above)** — confirm the password in `PGSTORE_DSN` is correct and the Supabase project isn't Paused.
- **`config_store` still shows the default template, not what you wrote** — confirm the SQL in [Step 3](#step-3-configure-config-yaml) actually ran successfully (check with the `select` query), and that you're checking *after* a Render restart rather than before.
- **The official panel still shows up instead of CPAMP** — check the spelling of `panel-github-repository` and confirm CPA can reach GitHub Releases.
