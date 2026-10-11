# Langflow v1 AI Agent Builder — Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Langflow v1 AI Agent Builder** (version label `v1` = Langflow major version; releases are tagged `v1.x.y`) |
| Description | Build agents, RAG pipelines and MCP servers visually, backed by Postgres. (82 chars) |
| Category | AI/ML |
| Tags | langflow, agents, rag, mcp, llm, low-code, ai |
| User input at deploy | **None.** Admin password and secret key are generated. |

## Base market template

- **`JMXEWp` "Langflow"** (2,883 projects, 1,409 active) — chosen as base: the most used, already Postgres-backed with `/health_check`. Kept: official image, Postgres, `LANGFLOW_DATABASE_URL`, generated `LANGFLOW_SECRET_KEY`. Replaced: `latest` tag, debug logging, missing volume, missing admin bootstrap, public Postgres TCP proxy.
- `LcJLmZ` "Langflow" (1,301 projects) — same layout with `latest`, no healthcheck, no volume. Nothing extra taken.
- `UsJ1uB` "Langflow Pre-releases" (358 projects) — pinned to `1.0-alpha`; obsolete. Nothing taken.
- No `_upstream` repos for this app. Market templates have no license → reference only.

## Pinned versions

| Service | Image / source | Tag | Release date | Checked |
|---|---|---|---|---|
| Langflow | `langflowai/langflow` (default profile) | `1.12.5` | 2026-10-06 | 2026-10-10 |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl` | `18` | Railway base | 2026-10-10 |
| Redis | `redis` | `8.2` | Railway base | 2026-10-10 |

- Image profile: the *default* image (`langflowai/langflow:<ver>`) matches `pip install langflow` (curated `lfx-*` providers, no PyTorch). Swap to `langflowai/langflow-all:1.12.5` for the extended provider set or `langflowai/langflow:base-1.12.5` for the smallest image.
- Langflow ≥ 1.12 requires PostgreSQL ≥ 15 (checked at boot); Railway Postgres 18 satisfies it.
- Upgrade path: back up Postgres, change the tag, redeploy. Migrations run on start under a Postgres advisory lock.

## Architecture

```
Internet ──HTTPS──▶ Langflow :7860  (UI + API + MCP endpoints, 2 Gunicorn workers)
                      ├─▶ Postgres  (flows, users, messages, variables)
                      ├─▶ Redis     (DB 1: shared build/job queue · DB 2: rate-limit counters)
                      └─▶ Volume /app/langflow (uploaded files, knowledge bases, logs, MCP state)
