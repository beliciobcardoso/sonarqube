# Deploy SonarQube (custom build) on Coolify

See decisions and acceptance criteria in [`PRD.md`](./PRD.md).

## 1. Host prerequisite (one-time)

Elasticsearch (embedded in SonarQube) requires `vm.max_map_count >= 262144`. Set this
**on the host** where Coolify runs the containers, not in the compose file:

```bash
sudo sysctl -w vm.max_map_count=262144

# Persist across reboots:
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

Without this, the SonarQube container crashes on boot with:
`max virtual memory areas vm.max_map_count [65530] is too low`.

## 2. Coolify configuration

### 2.1 Connect the source

1. In Coolify, go to **Sources** and confirm a Git source (GitHub App, GitLab, or
   generic Git via deploy key) is already connected to this repository. Add one if not —
   **Settings → Sources → + Add**.
2. Note which **Server** (and, if used, **Project/Environment**) you want this
   resource deployed to — you'll pick it in the next step.

### 2.2 Create the resource

1. **Project → + New Resource → Docker Compose**.
2. Pick the Git source from 2.1, select this repository, and set the **branch**
   to deploy (e.g. `master` or your release branch — not necessarily `dev`).
3. Set **Base Directory** to `deploy/coolify` and **Docker Compose Location** to
   `docker-compose.yml` (i.e. Coolify should resolve
   `deploy/coolify/docker-compose.yml`). The build `context: ../..` inside the
   compose file still correctly reaches the repo root regardless of this setting.
4. Leave **Build Pack** as Docker Compose (auto-detected once the file is found).

### 2.3 Environment variables

Open the resource's **Environment Variables** tab and add (values, not
placeholders — use `.env.example` only as a reference for the names):

- `POSTGRES_DB`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD` — generate a real secret here, don't reuse the local test value

Do **not** commit a real `.env` to the repo — Coolify injects these directly into
the compose file's `${VAR}` interpolation at build/deploy time.

### 2.4 Networking

1. Leave the `sonarqube` service **without a published port** (already the case
   in `docker-compose.yml`) — Coolify's proxy handles this via the FQDN it assigns.
2. In the resource's **Domains** section, either accept the auto-generated
   `*.sslip.io`-style FQDN for a quick test, or set your own domain (Coolify
   provisions Let's Encrypt TLS automatically once DNS points at the server).
3. Confirm the FQDN targets the `sonarqube` service on container port `9000`
   (Coolify infers this from the compose file's `EXPOSE`/service definition;
   double-check it if multiple services are exposed).

### 2.5 Host prerequisite reminder

Steps 2.1-2.4 configure the resource, but the host itself still needs the
`vm.max_map_count` change from **Section 1** applied once, before the first
deploy — Coolify doesn't do this for you (it's a host `sysctl`, out of any
container's control).

### 2.6 Deploy

1. Click **Deploy**. Watch the build logs — the Gradle build stage is the long
   part (see Section 3).
2. Once containers are up, check **Logs** for the `sonarqube` service, and/or
   the resource's health indicator, for the `healthy` status described in
   Section 3.
3. Open the FQDN from 2.4 in a browser and confirm the SonarQube login screen
   loads (`admin` / `admin`, forced password change).

## 3. What to expect on the first deploy

- The image build runs the full Gradle build (`:sonar-application:build`) — it can
  take a while (multiple Java modules). This isn't a configuration issue, it's the
  cost of building from source on every deploy.
- Container boot: Postgres becomes `healthy` first, then SonarQube comes up
  (Web + Compute Engine + Elasticsearch) — the healthcheck only reports healthy
  once `/api/system/status` returns `"status":"UP"` (can take 1-2min).
- Initial login: `admin` / `admin`, forced password change on first access.

## 4. Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| Container restart-loops citing `vm.max_map_count` | Step 1's setting wasn't applied on the host | Run `sysctl -w` on the host, not inside the container |
| `Elasticsearch did not exit normally` / silent crash | Out of memory (`mem_limit` too low) | Check `docker stats`, consider raising the `sonarqube` service's `mem_limit` in the compose file |
| Build fails from lack of memory/time on Coolify's build runner | Build runner is under-resourced for Gradle | Increase Coolify's build runner resources, or revisit building outside the Dockerfile (out of scope for this PRD) |
| SonarQube can't resolve Postgres | Wrong service name or env var not set | Confirm `SONAR_JDBC_URL=jdbc:postgresql://postgres:5432/<db>` and that `postgres` is healthy |

## 5. Files

- `Dockerfile.coolify` — multi-stage build (Gradle → non-root runtime). Non-default name to avoid colliding with the repo root's generic `.dockerignore`
- `Dockerfile.coolify.dockerignore` — ignore-file override, **must sit in the same directory as the Dockerfile** (not the build context root) for BuildKit to resolve it correctly
- `docker-compose.yml` — `sonarqube` + `postgres` services, volumes, healthchecks
- `.env.example` — expected variables
- `PRD.md` — decisions and acceptance criteria
