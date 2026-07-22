# Open PRs targeting `dev`

Snapshot: 2026-07-21. Only PRs open against `dev` are listed. Descriptions are derived from the actual diff vs `dev`, not just PR titles/bodies.

---

## case-management-system

- [#38 — build(deps-dev): bump @nestjs/cli from 11.0.7 to 11.0.10](https://github.com/tazama-lf/case-management-system/pull/38) — dependabot, `dependabot/npm_and_yarn/nestjs/cli-11.0.10`
  Dependabot devDependency bump: `@nestjs/cli` 11.0.7 → 11.0.10.

- [#26 — build(deps-dev): bump @swc/core from 1.13.2 to 1.13.5](https://github.com/tazama-lf/case-management-system/pull/26) — dependabot, `dependabot/npm_and_yarn/swc/core-1.13.5`
  Dependabot devDependency bump: `@swc/core` 1.13.2 → 1.13.5.

- [#16 — build(deps-dev): bump @nestjs/schematics from 11.0.5 to 11.0.7](https://github.com/tazama-lf/case-management-system/pull/16) — dependabot, `dependabot/npm_and_yarn/nestjs/schematics-11.0.7`
  Dependabot devDependency bump: `@nestjs/schematics` 11.0.5 → 11.0.7.

- [#13 — build(deps-dev): bump @swc/cli from 0.6.0 to 0.7.8](https://github.com/tazama-lf/case-management-system/pull/13) — dependabot, `dependabot/npm_and_yarn/swc/cli-0.7.8`
  Dependabot devDependency bump: `@swc/cli` 0.6.0 → 0.7.8.

- [#12 — build(deps-dev): bump typescript from 5.8.3 to 5.9.2](https://github.com/tazama-lf/case-management-system/pull/12) — dependabot, `dependabot/npm_and_yarn/typescript-5.9.2`
  Dependabot devDependency bump: `typescript` 5.8.3 → 5.9.2.

## biar

- [#103 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/biar/pull/103) — Justus-at-Tazama, `sync-workflows-update`
  Automated sync from `central-workflows`. Pins the `actions/checkout` step in `package-rule-rc.yml` (and sibling RC build workflows) to `ref: dev`, and relaxes the "RC must publish from dev" guard so it only fires on `push` events — `repository_dispatch`/`workflow_dispatch` triggers now rely on the pinned checkout instead.

- [#52 — fix: update TAZAMA_WAREHOUSE_HOST_PATH to /opt/Warehouse](https://github.com/tazama-lf/biar/pull/52) — Justus-at-Tazama, `fix/update-warehouse-host-path`
  One-line `.env.example` change repointing `TAZAMA_WAREHOUSE_HOST_PATH` from `/opt/Tazama_Warehouse` to `/opt/Warehouse` to match the new Data Lakehouse directory layout. No code or compose changes.

## connection-studio

*(Note: repo name is `connection-studio`, not `connections-studio`.)*

- [#95 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/connection-studio/pull/95) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync as biar#103 — pins checkout to `ref: dev` and scopes the RC-from-dev guard to `push` events only.

- [#89 — feat: TCS DEMS Config Event Type Required](https://github.com/tazama-lf/connection-studio/pull/89) — MuhammadAli-Paysys, `feat-paysys-msgFam-required`
  Makes `msgFam`, `version`, `schema`, and `payload` required (removes `@IsOptional`, drops the `endpointPath` field) in `CreateConfigDto`, matching the backend requirement. `ConfigService.create` now runs `validatePayloadContent` (new AJV-backed helper in `utils/helpers.ts`) before persisting and short-circuits with a friendly error on invalid JSON/XML; falsy `msgFam` falls back to `'unknown'` via `||` instead of `??`.

- [#74 — fix: JSON/XML payload validation enhanced](https://github.com/tazama-lf/connection-studio/pull/74) — MuhammadAli-Paysys, `feat-paysys-payload-fix`
  Earlier version of the same payload-validation work as #89 — introduces `validatePayloadContent` in `utils/helpers.ts` (rejects arrays/non-object JSON, validates XML strings), wires it into `ConfigService.create`, tightens `CreateConfigDto` requiredness, drops `dotenv.config()` from helpers, and enables `ignoreAttributes` in the simulation XML parser so `id="…"` attrs don't pollute generated fields.

## rule-studio

- [#130 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/rule-studio/pull/130) — Justus-at-Tazama, `sync-workflows-update`
  Same automated central-workflows sync — pins RC-build checkouts to `ref: dev` and scopes the "must publish from dev" guard to `push` events only.

- [#126 — feat: add variable with data type node](https://github.com/tazama-lf/rule-studio/pull/126) — ReebaPaysys, `feat-paysys-variable-type-node`
  Adds a new `SetVariableWithType` flow node to `CodeGenerator.ts` that emits typed TS declarations with `as unknown as <type>` casts for number/boolean/array/object/any, plus interpolation handling. Introduces shared helpers `escapeForInterpolatedTemplate` (correctly escapes `\` then `` ` `` while preserving `${...}`) and `toTsType`, and threads the new node type through `VariableManager` extraction/declaration checks and the validation schema registry.

- [#105 — refactor: Role management in claims](https://github.com/tazama-lf/rule-studio/pull/105) — MuhammadAli-Paysys, `feat-paysys-auth-login`
  Renames RBAC roles `editor`/`approver`/`publisher` to `trs_editor`/`trs_approver`/`trs_publisher` across `permissionMatrix.json` and `NotificationService`. Replaces hardcoded role sets in `TazamaAuthGuard` and `AuthService` with a dynamic `ALLOWED_ROLES` env-driven list (case-insensitive matching, generic "Invalid credentials" errors). Rotates `backend/public-key.pem` to a new JWT verification key.

## admin-service

- [#471 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/admin-service/pull/471) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync — pins `ref: dev` on RC-image and rule-package checkout steps, and gates the "must publish from dev" error to `push` events only.

- [#238 — fix: row level security enabling](https://github.com/tazama-lf/admin-service/pull/238) — cshezi, `feat/rls`
  Pre-query setup for Postgres RLS on admin-service. Bumps `@tazama-lf/frms-coe-lib` from `6.0.0` to `7.0.0-rc.3` (the RLS-aware release that adopts per-call pooled `connect()`+transactions) and refactors config repository work to run only against the configuration DB rather than multiple DBs, letting tenant-scoped RLS predicates apply cleanly.

## Full-Stack-Docker-Tazama

- [#251 — chore: align launcher shell scripts with batch counterparts](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/251) — Justus-at-Tazama, `tazama/feat/shell-script-rewrite`
  Rewrites `biar/tazama-biar.sh`, `core/tazama-core.sh`, and `extensions/tazama-extensions.sh` as faithful ports of their `.bat` equivalents. `biar` gains a real NATS `14222` reachability probe (nc with `/dev/tcp` fallback); `core` drops the Demo UI and Batch PPA addons, renumbers the addon menu to 1–6, switches to `tazama-core` project/container names, tears down sibling stacks on deploy, and points pgAdmin at `localhost:5050`; `extensions` swaps in a Server A/Server B launcher with core-reachability + auto-copy of `test-public-key.pem`.

- [#249 — fix: point CMS_BRANCH to dev to align schema and code](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/249) — ahmad-paysys, `feat-paysys-fsdt-fixes`
  Env-only changes across four files. Repoints `CMS_BRANCH` from `paysys/visualization` to `dev` (dev-mode build only). Fills `trs.env` with Redis (`valkey`) settings, DockerHub credential placeholders, testcontainers/DinD vars (Ryuk disabled), and fixes `IV_LENGTH` from the literal `abcdefghijklmnop` to `12`. Adds `DEMS_URL`/`DEAPI_URL`/`DEMS_STREAM` to `tcs.env` and a full `SIMULATION_DATABASE_*` block to `admin.env` for sim-studio.

- [#245 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/245) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync — pins the RC package checkout to `ref: dev` and scopes the RC-from-dev guard to `push` events only.

- [#140 — feat: row level security](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/140) — rtkay123, `feat/rls`
  Introduces multi-tenant Postgres RLS bootstrap. Splits the Postgres init from a single `00-CREATE.sql` into an ordered `00-DB.sql` → `01-RLS.sh` → `02-CREATE.sql` sequence (with `rls.sql` staged at `/tmp/rls.sql`), and renumbers the config-overlay data files to slots `03/04` across dev/hub/full/multitenant compose files. Adds `postgres/README.md` documenting the `tenant`/`user_tenant` tables, GUC-based tenant resolution, and support-user mapping model.

## frms-coe-lib

- [#438 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/frms-coe-lib/pull/438) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync — checkout pinned to `ref: dev` and the RC-from-dev guard restricted to `push` events.

- [#320 — feat: rls](https://github.com/tazama-lf/frms-coe-lib/pull/320) — rtkay123, `feat/rls`
  Reworks the DB manager to use per-call pooled connections with explicit `BEGIN`/`COMMIT`/`ROLLBACK` and `release()` cleanup so tenant context can be set safely for RLS. Tests are migrated from mocking `.query()` on the pool to mocking `.connect()` returning a mock client, and new failure-path assertions cover error propagation and rollback. Bumps package to `7.0.0-rc.3`.

## rule-executer

*(Note: repo name is `rule-executer`, not `rule-executor`.)*

- [#450 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/rule-executer/pull/450) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync — RC build/package checkouts pinned to `ref: dev` and dev-only guard scoped to `push` triggers.

- [#403 — build(deps): bump library dependencies](https://github.com/tazama-lf/rule-executer/pull/403) — Justus-at-Tazama, `dep/library-dependency-bump`
  Automated library rollout. Bumps `@tazama-lf/frms-coe-lib` `7.0.2-rc.2` → `8.0.0-rc.3`, `@tazama-lf/frms-coe-startup-lib` `3.0.2-rc.2` → `3.0.2-rc.5`, and rule alias `@tazama-lf/rule-901` `4.0.0-rc.0` → `4.0.0-rc.2`. Also collapses `dependabot.yml` to a single weekly npm entry targeting `dev` with `@tazama-lf/*` ignored (rollout handles those), removing the GitHub npm registry block.

## tms-service

- [#434 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/tms-service/pull/434) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync — RC image and rule-package checkouts pinned to `ref: dev`; dev-only publish guard now gated to `push` events.

- [#352 — feat: rls through lib update](https://github.com/tazama-lf/tms-service/pull/352) — rtkay123, `feat/rls`
  Pulls the RLS behaviour transitively by bumping `@tazama-lf/frms-coe-lib` from `7.0.0-rc.0` to `7.0.0-rc.2` in `package.json` and `package-lock.json`. No application-code changes beyond `peer` flag churn in the lockfile.

## event-monitoring-service

- [#50 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/event-monitoring-service/pull/50) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync, plus this repo additionally switches DockerHub image builds to use `secrets.GH_TOKEN_LIB` instead of `secrets.GH_TOKEN` for private-npm auth in both the dev and RC image-build workflows.

- [#43 — feat: Approver claims added for activation](https://github.com/tazama-lf/event-monitoring-service/pull/43) — MuhammadAli-Paysys, `feat-paysys-approver-claim`
  Adds a new `APPROVER: 'approver'` claim to `EventMonitoringClaims` and a `RequireActivationRole()` decorator, then swaps the `PATCH /config-notify/:id` guard from `RequireDemsWriteRole` to `RequireActivationRole` so approvers (not `dems:write`) can toggle publishing status. Expands `dems-engine.service.spec.ts` with many new branch tests (null-mapping, dynamic mapping, invalid function names, `saveTransactionDetails` success/failure, `addDataModelTable` variants) to lift coverage past 95%.

- [#18 — build(deps): bump library dependencies](https://github.com/tazama-lf/event-monitoring-service/pull/18) — Justus-at-Tazama, `dep/library-dependency-bump`
  Automated library rollout — bumps `@tazama-lf/frms-coe-lib` from `7.0.1-rc.ak.17` to `8.0.0-rc.3` and `@tazama-lf/frms-coe-startup-lib` from `3.0.0-rc.ak.17` to `3.0.2-rc.5` in `package-lock.json` (dropping the transitive `@google-cloud/*` tree that the older RC pulled in).

## data-enrichment-service

- [#40 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/data-enrichment-service/pull/40) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync — pins RC image/rule-package checkouts to `ref: dev` and scopes the RC-from-dev guard to `push` events only.

- [#15 — build(deps): bump library dependencies](https://github.com/tazama-lf/data-enrichment-service/pull/15) — Justus-at-Tazama, `dep/library-dependency-bump`
  Automated library rollout — bumps `@tazama-lf/frms-coe-lib` `7.0.0-rc.4ar` → `8.0.0-rc.3` and `@tazama-lf/frms-coe-startup-lib` `3.0.0-proto.0` → `3.0.2-rc.5`, and drops the transitive `@google-cloud/bigquery`/`storage`/`common` tree the old prerelease pulled in.

## frms-coe-startup-lib

- [#291 — ci: sync workflows from central-workflows](https://github.com/tazama-lf/frms-coe-startup-lib/pull/291) — Justus-at-Tazama, `sync-workflows-update`
  Same central-workflows sync — pins the RC rule-package checkout to `ref: dev` and gates the dev-only publish guard to `push` events.
