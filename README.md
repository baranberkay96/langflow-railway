# Deploy and Host Langflow with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/langflow-v1-ai-agent-builder)

Langflow is an open-source visual builder for AI agents, RAG pipelines and MCP servers. You connect components on a canvas, test them in a playground and serve every flow as an API or MCP tool. This community template runs a pinned Langflow release with Postgres, Redis and persistent storage.

## About Hosting Langflow

Langflow is a Python application that serves its editor, REST API and MCP endpoints from one process. For a team deployment it needs a real database instead of SQLite, a persistent configuration directory for uploaded files, knowledge bases and logs, a stable encryption key for stored credentials, and a login-protected admin account. Running more than one worker also requires a shared job queue, otherwise editor builds can land on the wrong worker. This template provides all of that on Railway: Postgres for flows and users, Redis for the job queue and rate limits, a volume for the config directory, and generated admin credentials.

## Common Use Cases

- Prototype and ship LLM agents with tools (web search, APIs, code) without writing a backend
- Build retrieval-augmented chat over your documents and expose it as an API
- Publish flows as MCP servers for Claude Desktop, Cursor and other MCP clients
- Give a team a shared, login-protected workspace for AI workflows and prompt experiments
- Embed a chat widget backed by a Langflow flow in your website or internal tools

## Dependencies for Langflow Hosting

- Langflow (`langflowai/langflow`, default image profile)
- PostgreSQL 15 or newer (Railway Postgres 18)
- Redis (job queue and rate-limit storage for multiple workers)
- A persistent volume for `LANGFLOW_CONFIG_DIR`

### Deployment Dependencies

- Langflow repository: https://github.com/langflow-ai/langflow
- Deploy Langflow on Docker: https://docs.langflow.org/deployment-docker
- Choose a Langflow Docker image: https://docs.langflow.org/deployment/docker-image-profiles
- Multiple workers with Redis: https://docs.langflow.org/deployment-multi-worker
- Environment variables: https://docs.langflow.org/environment-variables
- Authentication and API keys: https://docs.langflow.org/api-keys-and-authentication

### Implementation Details

| Service | Image | Role | Public |
|---|---|---|---|
| Langflow | `langflowai/langflow:1.12.5` | Editor, API, MCP server; 2 Gunicorn workers; volume at `/app/langflow` | **yes** (port 7860) |
| Postgres | Railway Postgres 18 | Flows, users, messages, encrypted variables | no |
| Redis | Railway Redis 8.2 | Shared build/job queue (DB 1) and rate-limit counters (DB 2) | no |

**First login**

1. Wait for the first deploy to pass its healthcheck (`/health_check`); migrations and starter projects can take a few minutes.
2. Open the public URL and sign in as `admin` with the password in the Langflow service variable `LANGFLOW_SUPERUSER_PASSWORD`.
3. Add your model provider keys under *Settings → Global Variables*. They are encrypted with `LANGFLOW_SECRET_KEY`; never change that key after first boot.
4. New users who sign up stay inactive until an admin activates them (`LANGFLOW_NEW_USER_IS_ACTIVE=false`).

**Scaling workers:** raise `LANGFLOW_WORKERS` (about one per vCPU, 0.6–1 GB RAM each) and give the service more resources. Redis keeps builds consistent across workers. Keep `LANGFLOW_WORKERS × (pool_size + max_overflow)` from `LANGFLOW_DB_CONNECTION_SETTINGS` below about 90 connections.

**Embedding the chat widget:** add each website origin that embeds Langflow to `LANGFLOW_CORS_ORIGINS` (comma-separated).

**Custom domain:** add it to the Langflow service, then update `LANGFLOW_CORS_ORIGINS` and `LANGFLOW_MCP_BASE_URL` to the new origin.

**Pinning and upgrading:** the image is pinned to `1.12.5`. To upgrade, back up Postgres, change the tag on the Langflow service and redeploy; Langflow migrates the database on start. Do not roll back to an older version against a migrated database without restoring the backup. To change the component inventory, switch between `langflowai/langflow:base-{ver}`, `langflowai/langflow:{ver}` and `langflowai/langflow-all:{ver}`.

### Why Deploy Langflow on Railway?

Railway gives Langflow a public HTTPS URL, managed Postgres and Redis on a private network, and a persistent volume in a single project, so your flows, files and credentials survive redeploys. Scaling is a matter of raising workers and resources, and upgrades are a tag change with one-click rollback. This is a community template and is not affiliated with the Langflow project or its maintainers.
