# FSDT deployment guide

A step-by-step guide to deploying the full Tazama stack using Full-Stack-Docker-Tazama (FSDT). Written for people who are comfortable running commands from a terminal but may not be Tazama or Docker experts.

There are two deployment shapes:

1. **Single-machine deployment** — everything runs on one computer. Best for demos, evaluation, local development, and small proof-of-concept work. Section 1.
2. **Three-machine deployment** — the stack is split across three servers (typically AWS EC2 instances). Better resource isolation, closer to a production topology. Section 2.

Read section 1 first even if you plan to do a three-machine deployment. The three-machine section builds on the single-machine one.

---

## Contents

- [What is FSDT and what is it not](#what-is-fsdt-and-what-is-it-not)
- [The three stacks (core, extensions, BIAR)](#the-three-stacks-core-extensions-biar)
- [Section 1: single-machine deployment](#section-1-single-machine-deployment)
  - [1.1 Prerequisites](#11-prerequisites)
  - [1.2 Get a GitHub token](#12-get-a-github-token)
  - [1.3 Clone the FSDT repository](#13-clone-the-fsdt-repository)
  - [1.4 Deploy the core stack](#14-deploy-the-core-stack)
  - [1.5 Verify core is working](#15-verify-core-is-working)
  - [1.6 Deploy the extensions stack (optional)](#16-deploy-the-extensions-stack-optional)
  - [1.7 Deploy the BIAR stack (optional)](#17-deploy-the-biar-stack-optional)
  - [1.8 Everyday operations on a single machine](#18-everyday-operations-on-a-single-machine)
- [Section 2: three-machine deployment](#section-2-three-machine-deployment)
  - [2.1 What changes on three machines](#21-what-changes-on-three-machines)
  - [2.2 Prepare each server](#22-prepare-each-server)
  - [2.3 Configure network reachability](#23-configure-network-reachability)
  - [2.4 Deploy Server A (core)](#24-deploy-server-a-core)
  - [2.5 Deploy Server B (extensions)](#25-deploy-server-b-extensions)
  - [2.6 Deploy Server C (BIAR)](#26-deploy-server-c-biar)
  - [2.7 Verify end-to-end](#27-verify-end-to-end)
- [Section 3: common problems and fixes](#section-3-common-problems-and-fixes)
- [Appendix A: port reference](#appendix-a-port-reference)
- [Appendix B: environment variable reference](#appendix-b-environment-variable-reference)

---

## What is FSDT and what is it not

**FSDT is:** a collection of Docker Compose files and helper scripts that spin up the Tazama transaction-monitoring platform for demos, evaluation, and sandbox work. It ships pre-built Docker images from DockerHub and lets you choose either those images or source builds from GitHub.

**FSDT is not:** a production deployment tool. Passwords are hardcoded, TLS is not enforced, and the security model assumes a trusted network. For production, follow the Kubernetes/Helm guides linked from the repo's main README (`On-Prem-helm`, `EKS-helm`, `GKE-helm`, `AKS-helm`).

---

## The three stacks (core, extensions, BIAR)

FSDT is divided into three independent stacks. Each lives in its own directory in the repo and runs as its own Docker Compose "project."

| Stack | Folder | What it does | Runs on |
|---|---|---|---|
| **Core** | `core/` | The transaction monitoring pipeline: TMS (the API you send transactions to), Event Director, Typology Processor, Event Adjudicator, rule processors, admin service, PostgreSQL, NATS, Valkey (Redis-compatible cache), Keycloak (optional auth) | Server A |
| **Extensions** | `extensions/` | Configuration and case management: Transaction Configuration Studio (TCS), Transaction Rule Studio (TRS), Case Management System (CMS), OpenSearch, CouchDB, Flowable, SFTP | Server B (plus two APIs that also run on Server A) |
| **BIAR** | `biar/` | Analytics: NiFi (data ingestion), Ozone (S3-compatible storage), Solr (search), Tika (document parsing), JupyterHub | Server C |

Each stack is optional. You can run core alone, or core + extensions, or all three. BIAR needs core running; extensions needs core running.

On a single machine, all three stacks live on the same computer but stay in separate Docker networks and talk to each other over host ports. On three machines, each server runs one stack and they talk over the network between them.

---

# Section 1: single-machine deployment

Everything below runs on one computer. This is the fastest way to get a working Tazama environment.

## 1.1 Prerequisites

You need to install these on your computer **before** you start:

1. **Git** — to clone the FSDT repository.
   - Windows: [git-scm.com/download/win](https://git-scm.com/download/win)
   - macOS: `xcode-select --install` (comes with Xcode command-line tools) or Homebrew: `brew install git`
   - Linux: `sudo dnf install git` (Fedora/RHEL) or `sudo apt install git` (Debian/Ubuntu)

2. **Docker** with **Docker Compose v2**.
   - Windows and macOS: install **Docker Desktop** from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/). Compose v2 is included.
   - Linux: install the **Docker Engine** package for your distribution and the `docker-compose-plugin` add-on. Verify with:
     ```bash
     docker compose version
     ```
     You should see something like `Docker Compose version v2.x` or `v5.x`. If you see a v1 number or `command not found`, upgrade before continuing.

3. **Machine resources**: at least **8 GB RAM** free for the core stack alone, **16 GB** if you plan to run extensions + BIAR alongside. At least **20 GB** free disk. A modern CPU with 4+ cores.

4. **Ports free on your machine**: FSDT publishes many host ports (see [Appendix A](#appendix-a-port-reference)). The important ones for core alone are: `5000`, `5100`, `14222`, `15432`, `16379`. Make sure nothing else on your machine is already listening on those.

## 1.2 Get a GitHub token

Some of the Docker images (and all of the source builds) come from `ghcr.io`, GitHub's Container Registry. That requires authentication even for public images.

1. Log into GitHub. Go to **Settings → Developer settings → Personal access tokens → Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Give it a name (`FSDT local`), an expiration, and check the `read:packages` scope.
4. Click **Generate token**. Copy the token immediately — GitHub only shows it once. It looks like `ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`.

Now put the token where Docker can find it:

**Windows PowerShell:**
```powershell
$env:GH_TOKEN = 'ghp_XXXXXXXXXXXX'
echo $env:GH_TOKEN | docker login ghcr.io -u <your-github-username> --password-stdin
```

**macOS or Linux (bash/zsh):**
```bash
export GH_TOKEN='ghp_XXXXXXXXXXXX'
echo "$GH_TOKEN" | docker login ghcr.io -u <your-github-username> --password-stdin
```

Replace `<your-github-username>` with your actual GitHub username. You should see `Login Succeeded`.

The `docker login` result is stored in `~/.docker/config.json` and reused for all future pulls, so you only need to do this once (or when the token expires and you generate a new one).

## 1.3 Clone the FSDT repository

Pick a working directory (any folder you have write access to) and run:

```bash
git clone https://github.com/tazama-lf/Full-Stack-Docker-Tazama.git
cd Full-Stack-Docker-Tazama
```

The repository defaults to the `main` branch, which is the last released version. If you want the latest development work instead, add `-b dev` to the clone command. `main` is the safer choice for a first deployment.

## 1.4 Deploy the core stack

Start Docker Desktop if it isn't already running. Then from the `Full-Stack-Docker-Tazama` folder:

```bash
cd core
./tazama-core.sh          # macOS / Linux
```

On Windows:

```
cd core
tazama-core.bat
```

You'll see a menu:

```
Select docker deployment type:

1. Public (GitHub)
2. Public (DockerHub)
3. Full-service (DockerHub)
4. Multi-Tenant Public (DockerHub)
5. Docker Utilities
6. Database Utilities
7. Consoles
```

**For your first deployment, pick option 2 (Public DockerHub).** This pulls pre-built images and is by far the fastest.

You'll then see an addons menu with checkboxes for extra features. The defaults (`pgAdmin` and `Hasura` on, everything else off) are fine. Type `a` to apply, then `e` to execute.

The launcher runs a long `docker compose ... -p tazama-core up -d` command and streams progress as images pull. First run takes 3–10 minutes depending on your network. Subsequent runs are seconds because images are cached.

**Important:** the launcher uses the compose project name `tazama-core`. Every Docker artifact it creates has this prefix — containers named `tazama-core-postgres-1`, `tazama-core-nats-1`, and so on, and a network called `tazama-core_default`. Remember this name; you'll use it for troubleshooting.

## 1.5 Verify core is working

Once the launcher finishes, run:

```bash
docker compose -p tazama-core ps
```

You should see about 19 containers, most in state `Up X seconds (healthy)`. If any are in `Restarting` or `Exited`, jump to [section 3](#section-3-common-problems-and-fixes).

Then check the two public HTTP endpoints:

```bash
curl http://localhost:5000
# expected: {"status":"UP"}

curl http://localhost:5100/health
# expected: {"status":"UP"}
```

`5000` is the TMS (Transaction Monitoring Service) — this is the API you'll send transactions to. `5100` is the admin service.

Open a web browser to `http://localhost:5100/documentation` and you'll see the admin service's Swagger UI. Same for `http://localhost:5000/documentation`. That's the whole core stack running.

## 1.6 Deploy the extensions stack (optional)

If you want the configuration studios and case management UI:

```bash
cd ../extensions
./tazama-extensions.sh
```

You'll get a menu with two phases:

- Options **1 or 2** — deploy DEMS and DEAPI **on top of the running core** (they join the core's Docker network). Pick **option 2** for DockerHub images.
- Options **3 or 4** — deploy the extensions services themselves (TCS, TRS, CMS, OpenSearch, etc.) in a *separate* Docker Compose project called `tazama-extensions`. Pick **option 4** for DockerHub images.

Run options 2 and 4 in order.

Web UIs you can then open:

- TCS: `http://localhost:5173`
- TRS: `http://localhost:5174`
- CMS: `http://localhost:5175`

## 1.7 Deploy the BIAR stack (optional)

Only useful if you want the analytics side (NiFi, Solr, Ozone, JupyterHub). BIAR is resource-heavy — probably don't run it on the same machine as core + extensions unless you have 32 GB of RAM.

```bash
cd ../biar
./tazama-biar.sh          # macOS/Linux
# or on Windows:
tazama-biar.bat
```

Pick **option 1 (DockerHub images)**. The launcher checks that it can reach the core stack's NATS port (`14222`) before starting — if that check fails, it refuses to start.

Wait 2–3 minutes; Ozone and NiFi have long startup routines.

Web UIs:

- NiFi: `http://localhost:8088`
- JupyterHub: `http://localhost:8000`

## 1.8 Everyday operations on a single machine

**See what's running:**
```bash
docker compose -p tazama-core ps
docker compose -p tazama-extensions ps    # if deployed
docker compose -p tazama-biar ps          # if deployed
```

**Tail logs for a service:**
```bash
docker compose -p tazama-core logs -f admin-service
```

Ctrl-C to stop tailing.

**Restart a single service:**
```bash
docker compose -p tazama-core restart admin-service
```

**Stop everything for the core stack:**
```bash
docker compose -p tazama-core down
```

**Wipe everything, including data volumes** (destroys the database):
```bash
docker compose -p tazama-core down --volumes --remove-orphans
```

**Reclaim disk space** (removes old images, unused build cache):
```bash
docker system prune -a --volumes -f
```

You can also re-run `tazama-core.sh` and pick option 5 ("Docker Utilities") for a menu-driven version of these commands.

---

# Section 2: three-machine deployment

You want core, extensions, and BIAR on three separate boxes. Reasons: resource isolation, closer-to-production topology, or you're deploying to AWS.

## 2.1 What changes on three machines

**Same as single-machine:**
- Prerequisites (git, docker, GH_TOKEN, `docker login ghcr.io`).
- The same launcher scripts on each machine (`tazama-core.sh` on A, `tazama-extensions.sh` on B, `tazama-biar.sh` on C).
- The same compose project names (`-p tazama-core`, `-p tazama-extensions`, `-p tazama-biar`).
- The same host ports (they're published on each server; see [Appendix A](#appendix-a-port-reference)).

**Different from single-machine:**
- Services on one server reach services on another server over the **network** (IP addresses or DNS names), not `localhost`.
- The launcher scripts read `SERVER_A_HOST`, `SERVER_B_HOST`, and `SERVER_C_HOST` from each stack's `.env` file. On a single machine those default to `localhost` (or `host.docker.internal` in a couple of places), which works because everything is on the same box. On three machines you must change them so each server knows how to reach the other two.
- Each server's firewall (or AWS security group) must allow inbound traffic on the specific ports the other servers will call. See [section 2.3](#23-configure-network-reachability).
- The extensions launcher can no longer copy the auth public key from `../core/auth/` automatically — the folder is on a different machine. You must copy it manually (or use the AWS deploy scripts, which do it for you).

## 2.2 Prepare each server

Do this on **all three** servers (A, B, C):

1. Install git and Docker (per [1.1](#11-prerequisites)).
2. Set up the GitHub token and `docker login ghcr.io` (per [1.2](#12-get-a-github-token)).
3. Clone the repo (per [1.3](#13-clone-the-fsdt-repository)).

Machine sizing:

| Server | Minimum RAM | Minimum CPU | Minimum disk |
|---|---|---|---|
| Server A (core) | 8 GB | 4 cores | 30 GB |
| Server B (extensions) | 8 GB | 4 cores | 30 GB |
| Server C (BIAR) | 16 GB | 4 cores | 50 GB |

BIAR is the memory hog — Ozone plus NiFi plus JVM overhead adds up quickly.

## 2.3 Configure network reachability

Pick an addressing scheme for the three servers. There are three common ones:

**Option A — private IP addresses on a shared LAN or VPN.** Example: A=`10.0.1.10`, B=`10.0.1.20`, C=`10.0.1.30`. Simplest.

**Option B — private DNS names in a shared zone.** Example: `core.tazama.internal`, `extensions.tazama.internal`, `biar.tazama.internal`. Cleaner in the long run because you can move a server without editing configs.

**Option C — AWS VPC with Route 53 private hosted zone.** This is what the `infra/aws/` OpenTofu configuration sets up automatically. Read `infra/aws/aws-deployment-instructions.md` for that flow — it's out of scope for this manual guide.

Whichever scheme you pick, on **each server**, edit that server's `.env` files as follows.

**Server A — `core/.env`:** no cross-server change needed. Core is the origin; it doesn't call B or C by name.

**Server B — `extensions/.env`:** find the line `SERVER_A_HOST=host.docker.internal` (or `localhost`) and change it to Server A's address:
```
SERVER_A_HOST=10.0.1.10                              # option A
SERVER_A_HOST=core.tazama.internal                   # option B
```

**Server C — `biar/.env`:** find the lines `SERVER_A_HOST=` and `SERVER_B_HOST=` and set them to Server A's and Server B's addresses:
```
SERVER_A_HOST=10.0.1.10
SERVER_B_HOST=10.0.1.20
```

### Firewall / security group rules

Each server needs to accept inbound TCP on the specific ports the other servers will call. Minimum required inbound rules:

**Server A must accept from Server B on:**
- `14222` (NATS — extensions send messages to core)
- `15432` (PostgreSQL — TCS/CMS backends read core config data)
- `16379` (Valkey — CMS reads cache)
- `3020` (auth service — TCS/TRS/CMS validate tokens)
- `5100` (admin service — TRS backend calls admin API)
- `3001` and `3002` (DEAPI, DEMS — TCS frontend calls them; note these are deployed *on* Server A even though they're extensions services)

**Server A must accept from Server C on:**
- `14222` (NATS — BIAR reads message flow)
- `15432` (PostgreSQL — NiFi reads raw_history / event_history)

**Server B must accept from Server A on:**
- `3001` and `3002` (return path for DEAPI/DEMS calls originating from Server A to Server B, if applicable to your deployment)

**Server B must accept from Server C on:**
- `15433` (PostgreSQL — NiFi reads CMS database)
- `9200` (OpenSearch — NiFi reads audit logs)
- `3090` (CMS backend — Server C's data lakehouse API may call it)

**Server C must accept from Server B on:**
- `8282` (data lakehouse API — CMS backend queries it directly)

Publicly-facing ports (the ones you access from your workstation) usually only need to be open on the server that hosts them:

- Server A: `5000`, `5100`, `8080` (Keycloak), `3020` (auth) if you'll log in from outside
- Server B: `5173`, `5174`, `5175` (the three studio UIs), plus their backend ports if you'll call them directly
- Server C: `8088` (NiFi), `8000` (JupyterHub)

## 2.4 Deploy Server A (core)

On Server A only:

```bash
cd Full-Stack-Docker-Tazama/core
./tazama-core.sh
```

Same menu as single-machine ([1.4](#14-deploy-the-core-stack)). Pick option 2 (DockerHub) or option 1 (GitHub source) depending on whether you want to build from source.

**Do not proceed to Server B or C until Server A is fully up and `curl http://localhost:5000` returns `{"status":"UP"}` on Server A itself.**

Also test from Server B and Server C that they can reach Server A's ports:

```bash
# on Server B
nc -zv 10.0.1.10 14222
nc -zv 10.0.1.10 15432
nc -zv 10.0.1.10 5100

# on Server C
nc -zv 10.0.1.10 14222
nc -zv 10.0.1.10 15432
```

Each should print "succeeded" or "open". If not, fix the firewall/security group before continuing.

## 2.5 Deploy Server B (extensions)

**Before running the extensions launcher on Server B**, copy the auth public key from Server A. The extensions services need it to validate JWT tokens issued by Server A's auth service.

On Server B:
```bash
mkdir -p Full-Stack-Docker-Tazama/extensions/auth
scp user@10.0.1.10:/path/to/Full-Stack-Docker-Tazama/core/auth/test-public-key.pem \
    Full-Stack-Docker-Tazama/extensions/auth/test-public-key.pem
```

(Adjust the path to match where you cloned the repo on Server A. `user` is whatever SSH user you have on Server A.)

Then run the launcher:

```bash
cd Full-Stack-Docker-Tazama/extensions
./tazama-extensions.sh
```

The menu is the same as [1.6](#16-deploy-the-extensions-stack-optional), but pick **only options 3 or 4**. Options 1 and 2 (deploying DEMS/DEAPI to Server A) must be run from Server A, not Server B, because they add containers to Server A's `tazama-core` project.

To deploy DEMS/DEAPI on Server A, go back to Server A and run:

```bash
cd Full-Stack-Docker-Tazama/extensions
./tazama-extensions.sh
```

Then pick option 2 (DEMS+DEAPI, DockerHub).

Verify from Server B:
```bash
curl http://localhost:3010/health   # TCS backend
curl http://localhost:3005/health   # TRS backend
curl http://localhost:3090/health   # CMS backend
```

## 2.6 Deploy Server C (BIAR)

On Server C:

```bash
cd Full-Stack-Docker-Tazama/biar
./tazama-biar.sh
```

Pick option 1 (DockerHub). The launcher runs a TCP reachability check against `${SERVER_A_HOST}:14222` before proceeding. If it fails, the launcher aborts with an error message — go back to [2.3](#23-configure-network-reachability) and fix the network path.

Wait 2–3 minutes for Ozone and NiFi to initialise. Then:

```bash
curl http://localhost:8088/nifi/     # returns HTML (the NiFi UI)
```

## 2.7 Verify end-to-end

From your workstation, test that services on each server are reachable:

```bash
curl http://<server-a-address>:5000            # TMS (should be {"status":"UP"})
curl http://<server-b-address>:3010/health     # TCS backend
curl http://<server-c-address>:8088/nifi-api/system-diagnostics  # NiFi API
```

If all three return sensible responses, your three-machine deployment is up.

For a real end-to-end sanity check, send a test transaction to TMS on Server A and confirm it flows through the pipeline. The FSDT repo's [`postman/`](https://github.com/tazama-lf/postman) collection has ready-made Newman-runnable tests for this.

---

# Section 3: common problems and fixes

## "Docker Compose version v1.x" or "docker-compose: command not found"

You have Docker Compose v1 or nothing. FSDT requires **Compose v2 or later**, invoked as `docker compose` (with a space), not `docker-compose` (with a hyphen).

- On Linux: `sudo dnf install docker-compose-plugin` (Fedora) or the equivalent for your distro. On some distros you'll have to grab the binary from GitHub releases and put it in `~/.docker/cli-plugins/docker-compose`.
- On Windows or macOS: install Docker Desktop, which bundles v2.

## `docker: Error response from daemon: network tazama_default not found`

You're seeing this because something is trying to connect to a Docker network called `tazama_default`, but the actual network is called `tazama-core_default`. The network name is derived from the compose project name (the `-p` flag). The current launcher uses `-p tazama-core`, so the network is `tazama-core_default`.

If it's a launcher script you've written yourself, edit it to say `--network tazama-core_default`. If it's the FSDT launcher itself failing, something is very wrong — file an issue in the FSDT repo.

## Admin service crash-loops with `SimulationDB: 'err, error: database "simulation" does not exist'`

Your postgres initdb didn't create the `simulation` database. Two possible causes:

- You're on an old `dev` commit that predated the simulation-DB provisioning. Update to the latest `dev` (`git pull origin dev`) and re-run the launcher.
- The postgres volume was left over from an earlier deploy and initdb never re-ran. Fix: `docker compose -p tazama-core down --volumes`, then re-run the launcher.

## `Port already in use` when running the launcher

Something else on your machine is already bound to a port FSDT wants. Find and kill it:

```bash
# Linux/macOS
sudo ss -tlnp | grep :5000
# or
sudo lsof -i :5000
```

Kill the offending process, or change the port in the relevant `.env` file (e.g. `core/.env` has `TMS_PORT=5000` — change it to `5001`).

## OpenSearch (Server B) exits with `max_map_count [65530]` error

Linux kernel setting. Fix on Server B:

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
```

## Extensions fail to start with `ERROR: tazama-core is not running`

The extensions launcher checks for a running `tazama-core` compose project on the local machine before deploying DEMS/DEAPI. If you're on Server B (three-machine deployment), you cannot run options 1 or 2 there — the check requires core to be on the same box. Only options 3 and 4 are valid on Server B.

## BIAR launcher aborts with `tazama-core is not reachable at ...:14222`

The BIAR launcher does a TCP check against `${SERVER_A_HOST}:14222`. Fails when:

- Server A's core stack isn't running.
- `biar/.env` still has `SERVER_A_HOST=localhost` on a three-machine deployment. Fix it to Server A's real address.
- Server C's outbound firewall or Server A's inbound firewall blocks port 14222. Open the port.
- Server A's NATS container isn't published on `14222`. Verify with `docker compose -p tazama-core ps nats`.

## TCS/TRS/CMS show `Unauthorized` or `JWT validation error`

They can't validate the JWT token issued by Server A's auth service. On a single machine the extensions launcher copies the public key automatically. On three machines you have to copy it yourself; see [2.5](#25-deploy-server-b-extensions). If the file exists but validation still fails, check that the file on Server B is byte-identical to Server A's `core/auth/test-public-key.pem`.

## Hasura won't add databases (log shows `Failed: Adding <name> database`)

Restart the init container:

```bash
docker restart tazama-core-hasura-init-1
```

If that doesn't fix it, disable Hasura in the launcher menu and re-run. Hasura is optional; you don't need it for basic Tazama functioning.

---

## Appendix A: port reference

Published host ports per stack. On a single-machine deployment, all of these are on `localhost`. On a three-machine deployment, each set is on its respective server.

### Core stack (Server A)

| Port | Service | Purpose |
|---|---|---|
| 5000 | TMS | Transaction submission API |
| 5100 | Admin service | Rule/typology/condition admin API |
| 3020 | Auth service | JWT issuance and validation |
| 8080 | Keycloak | Identity provider (if auth addon enabled) |
| 14222 | NATS | Messaging bus (client port) |
| 16222 | NATS | Cluster port (internal) |
| 18222 | NATS | Monitoring HTTP API |
| 15432 | PostgreSQL | Primary database |
| 16379 | Valkey | Redis-compatible cache |
| 3001 | DEAPI | Data enrichment API (deployed on Server A via extensions launcher option 2) |
| 3002 | DEMS | Data enrichment monitoring service (same as above) |
| 5050 | pgAdmin | Optional PostgreSQL web UI |
| 6100 | Hasura | Optional GraphQL layer over PostgreSQL |
| 3011 | Tazama Demo UI | Optional demo web UI |
| 4000 | NATS utilities | Optional NATS REST proxy |

### Extensions stack (Server B)

| Port | Service | Purpose |
|---|---|---|
| 3010 | TCS backend | Configuration studio API |
| 5173 | TCS frontend | Configuration studio web UI |
| 3005 | TRS backend | Rule studio API |
| 5174 | TRS frontend | Rule studio web UI |
| 3090 | CMS backend | Case management API |
| 5175 | CMS frontend | Case management web UI |
| 18866 | Voila | Notebook server (embedded in CMS UI) |
| 15433 | PostgreSQL (extensions) | CMS + config data. **Note the different port** — this postgres is separate from Server A's postgres |
| 9200 | OpenSearch | Audit log store |
| 5984 | CouchDB | Case evidence document store |
| 8081 | Flowable | Workflow engine |
| 12222 | SFTP | File exchange for bulk imports |
| 5051 | pgAdmin | Optional |

### BIAR stack (Server C)

| Port | Service | Purpose |
|---|---|---|
| 8088 | NiFi | Data flow designer UI |
| 8000 | JupyterHub | Analytics notebook environment |
| 7619 | Automation Orchestrator | PySpark orchestration API |
| 8282 | Datalakehouse API | Data lakehouse query API (called by CMS backend) |
| 8983 | Solr | Full-text search |
| 9878 | Ozone S3G | S3-compatible object storage gateway |
| 9874 | Ozone OM | Ozone Object Manager |
| 9876 | Ozone SCM | Ozone Storage Container Manager |
| 9888 | Ozone Recon | Ozone monitoring UI |
| 9998 | Tika | Document parsing |

---

## Appendix B: environment variable reference

The variables that matter most for a first deployment. Everything is set in the `.env` file of the stack it belongs to (`core/.env`, `extensions/.env`, `biar/.env`).

| Variable | Where set | Default | Purpose |
|---|---|---|---|
| `TAZAMA_VERSION` | `core/.env` | `rc` | DockerHub tag. `rc` = latest dev builds, `latest` = last release, or a specific version like `3.0.0` |
| `ADMIN_BRANCH`, `TMS_BRANCH`, etc. | `core/.env` | `dev` | Which git branch to build from (only used if you pick option 1 GitHub in the launcher) |
| `SERVER_A_HOST` | `extensions/.env`, `biar/.env` | `host.docker.internal` (ext) / `localhost` (biar) | Where to reach Server A. Change this on three-machine deployments |
| `SERVER_B_HOST` | `biar/.env` | `localhost` | Where to reach Server B. Change this on three-machine deployments |
| `SERVER_C_HOST` | rarely used directly | `localhost` | Where to reach Server C (rarely needed since C doesn't call itself) |
| `GH_TOKEN` | shell environment (not in `.env`) | (empty) | GitHub PAT for `docker login ghcr.io`. See [1.2](#12-get-a-github-token) |
| `TMS_PORT`, `ADMIN_PORT`, `PGADMIN_PORT`, etc. | `core/.env` | see [Appendix A](#appendix-a-port-reference) | Change if you have port collisions on your machine |
| `POSTGRES_HOST_AUTH_METHOD` | set inside `docker-compose.base.auth.yaml` when auth is enabled | `trust` | Postgres auth mode. `trust` = no password required. Fine for FSDT; not for production |

### For AWS deployments only

If you go the OpenTofu / EKS route, additional variables come into play (SSM parameter names, ALB DNS names, VPC IDs). Those live in `infra/aws/*.tfvars` and are documented separately in `infra/aws/aws-deployment-instructions.md`.

---

## Where to go from here

- **Full API surface**: `postman/` in the [tazama-lf/postman](https://github.com/tazama-lf/postman) repository has ready-made request collections for TMS, admin service, TCS, TRS, and CMS.
- **AWS OpenTofu deployment**: `infra/aws/aws-deployment-instructions.md` in the FSDT repo. Six phases (A-F), fully scripted.
- **Production deployment**: use one of the Helm charts referenced in `README.md` — `On-Prem-helm`, `EKS-helm`, `GKE-helm`, or `AKS-helm`. FSDT is not appropriate for production.
- **Troubleshooting deeper issues**: `core/README.md`, `extensions/README.md`, and `biar/README.md` have per-stack troubleshooting sections that go beyond what's covered here.
