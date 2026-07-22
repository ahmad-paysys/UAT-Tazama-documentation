# BullMQ in rule-studio/backend — Status Report

<a id="top"></a>

**Repository:** [tazama-lf/rule-studio](https://github.com/tazama-lf/rule-studio) — branch `dev`
**HEAD at time of investigation:** `ef48726` (Merge PR #132 from `feat-paysys-fix-simulation`)
**Investigated:** 2026-07-20
**Author:** ahmad.khalid@paysyslabs.com

---

## TL;DR

**BullMQ is present, current, and load-bearing in rule-studio/backend on `dev` — but it lives in the wrong place.** It belongs entirely to the OLD simulation flow (`SendToDemsService`, `SimulationProcessor`, `QueuesModule`) plus two peer modules that consume `SendToDemsService` (`fetch-from-dlh`, `rerun-simulation`). Sim-studio itself ([src/services/simulation-studio/](../repos/rule-studio/backend/src/services/simulation-studio/), 44 files) is BullMQ-free.

**Intent vs reality gap:** the design intent was that `send-to-dems` (and by extension the simulation flow) should be independent of BullMQ — i.e. synchronous or driven by a different mechanism. **That refactor has NOT landed on `dev`.** `SendToDemsService` is, today, a pure BullMQ enqueuer with no synchronous DEMS call path. This is flagged for future work — see §7.

Any deployment that starts `trs-backend` without a working Redis/valkey endpoint will fail to initialise `BullModule` at boot, regardless of whether the client ever hits the `/send-to-dems/simulate` route — so the FSDT Redis env fix (commit `b776b07`) remains necessary until the refactor lands.

---

## Table of Contents

- [1. Dependencies](#1-dependencies)
- [2. Source-level usage](#2-source-level-usage)
- [3. Runtime contract with Redis / valkey](#3-runtime-contract-with-redis--valkey)
- [4. Recency — how current is this code](#4-recency--how-current-is-this-code)
- [5. Implications for FSDT deployment](#5-implications-for-fsdt-deployment)
- [6. Sim-studio boundary — confirmed clean](#6-sim-studio-boundary--confirmed-clean)
- [7. Intent gap — `send-to-dems` should not use BullMQ (future work)](#7-intent-gap--send-to-dems-should-not-use-bullmq-future-work)
- [8. Recommendations](#8-recommendations)

[Back to top](#top)

---

## 1. Dependencies

From [rule-studio/backend/package.json](../repos/rule-studio/backend/package.json):

| Line | Package | Pinned version | Resolved (package-lock) |
|------|---------|----------------|--------------------------|
| 33 | `@nestjs/bullmq` | `^11.0.4` | `11.0.4` |
| 50 | `bullmq` | `^5.74.1` | `5.76.8` |

Both are production dependencies (not devDependencies). They are also present in `backend/package-lock.json` at lines 14, 31, 3310, 6309.

No sibling packages named `bull`, `@nestjs/bull` (the older, deprecated NestJS Bull adapter), or `bee-queue` — this is BullMQ specifically, not the legacy Bull v3/v4 API.

[Back to top](#top)

---

## 2. Source-level usage

Four production files and one test file reference BullMQ. Each is doing real, load-bearing work.

### 2.1 [backend/src/queues/queues.module.ts](../repos/rule-studio/backend/src/queues/queues.module.ts)

Wires the BullMQ NestJS module into the app graph:

```ts
import { BullModule } from '@nestjs/bullmq';
...
prefix: process.env.BULL_PREFIX ?? 'bull',
```

- Uses `BULL_PREFIX` env var to namespace the queue keys in Redis. Defaults to `'bull'` if unset.
- Reads Redis connection details from environment (see §3).

### 2.2 [backend/src/queues/simulation.processor.ts](../repos/rule-studio/backend/src/queues/simulation.processor.ts)

The actual worker that processes simulation jobs:

```ts
import { OnWorkerEvent, Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';
...
throw error; // re-throw so BullMQ marks the job as failed
```

- Extends `WorkerHost` — a BullMQ worker that consumes from the queue.
- Uses `OnWorkerEvent` decorators for worker lifecycle hooks.
- Explicitly re-throws errors so BullMQ retry/backoff semantics engage.

### 2.3 [backend/src/services/send-to-dems/send-to-dems.service.ts](../repos/rule-studio/backend/src/services/send-to-dems/send-to-dems.service.ts)

The producer side — enqueues simulation jobs bound for DEMS:

```ts
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
...
// Actual message processing is handled by SimulationProcessor (BullMQ worker).
```

Line 17's comment (verbatim from the source) is a direct statement that the DEMS send flow is asynchronous via a BullMQ worker. This is not vestigial code.

### 2.4 [backend/test/services/send-to-dems.service.spec.ts](../repos/rule-studio/backend/test/services/send-to-dems.service.spec.ts)

```ts
import { getQueueToken } from '@nestjs/bullmq';
```

Test suite mocks the queue token to inject a fake queue — standard NestJS BullMQ testing pattern. Confirms the queue is a first-class DI dependency, not an optional bolt-on.

[Back to top](#top)

---

## 3. Runtime contract with Redis / valkey

BullMQ requires a Redis-compatible server. The backend reads these env vars at boot (verified by grep for `process.env` in `backend/src/`):

| Env var | Purpose | Required? |
|---------|---------|-----------|
| `REDIS_HOST` | Redis host (or valkey — protocol-compatible) | Yes — module bootstrap fails if unreachable |
| `REDIS_PORT` | Redis port | Yes |
| `REDIS_PASSWORD` | Auth (may be empty for unauthenticated deploys) | Yes (empty string acceptable) |
| `BULL_PREFIX` | Queue-name prefix in Redis | Optional — defaults to `'bull'` |

The socket.io-redis adapter (WebSocket broadcast across replicas) reads the same trio, so a single Redis instance covers both concerns.

[Back to top](#top)

---

## 4. Recency — how current is this code

`git log -1` on the dev branch returns:

```
ef48726 Merge pull request #132 from tazama-lf/feat-paysys-fix-simulation
```

The BullMQ integration is part of the most recent merge on dev. This is not an old subsystem that survived by inertia — it is the current simulation pipeline.

Additional signal: the branch name that produced the latest merge is `feat-paysys-fix-simulation`. The simulation flow (which BullMQ powers) was actively being fixed as of the last merge, not deprecated.

[Back to top](#top)

---

## 5. Implications for FSDT deployment

Before this investigation, [FSDT-Gaps-Pre-Release.md §5.3](FSDT-Gaps-Pre-Release.md#53-backend-env--config-gaps) listed Redis as a required-at-boot env for `trs-backend`. This report confirms that finding independently:

- FSDT's committed [extensions/env/trs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/trs.env) prior to `feat-paysys-fsdt-fixes` did not set `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD`.
- Without those, `BullModule` (and by extension `SimulationProcessor` and `SendToDemsService`) cannot initialise. NestJS boot fails and the container crashloops.
- Commit `b776b07` on branch `feat-paysys-fsdt-fixes` adds the three Redis vars pointed at the existing `valkey` service. This is the correct fix.

`BULL_PREFIX` is not set in FSDT. It falls back to `'bull'`. On a shared valkey instance, this can collide with any other service that also uses BullMQ with default prefix. Not a blocker — but see §8.

[Back to top](#top)

---

## 6. Sim-studio boundary — confirmed clean

The subtree [src/services/simulation-studio/](../repos/rule-studio/backend/src/services/simulation-studio/) (44 files across 11 sub-modules) is **completely BullMQ-free.** Verified by two independent scans:

```
grep -rn -E "bullmq|BullMQ|BullModule|InjectQueue|@Processor|WorkerHost" src/services/simulation-studio/
(no output)
```

Sim-studio's outbound imports go only to `auth/`, `admin-service-client`, `msg-sample-generation/`, and its own sub-modules. It does NOT import `send-to-dems`, `simulation` (the OLD module), or `queues/`.

Sim-studio runs simulations synchronously via **testcontainers** — [ephemeral-env.service.ts:233,289,358](../repos/rule-studio/backend/src/services/simulation-studio/ephemeral-env/ephemeral-env.service.ts) spins up a `GenericContainer` and holds it as a `StartedTestContainer`. The variable name `ruleProcessor` in that file refers to the container handle, not a BullMQ processor. Completely different mechanism.

**Conclusion:** BullMQ and sim-studio do not cross paths in the request graph. Sim-studio itself matches the intended design.

[Back to top](#top)

---

## 7. Intent gap — `send-to-dems` should not use BullMQ (future work)

**Design intent:** `SendToDemsService` was supposed to be independent of any queue mechanism — a synchronous DEMS call, or driven by something other than BullMQ. That refactor is **not present on `dev` at `ef48726`.**

### Verified evidence that `send-to-dems` is still fully BullMQ-coupled

Direct reads of the four files in [src/services/send-to-dems/](../repos/rule-studio/backend/src/services/send-to-dems/):

**[send-to-dems.service.ts](../repos/rule-studio/backend/src/services/send-to-dems/send-to-dems.service.ts) — the entire service is a BullMQ producer:**

- L2-3: `import { InjectQueue } from '@nestjs/bullmq';` / `import { Queue } from 'bullmq';`
- L5-6: imports `SIMULATION_QUEUE`, `SIMULATION_JOB`, `DirectSimulationMessage` from [src/queues/simulation-queue.constants.ts](../repos/rule-studio/backend/src/queues/simulation-queue.constants.ts).
- L13: constructor `@InjectQueue(SIMULATION_QUEUE) private readonly simulationQueue: Queue`.
- L17 comment (verbatim): *"Actual message processing is handled by SimulationProcessor (BullMQ worker)."*
- L24: `await this.simulationQueue.add(SIMULATION_JOB, { jobId, token, tableNames });`
- L45: `await this.simulationQueue.add(SIMULATION_JOB, { jobId, token, messages, tableName, tenantId, totalMessages });`

Both public methods (`enqueueSimulation`, `enqueueDlhSimulation`) do exactly one thing: push a job onto the BullMQ queue and return the `jobId`. **There is no synchronous DEMS call path anywhere in this file.**

**[send-to-dems.module.ts](../repos/rule-studio/backend/src/services/send-to-dems/send-to-dems.module.ts):**

- L6: `import { QueuesModule } from '../../queues/queues.module';`
- L8: `import { SimulationProcessor } from '../../queues/simulation.processor';`
- L12: `imports: [HttpModule, QueuesModule, GatewaysModule, FetchEvaluationModule]` — pulls in BullMQ.
- L13: `providers: [SendToDemsService, AdminServiceClient, SimulationProcessor]` — registers the BullMQ worker inside this module.

**[send-to-dems.controller.ts](../repos/rule-studio/backend/src/services/send-to-dems/send-to-dems.controller.ts):**

- L19: `@HttpCode(HttpStatus.ACCEPTED)` — endpoint returns 202 by design.
- L22-25 Swagger description (verbatim): *"Enqueues a background simulation job and returns a `jobId` immediately. Connect to the WebSocket namespace `/simulation`, emit `joinJob` with `{ jobId }` to subscribe to real-time progress updates."*
- L35: delegates straight to `sendToDemsService.enqueueSimulation(...)`.

The Swagger description is unambiguous: the endpoint is contractually async-via-queue.

### Two downstream callers, both async-by-design

`SendToDemsService` is consumed from outside sim-studio at two call sites, both of which likewise operate through the BullMQ path:

- [fetch-from-dlh.service.ts:168,192](../repos/rule-studio/backend/src/services/fetch-from-dlh/fetch-from-dlh.service.ts) — calls `sendToDemsService.enqueueDlhSimulation(...)`. Its module imports `SendToDemsModule` and the OLD `SimulationService`.
- [rerun-simulation.service.ts:60](../repos/rule-studio/backend/src/services/rerun-simulation/rerun-simulation.service.ts) — calls `sendToDemsService.enqueueDlhSimulation(...)`. Its module imports `SendToDemsModule`.

Neither module lives under `simulation-studio/`. Both go through BullMQ.

### Git history — no BullMQ removal on `dev`

`git log src/services/send-to-dems/` on the current `origin/dev`:

```
a645dc6 fix: prettier check
11c7e5b fix: update branch with dev
9803720 feat: a record gets pushed in the trs_simulation as well
34049ef feat: data is being saved in simxyz_results table.
7129b46 feat: add code that support non pasc transactions
e89f0d4 feat: able to run entire flow with harcdcoded endpointPath. Facing BULLMQ inconsistency while picking up from queue
c9a27f2 feat: included logs with progress emssion
c5b9286 feat: websocket init
```

`git log src/queues/`:

```
099af69 fix: fixed lint issues and refactored fetchAllFromDlh to use promise.all
7a38e25 Merge remote-tracking branch 'origin/feat-paysys-fix-pagination' into feat-paysys-simulation
1e49afe fix: fixed the BULLMQ issue
6e33582 feat: removed DataCache from the payload to be ingested
```

The history shows BullMQ being **added, patched, and iterated on** — not removed. There is no commit anywhere on the `dev` history that decouples `send-to-dems` from BullMQ.

### Future-work item

The BullMQ coupling in `send-to-dems` (and by transitive dependency, `fetch-from-dlh` and `rerun-simulation`) is intended to be removed. Until that lands:

- `trs-backend` cannot boot without Redis/valkey. `BullModule.forRoot()` runs at app init and fails fast on connection error.
- Any client calling `POST /send-to-dems/simulate` gets a 202 + jobId and must subscribe to the `/simulation` WebSocket namespace for progress. Synchronous consumers will not work.
- `fetch-from-dlh` and `rerun-simulation` inherit the same async contract because they call `enqueueDlhSimulation`.

### What the refactor should look like (for the future PR)

Concretely, the refactor should:

1. Replace `simulationQueue.add(...)` in `SendToDemsService` with a direct DEMS HTTP call (or whatever the intended new mechanism is).
2. Change the controller from `@HttpCode(202)` + jobId to a synchronous response (or keep async but move to a different backing mechanism).
3. Remove `QueuesModule`, `SimulationProcessor` provider, and the `@nestjs/bullmq` / `bullmq` deps from `SendToDemsModule` and (if no other consumer remains) from `package.json`.
4. Update `fetch-from-dlh` and `rerun-simulation` to match the new contract.
5. Once every BullMQ consumer is gone, delete [src/queues/](../repos/rule-studio/backend/src/queues/) entirely and drop the two BullMQ deps.
6. Once no consumer needs Redis at all, the Redis env in FSDT's `trs.env` (added by commit `b776b07`) becomes dead weight and can be removed.

[Back to top](#top)

---

## 8. Recommendations

1. **Keep the Redis vars in `trs.env`** exactly as committed in `b776b07` — they are load-bearing today. Only remove them after the §7 refactor lands.
2. **Optionally add `BULL_PREFIX=trs`** (or `rule-studio`, matching whatever naming scheme is used in the valkey key-space) to `trs.env`. This namespaces the queue keys and prevents collisions if another service ever adopts BullMQ against the same valkey instance. Not required today.
3. **Do not remove `@nestjs/bullmq` or `bullmq`** from `backend/package.json` yet — they are wired into the active `send-to-dems` flow. Removal is a follow-up after §7.
4. **When touching the simulation pipeline in future PRs**, remember that TODAY "send to DEMS" is asynchronous by design (queue → worker → external call). Any refactor that assumes synchronous HTTP-style semantics from `SendToDemsService.enqueue()` will silently break until §7 lands.
5. **Track §7 as an explicit follow-up task.** The BullMQ coupling in `send-to-dems` is a known intent gap — not an accident to be worked around and not to be treated as the desired end state.

[Back to top](#top)
