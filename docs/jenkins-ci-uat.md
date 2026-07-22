# Jenkins CI/CD on the paysys UAT server

Reference documentation for the Jenkins instance running at `http://10.10.80.37:8080`. Captured on 2026-07-22 by directly reading Jenkins' on-disk config under `/var/lib/docker/volumes/jenkins_home/_data`.

Intended audience: anyone who needs to understand what the UAT Jenkins does, or rebuild it from scratch if the machine is lost.

---

## 1. Where Jenkins runs

- **Host:** `10.10.80.37` (RHEL 8.10, kernel 4.18)
- **Container:** `jenkins_server`, image `jenkins/jenkins:lts` (currently `8e6f48b11f82`, ~9 months old at time of writing)
- **Version:** Jenkins **2.541.2** LTS (from `java -jar /usr/share/jenkins/jenkins.war --version` inside the container)
- **Ports published on host:** `8080:8080` (web UI + API), `50000:50000` (agent connection port — currently unused, all builds run on the controller)
- **Docker socket:** the container has access to the host's Docker daemon (via `/var/run/docker.sock` bind or similar) — every job's build/deploy step uses `docker` commands, so this is a hard requirement
- **Persistent state:** Docker named volume `jenkins_home`, mounted at `/var/lib/docker/volumes/jenkins_home/_data` on the host. Contains all job configs, build history, plugins, credentials, and the encryption key ring. Size ~840 MB.

### Container run form (approximate)

The exact `docker run` line used to create the container is not recorded in-repo, but the running container's spec is consistent with the standard LTS-image invocation:

```bash
docker run -d \
  --name jenkins_server \
  --restart unless-stopped \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

The mount of `/var/run/docker.sock` is inferred from the fact that every job runs `docker build`, `docker stop`, `docker run` etc. directly, and the container has been observed to have UID/GID access to the socket.

---

## 2. Global configuration

From `config.xml` at the root of `jenkins_home`:

| Setting | Value |
|---|---|
| `numExecutors` | `2` — the controller runs up to 2 concurrent builds |
| `mode` | `NORMAL` |
| `useSecurity` | `true` |
| `authorizationStrategy` | `FullControlOnceLoggedInAuthorizationStrategy` with `denyAnonymousReadAccess=true` — every logged-in user has admin, anonymous cannot even read |
| `securityRealm` | `HudsonPrivateSecurityRealm` with `disableSignup=true` — local Jenkins user database, self-signup off |
| `quietPeriod` | `5` seconds — after SCM change, wait 5s before starting build (coalesces rapid pushes) |
| `scmCheckoutRetryCount` | `0` |
| `workspaceDir` | `${JENKINS_HOME}/workspace/${ITEM_FULL_NAME}` (default) |

No agents/clouds configured (`<nodes/>` directory does not exist). All 24 jobs run on the built-in `master` executor. With 2 executors and every job SCM-polling every minute, contention is real; see §7 for concerns.

### Views (tabs on the Jenkins landing page)

- `all` — default, everything
- `TRS` — `TRS Backend DEV`, `TRS Frontend DEV`
- `TCS` — `TCS Backend DEV`, `TCS Frontend DEV`
- `TRS Staging` — `Admin Service Staging`, `TRS Staging Backend`, `TRS Staging Frontend`

Other jobs are only visible in the `all` view.

---

## 3. Plugins

91 plugins are installed. The load-bearing ones for the current setup are:

**Pipeline / workflow:**
`workflow-aggregator`, `workflow-job`, `workflow-cps`, `workflow-durable-task-step`, `workflow-multibranch`, `pipeline-model-definition`, `pipeline-stage-view`, `pipeline-groovy-lib`, `pipeline-github-lib`, `pipeline-input-step`, `pipeline-milestone-step`

**Source control:**
`git`, `git-client`, `github`, `github-api`, `github-branch-source`, `scm-api`, `branch-api`

**Credentials:**
`credentials`, `credentials-binding`, `plain-credentials`, `ssh-credentials`

**Build utilities:**
`build-timeout`, `ws-cleanup`, `resource-disposer`, `timestamper`, `durable-task`

**Notifications:**
`email-ext`, `mailer`, `token-macro`

**Other:**
`cloudbees-folder`, `script-security`, `matrix-auth`, `config-file-provider`, `metrics`, `gradle`, `nodejs`

The full list is captured under `retesting/uat-server/jenkins-jobs/_plugins-full.txt` (untracked, evidence only).

Note: `docker` is not listed as a Jenkins plugin — every job invokes `docker` as a shell command via the host's Docker CLI, not through the `docker-workflow` plugin.

---

## 4. Credentials

Only one credential is configured, stored in `credentials.xml`:

| ID | Scope | Type | Purpose |
|---|---|---|---|
| `tazama-token` | GLOBAL | Username+password (or personal access token) | GitHub authentication for cloning `tazama-lf/*` and paysys-fork repos across all jobs |

The stored value is encrypted with Jenkins' `secret.key`. Every job's SCM block references `<credentialsId>tazama-token</credentialsId>`.

**Warning:** despite only one Jenkins-managed credential existing, several jobs bake **secondary secrets** into their pipeline script as environment variables (GitHub PATs, SFTP passwords, Keycloak client secrets, Gmail SMTP credentials, database passwords). These are not stored in Jenkins credentials — they live inside the job's `config.xml` and are visible to anyone with the workspace or Jenkins-config read access. This is a security issue flagged in §7.

---

## 5. Jobs

24 jobs total. Grouped here by role.

### 5.1 Core services (Tazama-lf)

Six jobs deploy the core tazama pipeline services. Every one follows the same shape: SCM-poll every minute → `docker build` → `docker stop` + `docker rm` the old container → `docker run` the new one.

#### `Admin-Service DEV`
- **Type:** Pipeline (`flow-definition`)
- **SCM:** `https://github.com/tazama-lf/admin-service.git`, branch `dev`, cred `tazama-token`
- **Trigger:** `hudson.triggers.SCMTrigger` spec `* * * * *`
- **Build:** injects an `.npmrc` with `NPM_TOKEN`, `docker build -t admin-service .`
- **Deploy:** `docker stop admin-service`, `docker rm`, then `docker run` on host port `3100` with environment vars for postgres (`10.10.80.37:5432`, user `postgres`, password `postgres`), Redis cluster (`10.10.80.37:16379`), Keycloak (`tcs` realm, `tcs-client`), NATS (`10.10.80.37:14222`), and a read-only bind mount of `/opt/connection-studio-FFF/backend/public.pem`
- **Publishers:** none
- **Enabled:** yes

#### `Admin Service Staging`
- **Type:** Pipeline
- **SCM:** same repo, branch `staging`, cred `tazama-token`
- **Trigger:** `* * * * *`
- **Build/Deploy:** same shape, deploys to container `admin-service-staging` on host port `3105`. `CONNECTION_STUDIO_URL` points at port `3000` instead of `3010`
- **Enabled:** no (disabled)

#### `Tazama Event Director`
- **Type:** Pipeline
- **SCM:** `https://github.com/tazama-lf/event-director.git`, branch `feat-paysys-poc` (**a paysys-specific branch, not `dev`**)
- **Trigger:** `* * * * *`
- **Build:** `docker build -t tazamaorg/event-director:rc .` (note: pushes to the `tazamaorg/*` local image namespace, mirroring the DockerHub layout the FSDT compose files expect)
- **Deploy:** container `tazama-ed-1`, joins the `tazama-core_default` docker network, connects to postgres (`configuration` DB), Redis, and NATS. Cache and APM disabled
- **Enabled:** yes

#### `Tazama TADP` (transaction-aggregation-decisioning-processor)
- **Type:** Pipeline
- **SCM:** `https://github.com/tazama-lf/event-adjudicator.git`, branch `dev`
- **Trigger:** `* * * * *`
- **Build:** `docker build -t tazamaorg/transaction-aggregation-decisioning-processor:rc .`
- **Deploy:** container `tazama-tadp-1` on `tazama-core_default` network. Reads `configuration`, `evaluation`, `raw_history` DBs. Forwards alerts to an investigation-service and to an interdiction-service NATS consumer
- **Enabled:** yes

#### `Tazama Typology Processor`
- **Type:** Pipeline
- **SCM:** `https://github.com/tazama-lf/typology-processor.git`, branch `feat-paysys-poc`
- **Trigger:** `* * * * *`
- **Build:** `docker build -t tazamaorg/typology-processor:rc .`
- **Deploy:** container `tazama-tp-1` on `tazama-core_default`. NATS producers/consumers wired to the interdiction service and tenant-scoped destinations. Env has a `SUPPRESS_ALERTS` toggle (currently `false`)
- **Enabled:** yes

#### `Nats Utilities DEV`
- **Type:** Pipeline
- **SCM:** `https://github.com/tazama-lf/nats-utilities.git`, branch `feat-paysys-fix-nats-bind`
- **Trigger:** `* * * * *`
- **Build:** `docker build -t nats-utilities .`
- **Deploy:** container `nats-utilities` on host port `4000`. Connects to `10.10.80.37:14222` (NATS). APM disabled
- **Enabled:** yes

### 5.2 Extensions and connected services

Fourteen jobs cover the TCS/TRS/CMS/DEAPI/DEMS extensions plus their staging variants.

#### `TCS Backend DEV`
- **SCM:** `https://github.com/tazama-lf/connection-studio.git`, branch `dev`, cred `tazama-token`
- **Trigger:** `* * * * *`
- **Build:** `docker build -t tcs-backend backend/`
- **Deploy:** container `tcs-backend` on host port `3010`. Env references the `tcs` postgres DB (at `10.10.80.37:5432`, distinct from the tazama-core postgres), NATS notification streams, SFTP consumer/producer (encrypted credentials in env vars), OpenSearch audit at `10.10.80.30:9200`, SMTP for email, and downstream integrations with admin-service (`3100`), Keycloak, DEMS (`3002`), DEAPI (`3001`). Session timeout 30 min
- **Enabled:** yes

#### `TCS Frontend DEV`
- **SCM:** same repo, branch `dev`
- **Trigger:** `* * * * *`
- **Build:** Vite-based frontend from `frontend/`
- **Deploy:** container `tcs-frontend` on host port `5173`. Points at TCS backend (`3010`) and DEAPI staging (`1969`)
- **Enabled:** yes

#### `TCS Frontend Staging`
- **SCM:** same repo, branch `feat-paysys-minor-fixes-755`
- **Trigger:** `* * * * *`
- **Deploy:** container `tcs-frontend-staging` on host port `5176`
- **Enabled:** yes

#### `TRS Backend DEV`
- **SCM:** `https://github.com/tazama-lf/rule-studio.git`, branch `dev`, cred `tazama-token`
- **Trigger:** `* * * * *`
- **Build:** backend Dockerfile
- **Deploy:** container `trs-backend` on host port `3006`. Authenticates against `10.10.80.17:3020`, references admin-service on staging port `3105`, OpenSearch audit at `10.10.80.30:9200`, mounts `/var/run/docker.sock` for TestContainers-based integration testing, and holds DockerHub credentials for spawning ephemeral rule-executer containers
- **Enabled:** no (disabled)

#### `TRS Frontend DEV`
- **SCM:** same repo, branch `dev`
- **Trigger:** `* * * * *`
- **Deploy:** container `mm-frontend` (model management frontend — TRS was rebranded to Model Management) on host port `5174`. Env points at mm-backend (`3005`), simulation sandbox (`3050`), NATS utilities (`4000`), and a simulation-endpoint on the core stack
- **Enabled:** yes

#### `TRS KASHIP TEST FRONTEND`
- **SCM:** same repo, branch `feat-paysys-fix-custom`
- **Deploy:** container `kaship-frontend` on host port `5177`
- **Enabled:** no

#### `TRS Staging Backend`
- **SCM:** same repo, branch `feat-paysys-complete`
- **Deploy:** container `trs-frontend` (note: name suggests frontend but the job is labeled backend — the config appears to have drifted) on host port `5175`
- **Enabled:** no

#### `TRS Staging Frontend`
- **Type:** manual "tail logs" pipeline for `trs-backend` container (not a build job despite the name — job naming has drifted)
- **Enabled:** yes

#### `CMS Backend`
- **SCM:** `https://github.com/tazama-lf/case-management-system.git`, branch `paysys/visualization` (**paysys-specific branch**)
- **Trigger:** `* * * * *`
- **Build:** Dockerfile in `backend/`
- **Deploy:** container `cms-backend` on host port `3090`. Wires to:
  - Postgres: `tazama_cms` DB and `cms_dwh_db` data warehouse DB
  - AI model endpoint: `http://10.10.80.16:8000/predict`
  - Flowable workflow engine: `http://10.10.80.16:8081/flowable-rest`
  - CouchDB evidence store: `http://10.10.80.16:5984/cms-evidence`
  - OpenSearch audit: `http://10.10.80.30:9200`
  - Voila notebook server and a Gold-lakehouse API
  - SMTP (Gmail) for notifications — hardcoded credentials in env
- **Enabled:** yes

#### `CMS Frontend`
- **SCM:** same repo, branch `paysys/visualization`
- **Deploy:** container `cms-frontend` on host port `5179` (mapped from internal `5175`). API URL points at `cms-backend` at `10.10.80.37:3090`
- **Enabled:** yes

#### `DEAPI DEV`
- **SCM:** `https://github.com/tazama-lf/data-enrichment-service.git`, branch `staging` (note: DEV job builds `staging` branch — apparent misnaming)
- **Trigger:** `* * * * *`
- **Deploy:** container `data-enrichment-service` on host port `3001`. Wires TCS postgres, NATS streams (`config.notification.response` producer, `config.notification` consumer, plus an enrichment stream), Redis on `6379`, Elastic APM at `10.10.80.17:8200`, Tazama auth at `10.10.80.17:3020`. Nightly enrichment cron every 30 minutes; 150 MB payload limit, 1000 batch size
- **Enabled:** yes

#### `DEAPI Staging`
- **SCM:** same repo, branch `staging`
- **Deploy:** container `data-enrichment-service-staging` on host port `1969`
- **Enabled:** yes

#### `DEMS DEV`
- **Note:** the job labeled "DEMS DEV" in the directory listing is actually the log-tailing pipeline (see §5.3); no dedicated DEMS build job exists in this Jenkins. The `event-monitoring` container that was seen running (§ pre-purge survey) is probably deployed by some other job or by hand

#### `Simulation Sandbox`
- **SCM:** `https://github.com/tazama-lf/rule-studio-devtestops.git`, branch `dev`
- **Trigger:** `* * * * *`
- **Build:** `docker build -t simulation-sandbox .`
- **Deploy:** container `simulation-sandbox` on host port `3050`. Uses a GitHub template repo (`psl-copilot/rule-template`) to generate rule scaffolding via git operations against a "Git-Core-Tech" org. NATS at `10.10.80.37:14222`. Holds tenant-based encryption keys and a **second GitHub API token in-line** (separate from `tazama-token`)
- **Enabled:** yes

### 5.3 Log-tail and maintenance jobs

Four utility jobs. All are pipelines with no SCM, triggered manually by clicking "Build Now."

#### `DEAPI Logs`
- Runs `docker logs --tail 50 data-enrichment-service`, prints to console
- 15-minute timeout
- Enabled

#### `Logs Model Management`
- Runs `docker logs --tail 50 mm-backend`
- 15-minute timeout
- Enabled

#### `Logs TCS`
- Runs `docker logs --tail 50 tcs-backend`
- 15-minute timeout
- Enabled

#### `Space Reclamation Pipeline`
- Three-stage housekeeping pipeline:
  1. `docker image prune -af` — remove unreferenced images
  2. `docker container prune -f` — remove stopped containers
  3. `docker builder prune -af` — remove build cache
- Manual trigger only, no schedule
- Enabled

---

## 6. Rebuild-from-scratch procedure

If the machine is lost and someone has to reconstruct this Jenkins from the ground up.

### 6.1 Prerequisites on the target host

- Docker Engine ≥ 20 with the Compose plugin
- Network access to all the fixed IPs the jobs assume (see §7)
- Firewall opened for host ports `8080` (Jenkins UI) and `50000` (agent — future use)
- A backup of the `jenkins_home` volume from a healthy source, or a willingness to re-configure everything from this document

### 6.2 Start the container

```bash
docker volume create jenkins_home
docker run -d \
  --name jenkins_server \
  --restart unless-stopped \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

On first boot, `docker logs jenkins_server` prints the initial admin password (`initialAdminPassword`). Use it to open the setup wizard at `http://<host>:8080`.

### 6.3 Restore from backup (fastest path)

If you have a tar of the `jenkins_home` volume from a working Jenkins:

```bash
docker stop jenkins_server
docker run --rm -v jenkins_home:/data -v $(pwd):/backup alpine \
  tar xzf /backup/jenkins_home.tar.gz -C /data --strip-components=1
docker start jenkins_server
```

All 24 jobs, plugin set, credential vault, and user accounts come back verbatim.

### 6.4 Rebuild from scratch (no backup)

If you don't have a backup, work through the setup wizard:

1. **Complete the setup wizard** — install "suggested plugins" plus the ones listed in §3 that aren't in the suggested set (Docker-related, `github-branch-source`, `pipeline-github-lib`, etc.).
2. **Create the admin user** — Jenkins' private security realm, no signup.
3. **Configure global security**:
   - Enable `FullControlOnceLoggedInAuthorizationStrategy`
   - Deny anonymous read access
   - Disable user signup
4. **Add the `tazama-token` credential**:
   - Manage Jenkins → Credentials → System → Global credentials → Add credentials
   - Kind: Username with password (or Secret text if using a bare PAT)
   - Username: your GitHub username or bot account name
   - Password/Token: a GitHub personal access token with `repo` scope
   - ID: `tazama-token`
5. **Set the number of executors to 2** — Manage Jenkins → Nodes → Built-in Node → Configure
6. **Recreate each job**. The most reliable method is to copy the `config.xml` from the archive under `retesting/uat-server/jenkins-jobs/<job name>/config.xml` (captured on 2026-07-22) directly into `${JENKINS_HOME}/jobs/<job name>/config.xml`, then reload Jenkins config via Manage Jenkins → Reload Configuration from Disk.
7. **Recreate the three custom views** (`TRS`, `TCS`, `TRS Staging`) either by hand or by copying their `<listView>` sections from the archive of `${JENKINS_HOME}/config.xml`.
8. **Verify** — open each job, click "Build Now", confirm the SCM checkout succeeds. If `tazama-token` isn't reachable to a repo, GitHub returns 401 and the job fails at the checkout stage.

### 6.5 Non-Jenkins prerequisites the jobs assume

The jobs are not self-contained — they publish over the host Docker socket into containers on other well-known networks. Before builds actually deploy usefully, these must exist:

- **Docker network `tazama-core_default`** — created by the FSDT `docker compose -p tazama-core up` (§5.1 Core service jobs join this network). Named after the compose project (`-p tazama-core`), so the network name changes if the project name does. If FSDT isn't running, the core-service jobs succeed at the build step but fail at deploy because there is no shared network.
- **Postgres** at `10.10.80.37:5432` accepting `postgres/postgres` — the `tcs` container in the FSDT extensions stack, or an equivalent. This is different from the FSDT-managed `tazama-postgres` at `15432`.
- **NATS** at `10.10.80.37:14222` — provided by `tazama-nats-1` when FSDT is running.
- **Redis / Valkey** at `10.10.80.37:16379` — provided by `tazama-valkey-1` when FSDT is running.
- **Keycloak** at `10.10.80.33:8080` (external), realm `tcs`, client `tcs-client`.
- **Auth service** at `10.10.80.17:3020`.
- **Elastic APM** at `10.10.80.17:8200`.
- **OpenSearch** at `10.10.80.30:9200`.
- **CMS auxiliary services** at `10.10.80.16` — AI model on `8000`, Flowable on `8081`, CouchDB on `5984`.

Without those, the jobs still build images, but the containers they run will fail their health checks and log connection errors.

---

## 7. Concerns and recommended follow-ups

The current setup is functional but has real hygiene issues. None of these are the responsibility of a rebuild — they should be tracked as separate improvements.

### 7.1 Security

- **Secrets in job configs.** Every job that references SMTP, DockerHub, SFTP, or a second GitHub token embeds those secrets as plaintext environment variables in the pipeline script. They should be moved into Jenkins credentials and injected via `withCredentials { }` bindings.
- **Database credentials.** `postgres/postgres` hardcoded across many jobs. Should be per-service accounts with rotated passwords stored as Jenkins credentials.
- **Docker socket exposure.** Jenkins mounts the host's `/var/run/docker.sock`, which is functionally equivalent to root on the host. Any job with a `sh` step can `docker exec` into any container or spawn privileged containers. Fine for a paysys-only UAT box, unacceptable for anything shared.
- **Anyone with a Jenkins login has admin.** `FullControlOnceLoggedInAuthorizationStrategy` is coarse; even read-only observers get global admin. Consider `matrix-auth` for role separation.

### 7.2 Reliability

- **Every-minute SCM polling on 16+ jobs**, all running on 2 executors, all hitting the same GitHub API rate limit under a single credential. Under load, jobs will queue for tens of seconds. Consider replacing polling with GitHub webhooks (`ghprbPlugin` or `github-branch-source`'s scan-and-notify).
- **No email or Slack notifications.** Build failures are only visible if you go looking. `email-ext` is installed but no jobs configure recipients.
- **No archived artifacts, no test reports.** If a build breaks in a way that isn't logged during the run, there's no post-mortem trail.
- **Every deploy is `docker stop && docker rm && docker run`** with no health check between stop and run. If the new container fails to start, there's a service outage for the polling interval until the next build attempt.

### 7.3 Configuration drift

- **Job names and container names have diverged.** `TRS Staging Frontend` is actually a log-tail job. `TRS Frontend DEV` builds a `mm-frontend` container. `DEAPI DEV` builds the `staging` branch. Someone should rename either the jobs or the artifacts so they match again.
- **Multiple `feat-paysys-*` branches feeding "DEV" jobs** — `feat-paysys-poc`, `feat-paysys-fix-nats-bind`, `feat-paysys-fix-custom`, `feat-paysys-complete`, `feat-paysys-minor-fixes-755`, `paysys/visualization`. These branches are essentially the paysys-facing production pinning. When they drift from upstream `dev`, the UAT machine's behavior stops matching what FSDT contributors see.
- **Hardcoded IPs everywhere.** `10.10.80.37` (self), `10.10.80.17` (auth/APM), `10.10.80.16` (CMS aux), `10.10.80.30` (OpenSearch), `10.10.80.33` (Keycloak). If any of these moves, half the jobs need editing. Consider a Jenkins Global Environment section that centralizes them.
- **Docker network name is coupled to the FSDT compose project name.** The three core-service jobs (TADP, Event Director, Typology Processor) hardcode `--network tazama-core_default` in their deploy step. That network name is only correct while FSDT is deployed with `docker compose -p tazama-core up`. On 2026-07-22 the same jobs referenced `--network tazama_default` (the network produced by the older `-p tazama` invocation) and failed with `network tazama_default not found` after the FSDT redeploy switched to `-p tazama-core`. Fixed by editing the three job configs to match. Same class of coupling remains: if the FSDT launcher's `-p` value ever changes again, these three job configs need to change with it. Related upstream doc drift: FSDT `core/README.md` still documents `-p tazama` in one code sample as of this writing — worth a follow-up PR.
- **Jenkins assumes a single-machine deployment of FSDT.** The `--network tazama-core_default` join in the three core-service jobs works because everything Jenkins deploys — including Jenkins itself — runs on the same Docker daemon as the FSDT core stack. In FSDT's canonical three-machine topology (Server A = core, Server B = extensions, Server C = BIAR — see `docs/FSDT-for-deployment-guide.md`), a Jenkins runner sitting on any one of those servers can only reach the local network. If the UAT ever splits into three machines, the CI has to either (a) move to Server A and stay single-network-attach, (b) stop using `--network` and reach core services over published host ports on Server A's private IP, or (c) run one Jenkins per server. The current configuration only works because dev-uat is a single-box deployment.

### 7.4 Disabled jobs

Four jobs are disabled: `Admin Service Staging`, `TRS Backend DEV`, `TRS KASHIP TEST FRONTEND`, `TRS Staging Backend`. Reason not documented in-config. Before deleting them, someone should check whether they're paused for a reason or abandoned and clean up either way.

---

## 8. Evidence archive

The full snapshot used to write this document is under `retesting/uat-server/jenkins-jobs/` on the paysys workstation (not committed):

- One directory per job, each containing the job's `config.xml`
- `_plugins-full.txt` — full plugin list with versions
- (No `builds/`, `workspace/`, or `nextBuildNumber` — these are large and rebuildable)

This archive plus the `jenkins_home` volume tarball (also under `retesting/uat-server/backups/` if captured) are sufficient to reconstruct the Jenkins side of the UAT deployment.