```

Upstream reference: `docker_example/docker-compose.yml` (Langflow + Postgres + `/app/langflow` volume) and the multi-worker guide (`docs/docs/Deployment/deployment-multi-worker.mdx`), which adds Redis as the job queue so more than one worker can serve the visual editor. Langflow has no separate worker image in 1.12 (the old Celery/RabbitMQ `deploy/` stack is legacy), so none is added.

Why Redis: with a single worker, CPU-heavy components block every other user. Langflow refuses to start more than one worker unless the job queue is shared (`LANGFLOW_JOB_QUEUE_TYPE=redis`); otherwise editor builds randomly fail with "Job queue not found". Redis also makes login rate limits global instead of per worker.

Why no Bucket: the config dir must persist anyway (knowledge bases, MCP state, logs), so uploaded files stay on the same volume (`LANGFLOW_STORAGE_TYPE=local`). Upstream's `prod` deployment profile (S3 storage + pgvector + Redis, stateless replicas) is documented under Open questions as the scale-out path.

## Services

Create in this order. Values are entered exactly as shown (Railway resolves `${{...}}`).

### Langflow

- **Source:** Docker image `langflowai/langflow:1.12.5`
- **Start command:** image default
- **Healthcheck:** `/health_check` (timeout 600 s)
- **Public domain:** yes → target port `7860`
- **TCP proxy:** none
- **Volume:** `/app/langflow`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1

| Variable | Value | Description |
|---|---|---|
| `PORT` | `7860` | Port Railway routes the public domain to. |
| `LANGFLOW_HOST` | `0.0.0.0` | Bind address (Langflow's port check only supports IPv4 literals). |
| `LANGFLOW_PORT` | `${{PORT}}` | Langflow listen port. |
| `RAILWAY_RUN_UID` | `0` | Run as root so the Railway volume (root-owned) is writable; the image user is uid 1000. |
| `LANGFLOW_CONFIG_DIR` | `/app/langflow` | Config dir on the volume: uploaded files, logs, MCP/Composer state. |
| `LANGFLOW_KNOWLEDGE_BASES_DIR` | `/app/langflow/knowledge_bases` | Knowledge bases on the volume (default ~/.langflow is outside it). |
| `LANGFLOW_FS_TOOL_BASE_DIR` | `/app/langflow/fs_tool` | File System component sandbox on the volume. |
| `LANGFLOW_DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Postgres connection (converted to the async psycopg driver by Langflow). |
| `LANGFLOW_DATABASE_CONNECTION_RETRY` | `true` | Retry the DB connection while Postgres is still starting. |
| `LANGFLOW_DB_CONNECTION_SETTINGS` | `{"pool_size": 10, "max_overflow": 10, "pool_timeout": 30, "pool_pre_ping": true, "pool_recycle": 1800, "echo": false}` | SQLAlchemy pool per worker, sized for Postgres' 100-connection limit. |
| `LANGFLOW_SECRET_KEY` | `${{secret(43, "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_")}}` | Encryption key for stored credentials/API keys and JWT signing. Never change after first boot. |
| `LANGFLOW_AUTO_LOGIN` | `false` | Require login (multi-user mode). |
| `LANGFLOW_SUPERUSER` | `admin` | Admin username created on first boot. |
| `LANGFLOW_SUPERUSER_PASSWORD` | `${{secret(24, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")}}` | Admin password (generated). Read it here to sign in. |
| `LANGFLOW_NEW_USER_IS_ACTIVE` | `false` | New sign-ups stay inactive until an admin activates them. |
| `LANGFLOW_WORKERS` | `2` | Gunicorn worker processes. Safe above 1 because the job queue is in Redis. |
| `LANGFLOW_WORKER_TIMEOUT` | `300` | Worker timeout in seconds. |
| `LANGFLOW_JOB_QUEUE_TYPE` | `redis` | Shared build/job queue so any worker can serve any flow run. |
| `LANGFLOW_REDIS_QUEUE_URL` | `redis://${{Redis.REDISUSER}}:${{Redis.REDIS_PASSWORD}}@${{Redis.REDISHOST}}:${{Redis.REDISPORT}}/1` | Redis URL for the job queue (DB 1, authenticated). |
| `LANGFLOW_RATE_LIMIT_STORAGE_URI` | `redis://${{Redis.REDISUSER}}:${{Redis.REDIS_PASSWORD}}@${{Redis.REDISHOST}}:${{Redis.REDISPORT}}/2` | Shared login/public-flow rate-limit counters (DB 2). |
| `LANGFLOW_RATE_LIMIT_TRUST_PROXY` | `true` | Use X-Forwarded-For from Railway's proxy so limits apply per client, not per proxy. |
| `LANGFLOW_CORS_ORIGINS` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Allowed browser origins. Add sites that embed the Langflow chat widget (comma-separated). |
| `LANGFLOW_MCP_BASE_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Public origin used in the MCP server URLs Langflow shows for projects. |
| `LANGFLOW_LOG_LEVEL` | `info` | Log level (debug, info, warning, error). |
| `LANGFLOW_ALEMBIC_LOG_TO_STDOUT` | `true` | Show database migration logs in Railway logs. |
| `LANGFLOW_DO_NOT_TRACK` | `true` | Disable anonymous product telemetry. |

### Postgres

- **Source:** Docker image `ghcr.io/railwayapp-templates/postgres-ssl:18`
- **Start command:** image default
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/var/lib/postgresql/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1
- **Note:** Copied unchanged from templates/_railway-base/postgres.json

| Variable | Value | Description |
|---|---|---|
| `PGDATA` | `/var/lib/postgresql/data/pgdata` | Location where the database will be initialized |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Railway Private Domain Name. |
| `PGPORT` | `5432` | Port to connect to Postgres. |
| `PGUSER` | `${{ POSTGRES_USER }}` | Required variable for Data panel |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Required variable for the data panel. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Required variable for Data panel |
| `POSTGRES_DB` | `railway` | Default database created when image is started. |
| `DATABASE_URL` | `postgresql://${{PGUSER}}:${{POSTGRES_PASSWORD}}@${{RAILWAY_PRIVATE_DOMAIN}}:5432/${{PGDATABASE}}` | URL to connect to Postgres database. |
| `POSTGRES_USER` | `postgres` | User to connect to Postgres DB |
| `SSL_CERT_DAYS` | `820` | SSL certificate expiry in days. |
| `POSTGRES_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Password to connect to DB |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Allow Postgres to cleanly shut down |

### Redis

- **Source:** Docker image `redis:8.2`
- **Start command:** `/bin/sh -c "rm -rf $RAILWAY_VOLUME_MOUNT_PATH/lost+found/ && exec docker-entrypoint.sh redis-server --requirepass $REDIS_PASSWORD --save 60 1 --dir $RAILWAY_VOLUME_MOUNT_PATH"`
- **Healthcheck:** none (no HTTP endpoint)
- **Public domain:** no (private network only)
- **TCP proxy:** none
- **Volume:** `/data`
- **Restart policy:** ON_FAILURE (max 10 retries)
- **Replicas:** 1
- **Note:** Copied unchanged from templates/_railway-base/redis.json

| Variable | Value | Description |
|---|---|---|
| `REDISHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Private network hostname of the Redis service, only resolvable from services in the same environment |
| `REDISPORT` | `6379` | Port that Redis listens on |
| `REDISUSER` | `default` | Username for authenticating with Redis |
| `REDIS_URL` | `redis://${{ REDISUSER }}:${{ REDIS_PASSWORD }}@${{ REDISHOST }}:${{ REDISPORT }}` | Connection string for connecting to Redis using the private network |
| `REDISPASSWORD` | `${{REDIS_PASSWORD}}` | Alias of REDIS_PASSWORD for clients that expect the unseparated name |
| `REDIS_PASSWORD` | `${{ secret(32, "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ") }}` | Randomly generated password for authenticating with Redis |

