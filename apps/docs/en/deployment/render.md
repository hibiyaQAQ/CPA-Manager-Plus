# Render Deployment

Render is a managed container platform that can deploy directly from a Docker image or from a Dockerfile in this repository. This page explains how to deploy both **CPA (the CLIProxyAPI gateway itself)** and **CPAMP Full Mode (Manager Server)** on Render, and how to connect them.

If you first want to understand the difference between the two modes, see [Choosing A Panel](../guide/choosing-a-panel.md). If you don't need CPA itself on Render (it already runs elsewhere), skip step one and only do step two.

## Read Before You Deploy

- **Both services must run on a paid instance type that supports Disks** (Starter or above). Render's Free instances have no persistent Disk and spin down when idle. CPA needs to persist `config.yaml` and account auth files; CPAMP needs to persist its SQLite database and `data.key`. Both are required.
- Each Render service can attach only one Disk and only runs as a single instance (no autoscaling). That matches the single-instance requirement CPA and CPAMP already have, so it isn't an extra limitation.
- Render's public ingress only forwards HTTPS traffic and does not support the raw TCP protocol RESP collection needs, so CPAMP's usage collector must be explicitly set to HTTP queue mode (`USAGE_COLLECTOR_MODE=http`).
- The official CPA image `eceasy/cli-proxy-api` does not ship a `config.yaml` by default (only `config.example.yaml`), so one must be generated and persisted at startup.
- OAuth login on Render can only go through the "remote browser callback" flow (paste the callback URL manually). The extra ports CPA uses to receive OAuth callbacks (e.g. `8085` / `1455` / `54545` / `51121` / `11451`) are never exposed publicly, and don't need to be.

## Architecture Overview

| Service | Purpose | Port | Source |
| --- | --- | --- | --- |
| `cli-proxy-api` (CPA itself) | The gateway that receives and forwards real model requests; Codex / Claude Code / OpenCode and other clients connect here directly | `8317` | Docker image `eceasy/cli-proxy-api:latest` |
| `cpa-manager-plus` (CPAMP Full Mode) | Manager Server: request history, usage analytics, account health and automation | `18317` | Built from this repo's `Dockerfile.manager-server` |

To create both services at once, use the [`render.yaml`](https://github.com/seakee/CPA-Manager-Plus/blob/main/render.yaml) Blueprint at the repository root (Render Dashboard -> New -> Blueprint, pick this repo). Double-check field names against Render's current Blueprint documentation; if the Blueprint deploy fails, create the two Web Services manually using the steps below — the underlying configuration is identical.

## Step 1: Deploy CPA Itself

1. Render Dashboard -> **New** -> **Web Service** -> choose "Existing Image" and enter:

   ```text
   docker.io/eceasy/cli-proxy-api:latest
   ```

2. Pick **Starter or above** as the instance type (Disks require a paid plan).
3. Add a Disk: mount path `/data`, size 1–2GB is enough to start (config and auth files are small).
4. Override the start command (Docker Command). Render only lets you mount one Disk, but CPA needs two persistent locations (`config.yaml` and the auth directory), so use a small shell wrapper that symlinks files under `/data` to the paths the image expects:

   ```sh
   sh -c '
   mkdir -p /data/auths &&
   if [ ! -f /data/config.yaml ]; then cp /CLIProxyAPI/config.example.yaml /data/config.yaml; fi &&
   ln -sfn /data/config.yaml /CLIProxyAPI/config.yaml &&
   rm -rf /root/.cli-proxy-api &&
   ln -sfn /data/auths /root/.cli-proxy-api &&
   exec ./CLIProxyAPI
   '
   ```

5. Add an environment variable `PORT=8317` so Render forwards public traffic to the container's `8317` port (CPA itself does not read `PORT`; this only affects Render's routing).
6. Once deployed, open the Render-assigned URL (e.g. `https://cli-proxy-api-xxxx.onrender.com`) and check the Logs for configuration errors.
7. Open the service's **Shell** tab and edit `/data/config.yaml`, setting at least:

   ```yaml
   remote-management:
     secret-key: 'a sufficiently long random string' # this is your CPA Management Key
     allow-remote: true

   usage-statistics-enabled: true
   redis-usage-queue-retention-seconds: 60

   api-keys:
     - 'sk-your-own-client-key'
   ```

   Save and restart the service. CPA rewrites the plaintext `secret-key` into a bcrypt hash on startup and writes it back to `config.yaml`, so this file must stay writable — mounting it on the Disk guarantees that.

