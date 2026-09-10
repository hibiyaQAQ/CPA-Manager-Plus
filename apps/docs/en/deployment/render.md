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
2. Go to Project Settings -> Database -> **Connection pooling**, pick **Transaction** mode, and copy the connection string (something like `postgresql://postgres.xxxx:[YOUR-PASSWORD]@aws-xxx.pooler.supabase.com:6543/postgres`).

   Use the pooled connection (usually port `6543`) rather than the direct connection (port `5432`): a Render Free instance cold-starting repeatedly opens fresh connections each time, and the pooler avoids exhausting the limited connection slots on a Supabase free project.
3. Replace `[YOUR-PASSWORD]` with the database password you set when creating the project, and keep the full DSN handy — you'll paste it into Render next.

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

Open the service's **Shell** tab and edit the locally materialized config file (inside the image it's at `config.yaml`, typically `/CLIProxyAPI/config.yaml`), setting at least:

```yaml
remote-management:
  allow-remote: true # the browser reaches the panel from the public internet, so this must be on
  secret-key: 'a sufficiently long random string' # this is your CPA Management Key
  disable-control-panel: false
  disable-auto-update-panel: false
  panel-github-repository: 'https://github.com/seakee/CPA-Manager-Plus'

api-keys:
  - 'sk-your-own-client-key'
```

Save and restart the service. CPA syncs saved configuration back to Supabase through its built-in file watcher; if a restart shows your edits were "lost" (reverted to the default template), the sync didn't take effect that time — edit again in the Shell and confirm the logs show no errors on save. If it keeps happening, check CLIProxyAPI's own documentation for exactly when `PGSTORE` syncs — that behavior belongs to the CPA project, not CPAMP.

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

- **The panel won't open** — confirm `secret-key` is non-empty, `disable-control-panel: false`, the service actually restarted, and the logs show no config parsing errors.
- **Can't connect to Supabase / startup shows a database error** — confirm `PGSTORE_DSN` uses the pooled address (port `6543`) rather than the direct one, the password is correct, and the Supabase project isn't Paused.
- **Edits to config.yaml revert after a restart** — that edit didn't sync back to Supabase; redo [Step 3](#step-3-configure-config-yaml) and confirm it saved cleanly.
- **The official panel still shows up instead of CPAMP** — check the spelling of `panel-github-repository` and confirm CPA can reach GitHub Releases.