## Improvements over market templates

1. **Pinned image** `langflowai/langflow:1.12.5` instead of `latest` (upstream explicitly recommends pinning; `latest` can apply DB migrations without notice).
2. **Admin bootstrap**: since 1.11 the image defaults to `LANGFLOW_AUTO_LOGIN=false`, and upstream docs require `LANGFLOW_SUPERUSER_PASSWORD` in that mode; the market templates set neither. Here `LANGFLOW_SUPERUSER=admin` + generated password, sign-ups inactive until approved.
3. **Persistent volume** at `/app/langflow` (`LANGFLOW_CONFIG_DIR`), plus knowledge bases and the File System tool redirected onto it — market templates lose uploads, knowledge bases and logs on every redeploy.
4. **Volume permissions handled** (`RAILWAY_RUN_UID=0`): the image runs as uid 1000, Railway volumes mount root-owned.
5. **Two workers with a shared Redis job queue** (upstream multi-worker guide) instead of one blocking worker; Redis-backed rate limits with `LANGFLOW_RATE_LIMIT_TRUST_PROXY=true` so limits are per client, not per Railway proxy.
6. **Connection pool sized for Railway Postgres** (10 + 10 per worker, pre-ping) — upstream default 20 + 30 per worker could exhaust 100 connections; DB connection retry on cold start.
7. **CORS locked to the app's own origin** (upstream default `*` with credentials); MCP base URL set to the public domain so project MCP URLs are correct.
8. **Postgres not exposed publicly** (market templates add a TCP proxy) and uses the Railway Postgres 18 base; `info` logging instead of `debug`; telemetry off.
9. `/health_check` readiness probe (checks DB) with a 600 s timeout for first-boot migrations and starter-project creation.