8. To sign in to Codex / Claude / Gemini and other OAuth providers, see [Remote OAuth Login](#remote-oauth-login) below.

## Step 2: Deploy CPAMP Full Mode

1. Render Dashboard -> **New** -> **Web Service** -> connect this repository.
2. Set Runtime to **Docker**, Dockerfile Path to `Dockerfile.manager-server`, and Docker Context to the repo root `.`.
3. Pick **Starter or above** as the instance type.
4. Add a Disk: mount path `/data`, 5–10GB is a reasonable starting size (SQLite grows with request history, depending on volume and retention).
5. Environment variables:

   | Key | Suggested value | Notes |
   | --- | --- | --- |
   | `PORT` | `18317` | Tells Render which port to route to |
   | `HTTP_ADDR` | `0.0.0.0:18317` | Manager Server listen address |
   | `USAGE_DB_PATH` | `/data/usage.sqlite` | SQLite path |
   | `CPA_MANAGER_DATA_KEY_PATH` | `/data/data.key` | Data encryption key path |
   | `CPA_MANAGER_ADMIN_KEY` | a long random string | Set explicitly to avoid digging through logs for the generated key after every restart |
   | `USAGE_COLLECTOR_MODE` | `http` | Render's public ingress doesn't support the raw TCP protocol RESP needs |
   | `USAGE_BATCH_SIZE` | `100` | Max records collected per batch |
   | `USAGE_POLL_INTERVAL_MS` | `500` | Idle poll interval |
   | `USAGE_QUERY_LIMIT` | `50000` | Cap on recent usage events returned |
   | `USAGE_CORS_ORIGINS` | `*` or your own domain | Allowed CORS origins |

6. Set Health Check Path to `/health`.
7. Once deployed, open:

   ```text
   https://<your-cpamp-service>.onrender.com/management.html
   ```

   If you didn't set `CPA_MANAGER_ADMIN_KEY` explicitly, find the one-time generated admin key printed at startup in the Render Dashboard Logs.
8. Fill in the first-run setup with:
   - **Admin key**: the key from the previous step (or whatever you set as `CPA_MANAGER_ADMIN_KEY`).
   - **CPA URL**: the CPA service's Render public address, e.g. `https://cli-proxy-api-xxxx.onrender.com`.
   - **CPA Management Key**: the plaintext value you set for `remote-management.secret-key` in step 1's `config.yaml` (the original value, before CPA hashes it).

## Same-Region Private Networking (Optional)

If CPA and CPAMP run in the **same Render Region**, CPAMP can in principle reach CPA over Render's private network instead of going out over the public internet:

```text
http://<CPA service's Render name>:8317
```

Conditions and caveats:

- Both services must be in the same Region, or the internal address won't resolve.
- Whether Render's private network transparently carries the raw TCP protocol RESP collection needs is something to confirm against Render's current documentation or support — if unsure, keep `USAGE_COLLECTOR_MODE=http` and use the public HTTPS address; the performance cost is negligible.
- Either way, CPA still needs a public HTTPS address for Codex / Claude Code and other clients to call directly. This step only optimizes the CPAMP-to-CPA management traffic; it can't replace CPA's public entry point.

## Remote OAuth Login

Render only exposes CPA's `8317` port publicly. The extra ports CPA uses during login to receive OAuth callbacks from each provider (`8085` / `1455` / `54545` / `51121` / `11451`, etc.) are never exposed, and don't need to be:

1. Start the login from the OAuth Login page in CPAMP Full Mode.
2. After completing authorization with the provider, the browser will try to redirect to `http://localhost:<port>/...`, which will always fail — that port only exists inside the Render container and your browser can't reach it.
3. Copy the **full** callback URL from the browser's address bar and paste it into the callback URL field on the CPAMP page, then submit to finish login.

See [OAuth Login](../manual/oauth.md) for details. Vertex service account import doesn't use a browser callback and is unaffected.

## Backups

- **CPA**: Periodically tar up `/data/config.yaml` and `/data/auths` via the Render Shell and upload them to your own object storage. Render currently has no built-in Disk snapshot or one-click download feature.
- **CPAMP**: Back up `/data/usage.sqlite*` and `/data/data.key` the same way as [Backup And Restore](../operations/backup.md), just run the commands from the Render Shell instead. Losing `data.key` means any saved CPA Management Key can't be recovered — you'll need to save the CPA connection again from the panel.

## Troubleshooting

- **The service goes unreachable after a few idle minutes / cold starts are slow** — you're on a Free instance. Free instances spin down when idle and have no Disk; both services need at least Starter.
- **CPAMP keeps showing CPA as disconnected** — confirm the CPA URL is the full HTTPS address (with `https://`), the CPA service's Logs show no errors, and `remote-management.allow-remote: true` has taken effect after a restart.
- **Monitoring shows no data** — confirm CPA's `usage-statistics-enabled: true` was saved, CPAMP's `USAGE_COLLECTOR_MODE=http`, then see [Monitoring Has No Data](../troubleshooting/request-monitoring.md).
- **CPA auth data or CPAMP history disappears after a restart** — confirm the service actually has a Disk attached, and that the start command / env vars write data under the Disk's mount path (`/data`) rather than some other directory that gets rebuilt.
- **You see `unsupported RESP prefix 'H'`** — the collector is still trying RESP mode; confirm `USAGE_COLLECTOR_MODE` is set to `http` and redeploy.
