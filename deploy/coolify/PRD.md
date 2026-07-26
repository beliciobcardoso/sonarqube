# PRD — Deploy SonarQube (Custom Build) on Coolify

## Context

Custom fork of SonarQube Community Edition (source in this repo, v26.8) to evolve with proprietary features. Needs to build its own Docker image from source and run via docker-compose on Coolify.

## Licensing

Repo licensed under **LGPLv3** (`LICENSE.txt`). No `private/` submodule present → build produces **Community Edition** only. LGPLv3 permits modifying and running your own version; the obligation to share changes only applies if the binary is **distributed** to third parties — internal/self-hosted deployment doesn't trigger it. Legal review only recommended if the product is resold/distributed externally.

## Scope

Custom multi-stage Docker image build + docker-compose for deployment on Coolify, with a dedicated Postgres instance.

## Decisions

| # | Decision | Choice | Reason |
|---|---------|---------|--------|
| 1 | Image origin | Custom build from source | Fork with proprietary features, not served by the official image |
| 2 | Build strategy | Multi-stage Dockerfile (gradle build inside the image) | Simplicity — each Coolify deploy builds the zip itself, no external CI |
| 3 | Build stage | `eclipse-temurin:21-jdk-jammy` (Java 21 required, confirmed in `build.gradle`) | Matches the project's toolchain |
| 4 | Edition | Community Edition | No access to `private/` (commercial modules) |
| 5 | Runtime user | Non-root (uid 1000) | Embedded Elasticsearch refuses to run as root |
| 6 | Final runtime image | `eclipse-temurin:21-jdk-jammy` (full JDK, not JRE) | Elasticsearch's Java 21 entitlements system requires JDK-only modules (`jdk.attach`, `jdk.jlink`), missing from any JRE-only build. Confirmed empirically: a JRE-only runtime crash-loops Elasticsearch on boot |
| 7 | Database | PostgreSQL 16, own container in the compose stack | SonarQube requires Postgres 13+; 16 is the latest supported version |
| 8 | Database network | Internal network, no published port (`5432` not exposed) | Reduces attack surface |
| 9 | Persistence | Named volumes: `sonarqube_data`, `sonarqube_logs`, `sonarqube_extensions`, `postgres_data` | Coolify convention, integrates with platform backups |
| 10 | Elasticsearch bootstrap | `vm.max_map_count >= 262144` set **on the host** (outside the compose file) + `ulimits` (nofile/memlock) on the service | Per-container `sysctls` is inconsistent across managed environments |
| 11 | Public exposure | No manual port — proxy/TLS/domain via Coolify's native labels/FQDN | Avoids duplicating the proxy layer |
| 12 | Secrets/config | Env vars via Coolify UI (`.env` with `${VAR}` in the compose file) | Native platform flow, no secrets committed in plain compose |
| 13 | Memory sizing | SonarQube's default JVM auto-detect + `mem_limit` as a ceiling: **6GB** sonarqube / **1GB** postgres | Avoids premature over-engineering; tune later with real usage data |
| 14 | Healthcheck | `GET /api/system/status` (sonarqube) + `pg_isready` (postgres), with `depends_on: condition: service_healthy` | Avoids a false-positive "healthy deploy" while Elasticsearch is still initializing (1-2min) |
| 15 | Restart policy | `unless-stopped` on both services | Survives crashes/reboots without fighting a manual stop via Coolify |

## Deliverables

1. `deploy/coolify/Dockerfile.coolify` — multi-stage build (gradle → non-root runtime)
2. `deploy/coolify/docker-compose.yml` — `sonarqube` + `postgres` services, volumes, healthchecks, internal network
3. `deploy/coolify/.env.example` — expected variables (`SONAR_JDBC_URL`, `SONAR_JDBC_USERNAME`, `SONAR_JDBC_PASSWORD`, `POSTGRES_*`)
4. `deploy/coolify/README.md` — step-by-step: host `vm.max_map_count` setup, Coolify deployment, troubleshooting

## Known risks (out of scope for this PRD, future action by the user)

- A full Gradle build inside the Dockerfile is heavy (multiple Java modules) — Coolify deploy time will be high (potentially 10-15min+). Accepted at this stage; revisit external CI if it becomes a bottleneck.
- `vm.max_map_count` must be set manually on the Coolify host once — the compose file can't do this by itself.
- No additional plugin/license management — out of scope (pure CE).

## Acceptance criteria

- [x] `docker compose build` produces the image without errors
- [x] `docker compose up` brings up Postgres healthy, then SonarQube healthy (`/api/system/status` = `UP`)
- [x] Initial login works locally (admin/admin, forced password change) — validated via a temporary published port; Coolify FQDN flow itself not testable in this local environment
- [x] Host restart doesn't lose data (named volumes persist)
- [x] sonarqube container runs as uid 1000, not root
