# Notice

## Upstream software

| Software | License | Images used (publisher) |
|---|---|---|
| Langflow | MIT — Copyright (c) 2024 Langflow | `langflowai/langflow:1.12.5` (Langflow, Docker Hub) |
| PostgreSQL | PostgreSQL License | `ghcr.io/railwayapp-templates/postgres-ssl:18` (Railway) |
| Redis | RSALv2 / SSPLv1 / AGPLv3 (Redis 8 tri-license) | `redis:8.2` (Docker Official Image) |

## Verified against

| What | Upstream file | Ref | URL | Checked |
|---|---|---|---|---|
| Reference compose (Langflow + Postgres + config volume) | `docker_example/docker-compose.yml` | `v1.12.5` | https://github.com/langflow-ai/langflow/blob/v1.12.5/docker_example/docker-compose.yml | 2026-10-10 |
| Image user, default env (`LANGFLOW_AUTO_LOGIN=false`, host/port), profiles | `docker/build_and_push.Dockerfile`, `docs/docs/Deployment/docker-image-profiles.mdx`, `docs/docs/Deployment/deployment-docker.mdx` | `v1.12.5` | https://github.com/langflow-ai/langflow/tree/v1.12.5/docker | 2026-10-10 |
| Redis job queue, multi-worker guard, rate-limit storage | `docs/docs/Deployment/deployment-multi-worker.mdx`, `src/backend/base/langflow/__main__.py` | `v1.12.5` | https://github.com/langflow-ai/langflow/blob/v1.12.5/docs/docs/Deployment/deployment-multi-worker.mdx | 2026-10-10 |
| Settings names/defaults (paths, DB pool, cache, runtime, security, MCP) | `src/lfx/src/lfx/services/settings/groups/*.py` | `v1.12.5` | https://github.com/langflow-ai/langflow/tree/v1.12.5/src/lfx/src/lfx/services/settings/groups | 2026-10-10 |
| Health endpoints | `src/backend/base/langflow/api/health_check_router.py` | `v1.12.5` | https://github.com/langflow-ai/langflow/blob/v1.12.5/src/backend/base/langflow/api/health_check_router.py | 2026-10-10 |
| PostgreSQL ≥ 15 requirement | `src/backend/base/langflow/services/database/service.py` | `v1.12.5` | same repo | 2026-10-10 |
| Auth / superuser / CORS / secret key | `docs/docs/Develop/api-keys-and-authentication.mdx`, `docs/docs/Develop/environment-variables.mdx` | `v1.12.5` | https://docs.langflow.org/api-keys-and-authentication | 2026-10-10 |
| Production profile requirements | `docs/docs/Deployment/deployment-prod-best-practices.mdx` | `v1.12.5` | https://docs.langflow.org/deployment-prod-best-practices | 2026-10-10 |
| S3 storage implementation (for the open question) | `src/backend/base/langflow/services/storage/s3.py` | `v1.12.5` | same repo | 2026-10-10 |

## Adapted code

None. No files are copied; the template uses the published image unmodified and configures it through documented environment variables.

## Reference only (not copied)

- Marketplace templates `JMXEWp` ("Langflow"), `LcJLmZ` ("Langflow"), `UsJ1uB` ("Langflow Pre-releases"): no license; read for structure (image + Postgres + `LANGFLOW_DATABASE_URL`). No files or text copied.
- No `_upstream` repositories were provided for this app.

## Trademarks

"Langflow" and its logo belong to their respective owner. "Railway" belongs to Railway Corporation. This is a community template and is not affiliated with or endorsed by them.