## Scaling

- Raise `LANGFLOW_WORKERS` (≈ 1 per vCPU; each worker uses 0.6–1 GB RAM). Upstream's starting points: 5 workers on 4–8 GB hosts, plus `LANGFLOW_GUNICORN_PRELOAD=true` (experimental, lowers per-worker RAM).
- Keep `workers × (pool_size + max_overflow)` below ~90.
- Replicas: not possible while the config-dir volume is attached (Railway volumes are single-service). See Open questions for the stateless setup.

## Resource estimate

Prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month.

| Service | Idle RAM | Typical RAM | Typical vCPU | Volume |
|---|---|---|---|---|
| Langflow (2 workers) | 1.4 GB | 2.2 GB | 0.05 idle / 0.35 | ~2 GB |
| Postgres | 80 MB | 200 MB | 0.02 | ~1 GB |
| Redis | 10 MB | 30 MB | 0.01 | ~0.1 GB |
| **Total** | **≈ 1.5 GB** | **≈ 2.4 GB** | **0.08 / 0.38** | **≈ 3.1 GB** |

- **Idle:** 1.5 × $10 + 0.08 × $20 + 3.1 × $0.15 ≈ **$17/month**
- **Typical:** 2.4 × $10 + 0.38 × $20 + $0.5 ≈ **$32/month**
- With 4 workers for a busy team: ≈ $45–55/month.
- **Minimum plan: Hobby.** The Trial's 1 GB per service is below two Langflow workers (set `LANGFLOW_WORKERS=1` to squeeze into Trial for evaluation).

## Verification checklist

1. `GET https://<domain>/health_check` → `200` with `"status":"ok"`; `GET /health` → `{"status":"ok"}`.
2. Logs show Alembic migrations, `Launching Langflow...` and 2 booted workers; no "Refusing to start with 2 workers" error.
3. Sign in at `https://<domain>/` with `admin` / `LANGFLOW_SUPERUSER_PASSWORD` (Langflow service variables).
4. Create a flow from the *Basic Prompting* template, add an OpenAI/Anthropic key as a global variable, run it in the Playground several times (exercises the Redis job queue across both workers).
5. As admin, `GET /api/v1/monitor/job_queue` with an API key → `"backend": "redis"`, `"cancel_dispatcher_running": true`.
6. Upload a file under *My Files* and create a knowledge base; redeploy the service → file, knowledge base, flows and the global variable are still present and the variable still decrypts (secret key stable).
7. Open a project's MCP server panel → URLs use `https://<domain>`.
8. Six wrong logins within one minute from one client → the sixth returns `429` (limit 5/min per client IP), while a second client can still log in.

## Open questions

- **Not deployed yet**: verified against upstream source/docs for v1.12.5 only.
- **IPv6-only private networks**: Langflow binds `0.0.0.0` (its startup port check uses an IPv4 socket, so `::` fails). Only the public domain reaches it, which works; nothing in this template calls Langflow privately.
- **`RAILWAY_RUN_UID=0`** runs Langflow as root inside the container. Alternative: a wrapper image that `chown`s the volume and drops to uid 1000.
- **Stateless scale-out** (upstream `LANGFLOW_DEPLOYMENT_PROFILE=prod`): requires `LANGFLOW_STORAGE_TYPE=s3`, a pgvector Postgres with `CREATE EXTENSION vector`, and `PGVECTOR_CONNECTION_STRING`. Langflow's S3 backend uses aiobotocore without an endpoint option; a Railway Bucket would need `AWS_ENDPOINT_URL_S3` (standard botocore env) — untested, so not enabled here.
- **Embedded chat widget**: browsers on other sites need their origin added to `LANGFLOW_CORS_ORIGINS`.
- **Telemetry** is disabled (`LANGFLOW_DO_NOT_TRACK=true`); remove the variable to opt back in.
