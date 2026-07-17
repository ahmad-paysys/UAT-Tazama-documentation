# Rule Building Flow and Work — End-to-End Setup Guide

**Audience:** product engineers, product owners, platform engineers, and anyone tasked with standing up the Tazama Rule Studio + DevTestOps + Rule Executer stack so that a fintech's rule authors can go from "empty GitHub org" to "a rule running in a container on a server, subscribed to NATS, processing transactions".

**Scope:** this is the setup-and-operate guide, not the code guide. It tells you every environment variable to set, every GitHub secret/variable to create, every prerequisite to install on your self-hosted runner, and the end-to-end order of operations. It also flags the traps we've discovered while reviewing the three PRs currently open on this stack (`rule-studio-example#13`, `rule-studio#132`, `rule-studio-devtestops#5`).

**Repos in scope** (all under `tazama-lf`):

| Repo | Role |
|------|------|
| [`rule-studio`](https://github.com/tazama-lf/rule-studio) | Full-stack web app (React + NestJS/Fastify). Where rule authors write rules, simulate them, promote them, and view test reports. |
| [`rule-studio-devtestops`](https://github.com/tazama-lf/rule-studio-devtestops) | Fastify REST API that translates UI actions (bootstrap / populate / promote) into calls to the GitHub REST API and to git itself. |
| [`rule-studio-example`](https://github.com/tazama-lf/rule-studio-example) | GitHub *template repository*. Every new rule gets its own repo cloned from this template. Carries the four GitHub Actions workflows that drive build → publish → deploy. |
| [`rule-executer`](https://github.com/tazama-lf/rule-executer) | Generic runtime. Wraps a rule npm package, connects to NATS, subscribes to `sub-rule-<name>@<version>`, executes the rule, publishes to `pub-rule-<name>@<version>`. |
| [`frms-coe-lib`](https://github.com/tazama-lf/frms-coe-lib) | The core library both the rule and the rule-executer depend on. |

---

## 1. Architecture at a glance

```
  ┌─────────────────────┐
  │   Rule Author (UI)  │
  │  browser → frontend │
  └──────────┬──────────┘
             │ REST
             ▼
  ┌──────────────────────┐         ┌──────────────────────┐
  │   rule-studio        │  REST   │  rule-studio-        │
  │   backend (Nest/     │────────▶│  devtestops          │
  │   Fastify)           │         │  (Fastify)           │
  └──────┬───────────────┘         └────────┬─────────────┘
         │                                  │
         │ NATS (simulation)                │ GitHub REST API + git CLI
         ▼                                  ▼
  ┌─────────────────────┐         ┌──────────────────────┐
  │ nats-utilities REST │         │  GitHub organisation │
  │ bridge → NATS       │         │  (rule-<id> repos    │
  └──────┬──────────────┘         │  from template)      │
         │                        └───────────┬──────────┘
         │ pub/sub                            │
         │                                    │ push triggers Actions
         ▼                                    ▼
  ┌──────────────────────────────────────────────────┐
  │  rule-executer container(s) on self-hosted       │
  │  runner (per rule, per version). Talks to        │
  │  PostgreSQL (raw_history, configuration,         │
  │  event_history), Valkey/Redis, NATS.             │
  └──────────────────────────────────────────────────┘
```

Message shape on the wire: each deployed rule container subscribes to `sub-rule-<RULE_NAME>@<RULE_VERSION>` and replies on `pub-rule-<RULE_NAME>@<RULE_VERSION>`. Simulation from the UI publishes to the same subject via the `nats-utilities` HTTP bridge.

---

## 2. Prerequisites — the "shopping list" before you touch a keyboard

### 2.1 Human/organisational

- **Admin of a GitHub organisation.** You will create secrets, variables, and repositories at org level. If you are working inside `tazama-lf` you likely have this; if you are standing up a tenant fork (`psl-copilot`, `paysys`, etc.) you need Owner on that org.
- **Docker Hub organisation** *(optional, for simulation image browsing only — see §6.5).* The current deploy pipeline does **not** push images to Docker Hub. Only the simulation studio reads Docker Hub to list rule images that already exist there.
- **AES-256-CBC key material.** You need a 32-byte key and a 16-byte IV to encrypt the GitHub PAT at rest for `rule-studio-devtestops`. Generate them once and store them in a secrets vault; you will use them to encrypt the token and then to configure both the encryption/decryption tools and the DevTestOps service.

### 2.2 Servers/infrastructure

- **One or more self-hosted GitHub Actions runners.** These are the machines where rule containers actually run. Each runner needs:
  - Docker daemon with BuildKit support (Docker 20.10+; verify with `docker buildx version`).
  - `git` on `$PATH`.
  - Node.js 20+ (the `deploy.yml` workflow installs 20.x via `actions/setup-node@v4`, but the runner OS itself needs the toolchain to install successfully).
  - Network reachability to PostgreSQL, NATS, and Redis/Valkey (see §5.5 for the ports).
  - A GitHub Actions runner label that matches your rule repos' `RUNNER_LABEL` variable — see §5.4.
- **PostgreSQL** with three databases: `raw_history`, `configuration`, `event_history`. The `rule-executer` container expects to connect to each of them.
- **NATS server** (JetStream not strictly required for rule-only mode, but required for full simulation). The `rule-executer` connects with `STARTUP_TYPE=nats` and `SERVER_URL=<host>:14222` by default.
- **Redis / Valkey** for shared state (`REDIS_HOST=<host>`, `REDIS_PORT=6379`).

### 2.3 Tools on your workstation

- `gh` (GitHub CLI) authenticated to your organisation.
- `openssl` (for token encryption).
- `docker` + `docker compose` (for running Rule Studio locally).
- Node.js 20+ (for running the DevTestOps service locally or for token encryption scripts).

---

## 3. The single most important secret: `TAZAMA_TOKEN`

Every workflow in `rule-studio-example` and every git operation in `rule-studio-devtestops` authenticates to GitHub using a Personal Access Token that we consistently call **`TAZAMA_TOKEN`**. It is the linchpin. Get this wrong and nothing works.

### 3.1 What scopes it needs

If you are using a **classic PAT**:

- `repo` (full control of private repositories — needed to create repos in the org, push branches, read/write file contents via the API)
- `write:packages` (to publish and download `@<org>/*` npm packages from `npm.pkg.github.com`)
- `workflow` (to trigger and inspect GitHub Actions workflows via the API)

If you are using a **fine-grained PAT** (preferred), grant these repository permissions on the target organisation:

- **Administration** — Read & Write (to create repos)
- **Contents** — Read & Write (to push code, read files)
- **Actions** — Read & Write (to trigger and monitor workflows)
- **Packages** (on the *organisation* section, not per-repo) — Read & Write

### 3.2 Where to install it

Install `TAZAMA_TOKEN` at **both**:

1. **GitHub organisation → Settings → Secrets and variables → Actions → Secrets → New organization secret.** Scope: "Selected repositories" if you want to gate it to your rule repos + template repo; otherwise "All repositories". This is what the workflows in each rule repo consume.
2. **The DevTestOps service's `.env` file, as `GITHUB_TOKEN` — but encrypted.** See §3.3.

### 3.3 Encrypting the token for the DevTestOps service

`rule-studio-devtestops` requires the token in `.env` as `GITHUB_TOKEN=<AES_ENCRYPTED_GITHUB_TOKEN>`. The service decrypts it at startup using `ENCRYPTION_KEY` (32 bytes) and `ENCRYPTION_IV` (16 bytes).

Reference encryption snippet (Node, matching the format the service expects — check `rule-studio-devtestops/src/services/encryption.service.ts` or equivalent in your version of the repo before using this in production):

```javascript
// encrypt-token.js — run once, capture output
const crypto = require('node:crypto');
const key = Buffer.from(process.env.ENCRYPTION_KEY, 'utf8');   // 32 bytes
const iv  = Buffer.from(process.env.ENCRYPTION_IV, 'utf8');    // 16 bytes
const cipher = crypto.createCipheriv('aes-256-cbc', key, iv);
const encrypted = Buffer.concat([cipher.update(process.env.PLAIN_TOKEN, 'utf8'), cipher.final()]);
console.log(encrypted.toString('base64'));
```

```bash
ENCRYPTION_KEY='<32-char string>' \
ENCRYPTION_IV='<16-char string>' \
PLAIN_TOKEN='ghp_xxx…' \
  node encrypt-token.js
# → paste result into GITHUB_TOKEN in .env
```

Traps:

- **Key/IV length mismatch is a common cause of "service crashes on startup with a decrypt error".** `ENCRYPTION_KEY` must be exactly 32 bytes, `ENCRYPTION_IV` exactly 16 bytes. `openssl rand -hex 16` produces 32 chars (16 bytes) — you want `openssl rand -hex 16` for the IV (32 hex chars = 16 bytes) and `openssl rand -hex 32` for the key (64 hex chars = 32 bytes). Decide up front whether you're storing them as UTF-8 or hex and be consistent.
- **You must rotate the encrypted token whenever the underlying PAT is regenerated.**

---

## 4. Setting up `rule-studio-devtestops`

This is the service that translates rule-studio UI actions into GitHub operations. It runs as a container (see its `Dockerfile`) or as a Node process.

### 4.1 Environment variables

All required unless marked otherwise. Set them in the service's `.env` file (or inject via your orchestrator).

| Variable | Set where | Description |
|----------|-----------|-------------|
| `PORT` | `.env` | Service HTTP port; default `3050`. |
| `HOST` | `.env` | Bind address; default `0.0.0.0`. |
| `NODE_ENV` | `.env` | `development` / `production` / `test`. |
| `LOG_LEVEL` | `.env` | `debug` / `info` / `warn` / `error`. |
| `GITHUB_API_URL` | `.env` | Always `https://api.github.com` for github.com; use GHES URL for GitHub Enterprise Server. |
| `GITHUB_TEMPLATE_OWNER` | `.env` | The org/user that owns the template repo, e.g. `tazama-lf`. |
| `GITHUB_TEMPLATE_REPO` | `.env` | Template repo name, e.g. `rule-studio-example`. |
| `GITHUB_BRANCH` | `.env` | Branch of the template repo to clone from, e.g. `main`. **NOTE:** in older versions this was named `GITHUB_DEFAULT_BRANCH` — the rename lands in PR #5 (`rule-studio-devtestops#5`). If you are running a pre-#5 build, use the old name. |
| `GITHUB_TEST_REPORT_PATH` | `.env` | Where in each rule repo the unit-test HTML lives; e.g. `reports/unit-tests/latest/index.html`. |
| `ENCRYPTION_KEY` | `.env` | 32-byte AES-256-CBC key (see §3.3). |
| `ENCRYPTION_IV` | `.env` | 16-byte AES-256-CBC IV (see §3.3). |
| `GITHUB_TOKEN` | `.env` | The AES-encrypted `TAZAMA_TOKEN` (see §3.3). |
| `GITHUB_ORG_NAME` | `.env` | The org in which per-rule repos will be **created**, e.g. `psl-copilot`. Can differ from `GITHUB_TEMPLATE_OWNER`. |
| `GITHUB_INIT_BRANCH` | `.env` | The branch name to use inside each newly-bootstrapped rule repo, e.g. `staging`. |
| `NATS_URL` | `.env` | NATS server URL, e.g. `nats://<NATS_HOST>:14222`. |
| `CERT_PATH_PRIVATE` | `.env` *(optional)* | Path to a private key PEM if the service signs JWTs. |
| `CERT_PATH_PUBLIC` | `.env` *(optional)* | Path to a public key PEM if the service validates JWTs. |

### 4.2 Deployment prerequisites specific to this service

Per the changes in PR `rule-studio-devtestops#5`:

- The Docker runner image must have **`git` installed** (`RUN apk add --no-cache git` in the Dockerfile's runner stage). If you build your own image and forget this, the new `simple-git`-based bootstrap flow will fail with `spawn git ENOENT`.
- The runtime must be able to **shell out to `git`** — `simple-git` uses `child_process`. Make sure your process supervisor doesn't sandbox `/usr/bin/git` away.

### 4.3 How to run it

Locally, from `rule-studio-devtestops/`:

```bash
npm ci
cp .env.sample .env   # then edit .env per §4.1
npm run dev           # or `npm run start` for production
```

Health check: `curl http://localhost:3050/api/health` — should return `200 OK`.

---

## 5. Setting up per-rule repos (via the `rule-studio-example` template)

Every rule you create becomes its own GitHub repo, cloned from `rule-studio-example`. That template carries four workflows in `.github/workflows/`:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `unit-test.yml` | push/PR to `staging` | Install deps from GitHub Packages, run Jest, generate HTML report, commit report back. |
| `publish.yml` | push to `dev`, manual dispatch | Auto-increment version if already published, build TypeScript, `npm publish` to `npm.pkg.github.com` as `@<org>/<repo>@<version>`. |
| `deploy.yml` | after `publish.yml` succeeds, manual dispatch | Clone `rule-executer -b dev`, patch its Dockerfile with the rule ID, install the freshly-published rule package, `docker build` with a BuildKit secret, `docker run` the container locally on the self-hosted runner with NATS/DB/Redis env vars. |
| `deploy-to-uat.yml` | push to `prod` | Same idea as `deploy.yml` but hardcoded to `runs-on: tazama-uat` and to `10.10.80.37` server IPs. **Currently out of sync with `deploy.yml` per PR #13 — see §7.1.** |

### 5.1 Where to put secrets and variables

There are two levels: **org-wide** (recommended for anything shared) and **per-repo** (for anything specific to a single rule).

Set at **GitHub organisation → Settings → Secrets and variables → Actions**:

**Organization secrets:**

| Secret | Used by | Purpose |
|--------|---------|---------|
| `TAZAMA_TOKEN` | all four workflows in every rule repo | PAT with the scopes from §3.1. This authenticates `git clone` of `rule-executer`, `npm ci` / `npm publish` against `npm.pkg.github.com`, and the BuildKit secret mount that the rule-executer Dockerfile consumes. |

**Organization variables** (or repository variables, see §5.4):

| Variable | Used by | Default (if unset) | Purpose |
|----------|---------|--------------------|---------|
| `RUNNER_LABEL` | `deploy.yml` | `server-35` | Label of the self-hosted runner where the rule container should run. |
| `SERVER_IP` | `deploy.yml` | `10.10.80.35` | Target server IP for PostgreSQL / NATS / Redis. |

### 5.2 Creating the template repo in your org (if not already there)

If you are running under `tazama-lf`, `rule-studio-example` already exists. If you are standing up a new tenant org:

```bash
gh repo create <your-org>/rule-studio-example \
  --template tazama-lf/rule-studio-example \
  --public

# or clone-and-push if you need to fork with modifications
```

Then, in the new template repo, mark it as a template: **Settings → Template repository → check**. (`rule-studio-devtestops` uses the *template* repo purely as a source to `git clone` from — the "template repository" checkbox is not strictly required for the new `simple-git`-based flow in PR #5, but the older `POST /repos/…/generate` flow did require it. Leave it on for safety.)

### 5.3 Creating your first rule repo

Once `rule-studio-devtestops` is running and `rule-studio` is running, this happens **automatically** from the UI: an author clicks "Create Rule", the frontend POSTs to devtestops' `/v1/bootstrap` endpoint, and the service:

1. Creates an empty repo `<GITHUB_ORG_NAME>/<ruleId>` via `POST /orgs/{org}/repos`.
2. Clones `<GITHUB_TEMPLATE_OWNER>/<GITHUB_TEMPLATE_REPO>` at branch `<GITHUB_BRANCH>` into a temp dir.
3. Renames the branch to `<GITHUB_INIT_BRANCH>` (e.g. `staging`) via `git branch -M`.
4. Pushes to the new empty repo.

You can invoke it manually for testing:

```bash
curl -X POST http://localhost:3050/v1/bootstrap \
  -H 'Content-Type: application/json' \
  -H 'tenant-token: <the plaintext PAT — devtestops encrypts internally>' \
  -H 'organization-name: psl-copilot' \
  -H 'x-init-branch: staging' \
  -d '{"ruleId":"001","ruleVersion":"1.0.0"}'
```

### 5.4 Setting `RUNNER_LABEL` and `SERVER_IP` per rule (or per environment)

You have two choices:

- **Org-wide defaults.** Set `RUNNER_LABEL` and `SERVER_IP` as **Actions variables** at the org level. Every rule repo inherits.
- **Per-repo overrides.** For a specific rule (e.g. a heavy-load rule you want to isolate), go to that repo → Settings → Secrets and variables → Actions → Variables → New repository variable, and set the values there. Repo values win over org values.

**Trap from PR #13:** if you leave both unset, the workflow defaults changed in PR #13 from `server-18`/`10.10.80.18` to `server-35`/`10.10.80.35`. Every rule repo that had been silently relying on the old defaults will start deploying to a different host after PR #13 merges. Set the org variables explicitly to whatever host you actually want, regardless of what the template's defaults are.

### 5.5 Self-hosted runner setup

On the machine that will host your rule containers:

1. **Register the runner** against your GitHub org (Settings → Actions → Runners → New self-hosted runner) with the label you configured in `RUNNER_LABEL` (e.g. `server-35`).
2. **Install Docker** (20.10+ for BuildKit) and start the daemon.
3. **Install git** (`apt install git` / `yum install git` / etc.).
4. **Verify BuildKit is on:**
   ```bash
   docker buildx version
   DOCKER_BUILDKIT=1 docker build --help | grep -- --secret
   ```
   You should see `--secret` documented.
5. **Ensure network reachability** from the runner to:
   - PostgreSQL on `<SERVER_IP>:15432` (three databases: `raw_history`, `configuration`, `event_history`).
   - NATS on `<SERVER_IP>:14222`.
   - Redis/Valkey on `<SERVER_IP>:6379`.

   Note that PR #13 sets these all to the same `$SERVER_IP`; if your PostgreSQL and NATS live on different boxes, split the workflow env block or use DNS to route (`db.internal`, `nats.internal`).

6. **Ensure `TAZAMA_TOKEN`'s registry access.** As a smoke test, on the runner:
   ```bash
   echo "//npm.pkg.github.com/:_authToken=$TAZAMA_TOKEN" > ~/.npmrc
   npm whoami --registry=https://npm.pkg.github.com
   # → should print your GitHub username, not "Unauthenticated"
   ```

### 5.6 Environment variables the deployed rule container itself uses

These are set by `deploy.yml` via `docker run -e …`. You do **not** set these anywhere yourself; they're derived from `$SERVER_IP` and hardcoded defaults in the workflow. Here they are for reference so you can debug when a container fails to start:

| Env var | Value in deploy.yml | Purpose |
|---------|--------------------|---------|
| `RULE_NAME` | `$RULE_ID` (repo name with `rule-` stripped) | Used to build NATS subject `sub-rule-<RULE_NAME>@<VERSION>`. |
| `RULE_VERSION` | `1.0.0` (hardcoded!) | See "Trap: RULE_VERSION is hardcoded" in §7. |
| `RAW_HISTORY_DATABASE` | `raw_history` | Postgres database name. |
| `RAW_HISTORY_DATABASE_HOST` | `$SERVER_IP` | |
| `RAW_HISTORY_DATABASE_PORT` | `15432` | |
| `RAW_HISTORY_DATABASE_USER` | `postgres` | |
| `RAW_HISTORY_DATABASE_PASSWORD` | `unused` | Yes, literally `unused` — implies `trust`/`peer` auth. Fix for production. |
| `RAW_HISTORY_DATABASE_CERT_PATH` | `/usr/local/share/ca-certificates/ca-certificates.crt` | |
| `CONFIGURATION_DATABASE*` | same shape as above | |
| `EVENT_HISTORY_DATABASE*` | same shape as above | |
| `STARTUP_TYPE` | `nats` | Tells rule-executer which transport to use. |
| `SERVER_URL` | `$SERVER_IP:14222` | NATS host:port. |
| `PRODUCER_STREAM` / `CONSUMER_STREAM` / `STREAM_SUBJECT` | empty | JetStream config; empty means use core NATS. |
| `ACK_POLICY` | `Explicit` | |
| `PRODUCER_STORAGE` | `File` | |
| `PRODUCER_RETENTION_POLICY` | `Workqueue` | |
| `REDIS_HOST` | `$SERVER_IP` | |
| `REDIS_PORT` | `6379` | |
| `REDIS_PASSWORD` | `redis-password` | Hardcoded — change in your fork of the workflow. |
| `REDIS_AUTH` | empty | |

---

## 6. Setting up `rule-studio` (the UI)

Two moving parts here: the backend and the frontend, deployed together via `docker-compose.yml` for local dev.

### 6.1 Backend environment variables (`backend/.env`)

Reproduced from `backend/.env.example`. All required unless marked.

| Variable | Purpose |
|----------|---------|
| `NODE_ENV` | `dev` / `production` |
| `MAX_CPU` *(optional)* | Node cluster limit. |
| `FUNCTION_NAME` | Service identifier (e.g. `model-management-backend`). |
| `PORT` | Backend HTTP port (default 3000; Swagger at `/api/docs`). |
| `TAZAMA_AUTH_URL` | Tazama auth service URL (e.g. `http://localhost:3020/v1/auth`). |
| `AUTH_PUBLIC_KEY_PATH` | Path to public key PEM used to validate JWTs from the auth service. |
| `ADMIN_SERVICE_URL` | Admin service URL. |
| `OPENSEARCH_NODE` | OpenSearch URL. |
| `OPENSEARCH_SSL_REJECT_UNAUTHORIZED` *(optional)* | `false` for local dev with self-signed certs. |
| `OPENSEARCH_USERNAME` / `OPENSEARCH_PASSWORD` | Elastic/Opensearch creds. |
| `ALLOWED_ORIGINS` *(optional)* | CORS. |
| `CRYPTO_SECRET_KEY` | Symmetric key for rule-related crypto operations. |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_SECURE` / `SMTP_USER` / `SMTP_PASS` / `SMTP_FROM_EMAIL` / `SMTP_FROM_NAME` | Mail server for approval-workflow notifications. |
| `DOCKERHUB_TOKEN` | PAT for Docker Hub API — **only used to list simulation images**, not to push (see §6.5). |
| `DOCKERHUB_USERNAME` | Docker Hub username. |
| `DOCKERHUB_NAMESPACE` | Docker Hub namespace (`pslcopilot` in reference config). |
| `DOCKER_HOST` *(optional)* | e.g. `unix:///var/run/docker.sock`. |
| `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE` *(optional)* | For testcontainers-based simulation. |
| `TESTCONTAINERS_HOST_OVERRIDE` *(optional)* | e.g. `host.docker.internal` on macOS. |
| `TESTCONTAINERS_RYUK_DISABLED` *(optional)* | `true` for local dev. |
| `DEBUG` *(optional)* | Debug pattern. |
| `ENCRYPTION_KEY` / `IV_LENGTH` | For internal encryption. |
| `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD` *(optional)* | Redis/Valkey for shared state. |
| `POSTGRES_HOST` / `POSTGRES_PORT` / `POSTGRES_DB` / `POSTGRES_USER` / `POSTGRES_PASSWORD` | Postgres for rule metadata (via docker-compose `env_file`). |

### 6.2 Frontend environment variables (`frontend/.env`)

Vite build-time env — must be set before `npm run build` because they get inlined.

| Variable | Purpose |
|----------|---------|
| `VITE_ALLOWED_HOSTS` *(optional)* | Vite dev server allowed hosts (`all` for any). |
| `VITE_CRYPTO_KEY` | Client-side encryption key. |
| `VITE_APP_NAME` / `VITE_APP_VERSION` *(optional)* | Display strings. |
| `VITE_API_URL` | Backend API URL (e.g. `http://localhost:3005`). |
| `VITE_SANDBOX_API_URL` | Simulation studio API URL (may be same as backend). |
| `VITE_NATS_API_URL` | **Critical for simulation.** URL of the `nats-utilities` REST bridge that publishes to NATS. Rule-only simulation posts to `${VITE_NATS_API_URL}/natsPublish`. |
| `VITE_SIMULATION_ENDPOINT` *(optional)* | Extra simulation endpoint. |
| `VITE_DEMS_ENDPOINT` | URL of the DEMS end-to-end simulation service. |
| `VITE_ADMIN_ENDPOINT` | Admin service URL. |

### 6.3 Bringing it up locally

```bash
git clone git@github.com:tazama-lf/rule-studio.git
cd rule-studio
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
# Edit both .env files per §6.1 and §6.2
docker compose up -d          # brings up Postgres, NATS, Valkey, backend, frontend
```

Backend at `http://localhost:3005`, frontend at `http://localhost:5174`, Swagger at `http://localhost:3005/api/docs`.

### 6.4 The simulation flow (rule-only mode) — the "does anything actually route?" test

This is the fastest way to verify the whole stack is wired correctly.

1. Have `rule-studio` up.
2. Have at least one rule deployed via `deploy.yml` — you should see a container `docker ps` on your runner named after the rule (e.g. `rule-001`).
3. In Rule Studio UI, open a rule in the Rule Editor, switch to the Simulation tab, choose "read-only" mode, paste a valid transaction payload, click Run.
4. Under the hood, the frontend POSTs to `${VITE_NATS_API_URL}/natsPublish` with:
   ```json
   {
     "functionName": "",
     "awaitReply": true,
     "destination": "sub-<rule_config_id>",
     "consumer": "pub-<rule_config_id>",
     "message": { ...your payload... }
   }
   ```
5. `nats-utilities` publishes to `destination`, waits for a reply on `consumer`, and returns the result to the UI.

**Trap from PR #132.** The `destination` / `consumer` format in the code is `sub-<rule_config_id>` / `pub-<rule_config_id>` — but the *rule-executer* container is subscribed to `sub-rule-<RULE_NAME>@<RULE_VERSION>` (with the `rule-` prefix). Unless `rule_config_id` in your production data already reads `rule-<name>@<version>`, the subjects will not match and simulation will time out. See the PR-132 review in `pull-requests/TRS-PR-132.md` for the full analysis.

### 6.5 Where does Docker Hub fit in? (Short answer: not deployment.)

The user brief for this document asked how to deploy rules "on the servers and on dockerhub as well." The current pipeline does **not** push rule images to Docker Hub. Here's the precise picture:

- **`deploy.yml`** runs `docker build -t $ORG/$RULE_NAME:latest` on the self-hosted runner. It then `docker run`s that image immediately, on the same runner. The image never leaves the runner's local Docker daemon. There is no `docker push`.
- **Docker Hub credentials in `rule-studio` backend** (`DOCKERHUB_TOKEN`, `DOCKERHUB_USERNAME`, `DOCKERHUB_NAMESPACE`) are used **only by the simulation studio** to query Docker Hub's public API (`https://hub.docker.com/v2/repositories/…`) and list what rule images exist there for simulation purposes. They are not deployment credentials.

If you want rule images on Docker Hub for distribution across sites, you need to **add a Docker Hub push step to `deploy.yml`** — this is not currently in the template. Something like:

```yaml
- name: Log in to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- name: Push image to Docker Hub
  run: |
    docker tag "$ORG/$RULE_NAME:latest" "${{ secrets.DOCKERHUB_NAMESPACE }}/$RULE_NAME:latest"
    docker push "${{ secrets.DOCKERHUB_NAMESPACE }}/$RULE_NAME:latest"
```

Add `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` as org-level GitHub Actions secrets (Docker Hub PAT with `Public Repo Write` scope). The `docker/login-action` action is maintained by Docker and does not persist credentials to disk.

If instead you want images on **GitHub Container Registry** (`ghcr.io`) — which is closer to what the rest of the pipeline uses — replace the login/push with `docker/login-action` pointing to `registry: ghcr.io`, `username: ${{ github.actor }}`, `password: ${{ secrets.TAZAMA_TOKEN }}`, and push to `ghcr.io/$ORG/$RULE_NAME:latest`. This reuses the token you already have.

---

## 7. Traps, gotchas, and things that will bite you

### 7.1 The UAT workflow is currently out of sync with the prod workflow

Per PR `rule-studio-example#13`: `deploy.yml` was modernised to use BuildKit `--mount=type=secret,id=GH_TOKEN`, but `deploy-to-uat.yml` still uses the old `--build-arg GH_TOKEN=…` approach *and* patches the Dockerfile with a `sed '/^RUN npm ci/i…'` command that no longer matches anything on the `dev` branch of `rule-executer`. Result: UAT builds will silently 401 on `npm ci`.

Workaround until PR #13 is fully patched: **do not merge changes to your rule repos' `prod` branch** (which triggers `deploy-to-uat.yml`) until either (a) the UAT workflow is aligned with the prod workflow, or (b) you fork the template and apply the same BuildKit-secret changes to `deploy-to-uat.yml` yourself.

### 7.2 `RULE_VERSION` is hardcoded to `1.0.0` in `deploy.yml`

`deploy.yml` line 167 sets `-e RULE_VERSION=1.0.0` regardless of what version was actually just published by `publish.yml`. The workflow *does* extract the real version (`RULE_VERSION=$(node -p "require('./node_modules/rule/package.json').version")`) into `$RULE_VERSION` in a prior step, but then the `docker run` block re-hardcodes it. This means the rule-executer's NATS subject will always be `sub-rule-<name>@1.0.0` no matter what version you publish. Fix: change `-e RULE_VERSION=1.0.0` to `-e RULE_VERSION=$RULE_VERSION` in the `docker run` block.

### 7.3 Silent default change of `SERVER_IP` and `RUNNER_LABEL` in PR #13

If you have rule repos that were happily deploying under the old workflow to `server-18` at `10.10.80.18` without having set `RUNNER_LABEL`/`SERVER_IP` explicitly, they will silently start deploying to `server-35` at `10.10.80.35` after PR #13. **Set these variables explicitly at the org level** to whatever hosts you actually want, so future template updates cannot silently retarget you.

### 7.4 `GITHUB_DEFAULT_BRANCH` → `GITHUB_BRANCH` rename in PR #5

`rule-studio-devtestops` startup config validation is strict (`optional: false`). When PR #5 merges, existing deployments with `GITHUB_DEFAULT_BRANCH=main` in `.env` will crash on startup because they now need `GITHUB_BRANCH=main`. Roll out the `.env` change in lockstep with the code deploy.

### 7.5 Encrypted `GITHUB_TOKEN` key/IV mismatch

If DevTestOps starts up and immediately errors with a decrypt failure, 99% of the time your `ENCRYPTION_KEY` is not 32 bytes or your `ENCRYPTION_IV` is not 16 bytes. Print `Buffer.byteLength(process.env.ENCRYPTION_KEY, 'utf8')` in a one-liner to check. See §3.3.

### 7.6 Git credentials in `.git/config` after PR #5

The new `simple-git` bootstrap flow in PR #5 clones with `https://x-access-token:<TOKEN>@github.com/…` — which persists the token to `<tempDir>/.git/config` for the duration of the operation. If the process is killed mid-flight (OOM, container restart), the token can survive on disk in the OS tempdir. The `finally` `fs.rm(tempDir)` covers the happy path. See the PR-5 review in `pull-requests/RSDTO-PR-5.md` for suggested mitigations (use `-c http.extraheader=…` on the command line so the credential is not persisted, and scrub credentials from `handleError` log output).

### 7.7 rule-executer branch: `feat-paysys` vs `dev`

`deploy.yml` and `deploy-to-uat.yml` both clone `rule-executer -b dev` (was `-b feat-paysys`). If your fork of `rule-executer` uses a different branch (e.g. `main`), update the workflow's `git clone` lines. Also verify that your `dev` branch of `rule-executer` uses the BuildKit-secret Dockerfile shape (`RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci --ignore-scripts`) — if you're on an older Dockerfile the `sed` patch in `deploy.yml` will fail to match.

### 7.8 npm publish scope collisions

`publish.yml` publishes as `@<github.repository_owner>/<repo-name>`. If your org name has changed (e.g. `psl-copilot` → `paysys`), the same rule will publish under a different scope and existing installers will need updating. Consider standardising on one org name up front.

### 7.9 Database password `unused` in `deploy.yml`

The workflow literally sets `-e RAW_HISTORY_DATABASE_PASSWORD=unused`. This implies your PostgreSQL is configured with `trust` or `peer` auth on the runner's IP. **Do not carry this to a production environment.** Change the workflow to read the password from a GitHub Actions secret (`${{ secrets.POSTGRES_PASSWORD }}`) and set it up per-environment.

### 7.10 There is no rollback in `deploy.yml`

The workflow does `docker stop $RULE_NAME || true; docker rm $RULE_NAME || true; docker run -d --name $RULE_NAME …`. If the new container fails to start (crash loop, missing env), the old one is already gone. Consider adding a `docker inspect` check after `docker run` and rolling back to the previous image if the new container isn't healthy within N seconds.

---

## 8. Verifying the whole flow works — a smoke-test checklist

Once everything above is set up, run this end-to-end smoke test:

1. **Bootstrap:** in Rule Studio UI, create a new rule. Confirm a new repo appears under `<GITHUB_ORG_NAME>/<ruleId>` with the template's contents on the `staging` branch.
2. **Populate:** edit the rule code in the UI, save. Confirm `src/rule.ts` and `__tests__/unit/rule.test.ts` on the `staging` branch update.
3. **Unit tests:** confirm the `Unit Tests` workflow runs on push and turns green.
4. **Promote:** promote the rule from `staging` → `dev` in the UI. Confirm the `dev` branch on GitHub is now at the same commit as `staging`.
5. **Publish:** confirm the `Build and Publish Package` workflow runs on push to `dev` and publishes `@<org>/<ruleId>@<version>` to `npm.pkg.github.com` (visible at your org's Packages page).
6. **Deploy:** confirm the `Rule Executer - Rule processor Automation` workflow runs (either after publish, or via manual dispatch), lands on your self-hosted runner, and finishes successfully. On the runner: `docker ps | grep <ruleId>` should show a running container.
7. **Simulate:** in the Rule Editor, run a read-only simulation with a valid transaction payload. Watch the container's logs (`docker logs -f <ruleId>`) — you should see the payload received and processed, and the UI should display the result.

If step 7 hangs or times out, jump to §6.4 trap and check the NATS subject shape.

---

## 9. Where to look when something breaks

| Symptom | First place to look |
|---------|---------------------|
| DevTestOps crashes on startup | `ENCRYPTION_KEY` / `ENCRYPTION_IV` byte length; encrypted `GITHUB_TOKEN` format. §3.3. |
| Bootstrap fails with `spawn git ENOENT` | `git` not installed in the DevTestOps runtime image. §4.2. |
| Bootstrap succeeds but the new repo is empty | Template repo's default branch does not match `GITHUB_BRANCH` env var. §4.1. |
| `npm ci` in `deploy.yml` fails with 401 | `TAZAMA_TOKEN` scopes; BuildKit `--secret` mount syntax mismatch between the workflow and the rule-executer Dockerfile branch. §5.5, §7.1, §7.7. |
| Container runs but the simulation times out | NATS subject mismatch — the container subscribes to `sub-rule-<name>@<version>` but the UI publishes to `sub-<rule_config_id>`. §6.4, §7.2. |
| Container starts then immediately exits | Missing/wrong `SERVER_URL`, `RAW_HISTORY_DATABASE_HOST`, etc. `docker logs <container>` from the runner. §5.6. |
| Wrong runner picks up the workflow | `RUNNER_LABEL` variable unset → defaults to `server-35`. §5.4, §7.3. |
| Docker image not on Docker Hub | It isn't supposed to be — the current pipeline only builds locally. §6.5. |
| Unit-test HTML report missing | `GITHUB_TEST_REPORT_PATH` env var in DevTestOps; check the `unit-test.yml` workflow ran successfully. §4.1. |

---

## 10. Change log for this document

- **2026-07-17** — Initial version. Written against the state of the three open PRs (`rule-studio-example#13` on branch `feat-paysys-poc`, `rule-studio#132` on branch `feat-paysys-fix-simulation`, `rule-studio-devtestops#5` on branch `fix/bootstrap`). Update this document once those PRs merge and their traps (§7.1–§7.6) are resolved.
