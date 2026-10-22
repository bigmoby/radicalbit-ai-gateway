# Radicalbit AI Gateway Core — a deployment profile for edge installations

**Status:** proposal / preliminary static analysis, not yet validated end-to-end. The findings in §5 were checked against the code (`main` @ `9b901ec`, 2026-10-22), including a real `alembic upgrade head` run against SQLite.  
**Audience:** the Radicalbit team and the maintainers of the open source project  
**Upstream repository:** https://github.com/radicalbit/radicalbit-ai-gateway  
**Target:** edge installations — single-node, resource-constrained, often ARM64, frequently unattended. The first reference device is a Home Assistant Yellow (Raspberry Pi Compute Module 4, 4 GB RAM, eMMC storage), but the profile is deliberately device- and vendor-neutral.

## 1. Executive summary

This document proposes **AI Gateway Core**: an opt-in deployment profile of Radicalbit AI Gateway designed for edge installations. Core is the real gateway — not a rewritten proxy and not an integration built for a single product — running without the heavy observability stack that only makes sense in a data center.

The value proposition for the edge is that the governance an LLM gateway provides matters *more*, not less, when the gateway runs far from an operations team:

- **Cost control at the point of use.** Rate, token and budget limits are enforced locally, on the device that generates the traffic, so a misbehaving automation cannot run up a provider bill unnoticed.
- **Policy enforcement close to the data.** Guardrails run before a prompt leaves the site, which matters for homes, branches, plants and other locations where the data is generated.
- **Resilience to upstream failures.** Routing and fallback across providers keep local applications working when one provider is slow or unavailable.
- **Lower latency and fewer paid calls.** Exact and semantic caching answer repeated requests on the device itself.
- **Operating without an ops team.** Alerting tells a remote owner that something is wrong without needing dashboards, tracing or an analytics database.
- **One API for every local client.** Applications, home automation, scripts and agents all talk to the same OpenAI-compatible endpoint, with per-project and per-credential isolation.

Core **must keep**: the OpenAI-compatible API, routing, guardrails, management of projects/routes/credentials/groups, **rate limiting**, **token limiting**, **budget limiting** (route, project, credential), **exact and semantic caching**, **alerting** and the configuration UI. Candidates for exclusion are ClickHouse, the OpenTelemetry Collector, the metrics-worker, the historical analytics pipeline and distributed tracing.

Target architecture:

```text
Local clients: applications, agents, home/building automation, scripts
                  |
                  v
       Radicalbit AI Gateway Core
       - LLM API and routing
       - guardrails and credentials
       - rate/token/budget limiting
       - exact and semantic caching
       - operational alerting
       - configuration UI
                  |
          +-------+--------+
          |                |
          v                v
       SQLite         Valkey Bundle
   configuration     cache, counters,
   and rules         alert windows,
                     vector indexes
                  |
                  v
        LLM providers (cloud or on-premises)
```

SQLite would live on a persistent volume (for example `/data/radicalbit/gateway.db`); its actual location and journaling settings must be defined and tested on flash storage. Valkey Bundle is the candidate for unifying limiter storage, cache and alerts; **ARM64 compatibility and the sufficiency of a single instance are not yet verified**.

## 2. What "edge" means for this proposal

Core targets installations with most of the following traits:

| Trait | Consequence for the gateway |
|---|---|
| Single node, no cluster | No external databases or message brokers; everything runs in a few containers on one host |
| Constrained resources (2–8 GB RAM, few cores) | Small memory footprint; no analytics engine or collector |
| ARM64 as well as x86-64 | Multi-arch images; no native dependency that fails to build on ARM64 |
| Flash storage (eMMC, SD, SSD) | Few, small writes; sensible SQLite journaling and Valkey persistence settings |
| Unattended operation | Safe behaviour after power loss and restarts; self-explanatory alerts |
| Intermittent or metered connectivity | Clear failure modes when providers are unreachable; fallback routing; caching |
| Located on the user's premises | Admin UI not exposed by default; credentials stored safely; easy backup and restore |

Examples: home servers and home automation hubs (the initial reference is Home Assistant Yellow), industrial and IoT gateways, retail and branch-office appliances, on-premises boxes for small organizations, and developer laptops for local development.

## 3. Principles for the upstream proposal

1. **Full stays the default**: no regression for existing deployments.
2. **Core is opt-in**: an explicit profile, for example `GATEWAY_PROFILE=core` (name and configuration interface to be agreed; this is not an existing upstream variable).
3. **One body of business logic**: no duplication of routing, guardrails, limiters, budgets, caching or alerting.
4. **Separation of responsibilities**: distinguish operational events and alerts from the persistence of events for analytics.
5. **No device- or vendor-specific code upstream**: packaging for a given platform (for example a Home Assistant App) and platform-specific notifications live in separate integrations.
6. **Tests as the acceptance criterion**: a dependency being optional in the code does not, on its own, prove runtime correctness.

## 4. Non-negotiable functional requirements

| Capability | Core | Note |
|---|---|---|
| OpenAI-compatible API and streaming | Required | Tested with real requests |
| LLM routing and fallback | Required | Verify the error paths, including unreachable providers |
| Guardrails | Required | Including events that trigger alerts |
| Management of projects, routes, credentials and groups | Required | SQLite persistence |
| Rate limiting | Required | Valkey; concurrency and window resets |
| Token limiting | Required | Input/output and streaming |
| Budget limiting | Required | Route, project and credential; verify cost accounting |
| Budget-aware routing | Required if available in Full | No regression |
| Exact cache | Required | Hit/miss, TTL, isolation |
| Semantic cache | Required | Vector search; Valkey Bundle modules |
| Immediate and windowed alerts | Required | Must not depend on the analytics pipeline |
| Existing email notifications | Keep | Platform-specific webhooks in a separate PR |
| ClickHouse analytics history | Excluded | Endpoints/UI must handle its absence explicitly |
| OpenTelemetry tracing | Excluded | No Collector needed |

## 5. Findings verified in the upstream code

### 5.1 SQLite is anticipated, but not actually supported

- [`db/database.py`](../../gateway/radicalbit_ai_gateway/db/database.py) builds a `sqlite:///...` URL when `db_type == 'sqlite'`.
- [`utils/app_config.py`](../../gateway/radicalbit_ai_gateway/utils/app_config.py) declares `db_type: str = 'sqlite'` as the configuration class default; the official Compose file sets PostgreSQL instead.
- [`alembic/env.py`](../../gateway/alembic/env.py) special-cases PostgreSQL in the online migration path, but that does not make every revision portable.
- **Real run:** `alembic upgrade head` against a SQLite file already fails at revision [`1a2b1d01d33e_add_groups.py`](../../gateway/alembic/versions/1a2b1d01d33e_add_groups.py) (line 24, `op.drop_constraint`) with `NotImplementedError: No support for ALTER of constraints in SQLite dialect`. No revision uses `batch_alter_table`, and `env.py` does not set `render_as_batch`.
- [`d1e2f3a4b5c6_add_project_uuid_to_group_routes.py`](../../gateway/alembic/versions/d1e2f3a4b5c6_add_project_uuid_to_group_routes.py) uses `create_foreign_key`, `drop_constraint`, `create_primary_key` and `alter_column`: it needs to be adapted for SQLite, typically with Alembic batch mode.
- [`e0f1a2b3c4d5_alert_rule_scopes.py`](../../gateway/alembic/versions/e0f1a2b3c4d5_alert_rule_scopes.py) contains PostgreSQL-specific SQL (`p."UUID"::text` and `UPDATE ... FROM`) that must be rewritten or split by dialect.
- Other PostgreSQL-bound revisions: `b3c4d5e6f7a8_add_project_config_table.py` and `ca3e44546368_rename_user_key_to_key.py` use `postgresql.ENUM` / `postgresql.TIMESTAMP`; `c4d5e6f7a8b9_partial_unique_project_name.py` and `b3c4d5e6f7a8` define partial unique indexes with `postgresql_where`. **On SQLite `postgresql_where` is silently ignored**: the index becomes unique over the whole table (for example, a soft-deleted project would block reuse of its name). The `sqlite_where` equivalent is needed.
- **ORM not initialized on SQLite:** in [`server.py`](../../gateway/radicalbit_ai_gateway/server.py), `database.init_mappings()` is only called when `db_type == 'postgresql'`. The tables (for example `Group`, `KeyLimit`, `ProjectBudgetLimit`) inherit from `Reflected` (`DeferredReflection`), which is not mapped without `prepare()`: on SQLite the ORM almost certainly does not work even once the migrations are fixed.
- **Connection pool not suited to SQLite:** `Database.connect()` uses `create_engine(..., pool_size=20, max_overflow=40)`, sized for PostgreSQL; for SQLite on flash storage the pool, lock timeouts and WAL must be defined.
- **No test coverage:** the suite in `gateway/tests` has no references to SQLite.

**Conclusion:** only SQLite URL construction exists; **migrations (verified: they fail), ORM mappings and pool configuration are not supported**. Prefer portable migrations over creating the schema from the models, which would not solve future upgrades.

### 5.2 ClickHouse is wired into the code, but probably does not block startup

[`server.py`](../../gateway/radicalbit_ai_gateway/server.py) builds `ClickHouseDatabase`, `EventDAO`, `RequestEventDAO`, `OtelTracesDAO` and `AlertDAO`; the lifespan calls `ch_database.connect()`. However, [`ClickHouseDatabase.connect()`](../../gateway/radicalbit_ai_gateway/db/clickhouse_database.py) only calls `create_engine()`, which opens no connections: startup without ClickHouse will most likely **succeed**, and failures would only surface on the first analytics query (not yet verified with a real startup).

Refactoring is still needed, but for reasons other than unblocking startup: conditional initialization, optional dependencies and an explicit response from the unavailable analytics APIs instead of runtime errors.

The OTLP Collector is already conditional on the `collector_base_url` setting; that does not imply that every other tracing/telemetry dependency is absent.

### 5.3 Rate limiting and budgets can use Valkey

The modules [`limiting/rate_limiter.py`](../../gateway/radicalbit_ai_gateway/limiting/rate_limiter.py), [`limiting/budget_limiting.py`](../../gateway/radicalbit_ai_gateway/limiting/budget_limiting.py) and [`limiting/project_budget_limiter.py`](../../gateway/radicalbit_ai_gateway/limiting/project_budget_limiter.py) use `RedisStorage` when Redis/Valkey is configured, with an in-memory fallback. The operational counters do not need ClickHouse.

**Caveats:** checking and consuming a limit may happen in separate steps; test concurrency and thresholds. Budgets are accounted from token/duration costs: verify models, streaming, cache hits and interrupted requests. Do not confuse operational enforcement with historical analytics.

### 5.4 Exact and semantic caching

[`utils/gateway_route_factory.py`](../../gateway/radicalbit_ai_gateway/utils/gateway_route_factory.py) creates the Redis client when caching is configured. The semantic cache in [`caching/semantic_caching.py`](../../gateway/radicalbit_ai_gateway/caching/semantic_caching.py) uses `VectorField`, `FT.CREATE` indexes, `FT.SEARCH` queries, TTLs and filters by project, route and credential.

**Conclusion:** the semantic cache needs vector search support; a plain Valkey instance without the right modules is not enough. Verify Valkey Bundle on ARM64, versions/modules, index initialization and behaviour after a restart.

### 5.5 Alerts are already separable from analytics persistence

[`events/events_processor.py`](../../gateway/radicalbit_ai_gateway/events/events_processor.py) hands events to the `AlertDispatcher` on a separate executor **before** adding them to the Celery buffer. [`alerting/alert_dispatch.py`](../../gateway/radicalbit_ai_gateway/alerting/alert_dispatch.py) handles immediate and windowed events; [`alerting/alert_store.py`](../../gateway/radicalbit_ai_gateway/alerting/alert_store.py) supports Redis/Valkey and an in-process fallback for a single replica.

The only part of alerting that reads from ClickHouse is the per-rule history: `AlertRuleService` → `alert_dao.get_history(...)` ([`services/alert_rule_service.py`](../../gateway/radicalbit_ai_gateway/services/alert_rule_service.py)). Real-time alerts do not depend on it; in Core the history must be reported as unavailable.

Without a broker, the [`CeleryBuffer`](../../gateway/radicalbit_ai_gateway/events/buffer.py) catches the `send_task` exception and only logs the error: it does not break the flow, but it would fill the logs on every flush — and on flash storage, every log line is a write. In Core the sink must be **disabled**, not left to fail.

**Conclusion:** keep the dispatcher, rules, scheduler and notifications; disable only the publication of events to the analytics pipeline and handle the absence of the alert history.

### 5.6 Python dependencies

`psycopg2` (the source distribution, not `-binary`) is a mandatory runtime dependency ([`pyproject.toml`](../../gateway/pyproject.toml)) and must be compiled against libpq; in a local test the build failed with Python 3.14. Core would therefore still need libpq on the ARM64 image, unless the PostgreSQL driver becomes an optional dependency.

## 6. Components: Full vs Core

| Component | Full | Proposed Core |
|---|---|---|
| Python gateway + UI | Yes | Yes |
| PostgreSQL | Yes | No, replaced by SQLite |
| SQLite | Not in the official Compose | Yes, persistent |
| Valkey | Yes | Yes |
| Valkey Bundle for the semantic cache | Separate service in the Compose | Candidate single instance, to be verified |
| ClickHouse | Yes | No |
| ClickHouse migrations | Yes | No |
| Metrics worker / Celery consumer | Yes | No |
| OpenTelemetry Collector | Yes | No |
| Adminer, Redis Insight, CH-UI | Auxiliary tools | No |

**Note:** `valkey` and `valkey-cache` are separate services in the upstream Compose. Merging them into one is a hypothesis to be tested, not a proven compatibility.

## 7. Feasibility matrix

| Area | Static assessment | Blocker / test still needed |
|---|---|---|
| SQLite — connection | URL only; pool sized for PostgreSQL | Pool, lock timeout, WAL; test with a persistent DB |
| SQLite — ORM mappings | Not supported (`init_mappings` only for PostgreSQL) | Call `prepare()` for SQLite too; CRUD tests |
| SQLite — migrations | **Fail** (verified, revision `1a2b1d01d33e`) | Batch mode, `sqlite_where`, ENUM; `upgrade head` and incremental upgrades in CI |
| Startup without ClickHouse | Probably succeeds (lazy engine); runtime errors | Real startup; explicit response from analytics APIs |
| Startup without Celery worker | Plausible; the buffer logs errors on every flush | Disable the sink without losing alerts |
| Python dependencies | Source `psycopg2` is mandatory | Make the PostgreSQL driver optional or ship libpq |
| Rate limiting | Existing Redis implementation | Concurrency, thresholds, windows |
| Token limiting | Existing limiter implementation | Streaming and accounting |
| Budget limiting | Existing Redis implementation | Costs, scopes, persistence, resets |
| Budget-aware routing | Existing implementation | End-to-end tests |
| Exact cache | Existing implementation | TTL, isolation, hit/miss |
| Semantic cache | Existing vector implementation | Modules, ARM64, embeddings, index |
| Operational alerting | Separation already in place | Email, scheduler, thresholds and absence of ClickHouse |
| Core UI | To be designed | Hide/handle analytics sections |
| Edge hardware (ARM64, reference: Home Assistant Yellow) | Not verified | Build, startup, RAM, CPU, flash writes |

## 8. Upstream PR plan

### PR 1 — Decoupling observability

- Introduce an explicit profile or capability setting (name to be discussed).
- Keep Full unchanged by default.
- Make the ClickHouse connection, DAOs and dependent services optional.
- Separate the emission of operational/alert events from the Celery/analytics sink, disabling the `CeleryBuffer` in Core.
- Keep the alert scheduler and dispatcher; handle the absence of the alert history (`get_history`).
- Define how analytics endpoints respond in Core (explicitly unavailable, with no startup errors).
- Full regression tests and Core tests without analytics services.

### PR 2 — Real SQLite support

- Initialize the ORM mappings (`Reflected.prepare`) for SQLite too and adapt the pool configuration.
- Port the Alembic revisions to SQLite-compatible operations (batch mode where required, starting from `1a2b1d01d33e`).
- Rewrite PostgreSQL-specific SQL or split it by dialect; add `sqlite_where` alongside `postgresql_where` in partial indexes; handle `postgresql.ENUM`.
- Add an `alembic upgrade head` test against SQLite to CI.
- Evaluate making `psycopg2` an optional dependency.
- Test fresh schemas, incremental upgrades, foreign keys, integrity and transactions.
- Test CRUD for projects, configurations, credentials, groups, budgets and alert rules.
- Evaluate WAL, lock timeouts, connections, backup and restore.

### PR 3 — Core deployment and integration tests

- Provide a documented Core Compose file without the analytics stack, with multi-arch (ARM64 and x86-64) images.
- Test exact and semantic caching with Valkey Bundle.
- Test rate/token/budget limiting, budget-aware routing, alerting and streaming.
- Run smoke and load tests on ARM64 edge hardware.
- Document the capabilities present/absent, resource requirements and operating limits.

**Contribution strategy:** first open an RFC/GitHub Discussion to agree on scope and design, then small, reviewable PRs. Platform-specific packaging (for example the Home Assistant App) stays in a separate repository.

## 9. Mandatory end-to-end acceptance scenarios

1. **Minimal boot:** only the gateway, SQLite and Valkey Bundle; no connection to ClickHouse, the Collector or a worker.
2. **Persistence:** create a project, route, credential, group and rule; restart; verify they are all present.
3. **LLM:** standard and streaming chat requests, routing and fallback.
4. **Rate limit:** exceed a threshold, verify the block, the alert and the reset in the next window.
5. **Budget:** input/output/duration costs, route/project/credential thresholds, budget-aware routing, windows and concurrency.
6. **Exact cache:** first miss, subsequent hit, TTL, isolation between credentials and projects.
7. **Semantic cache:** vector index created, similar-query lookup, similarity threshold, TTL and restart.
8. **Alerts:** guardrails, fallback, limits, thresholds and time aggregations; email and no duplicates.
9. **UI/API:** full configuration and CRUD; analytics endpoints clearly disabled.
10. **Reliability:** Valkey unavailable, container restarts, concurrent load, RAM usage and flash writes.
11. **Edge conditions:** power loss during writes, upstream providers unreachable or slow, restart with no network, sustained operation for days without intervention.
12. **Full regression:** existing suite unchanged, including analytics and tracing.

## 10. Open questions for the team

- **Single profile or independent feature flags?** A Core profile simplifies deployment, but flags for the event sink, tracing and analytics could make the system more composable. Avoid unsupported combinations.
- **Long-term SQLite portability:** maintaining two dialects means ongoing maintenance of the migrations.
- **Budget and cache:** verify precisely whether and how a response served from the cache affects rate, token and budget limits; document the intended semantics.
- **Valkey persistence:** decide on RDB/AOF, TTLs, eviction policy and what happens to budgets after a restart or loss of counters. Budgets must behave safely and predictably, especially on devices that lose power.
- **Alerts:** which events need only operational state and which query ClickHouse? How can notifications be made reliable across restarts?
- **Semantic cache:** cost of embeddings, compatibility of the vector engine and the memory available on small devices.
- **Python dependencies / ARM64:** builds of native packages and availability of Valkey Bundle images for the target architectures.
- **Security and maintenance:** credentials, SQLite backups, schema upgrades, access to the UI on a device sitting on a local network, and compatibility with upstream release cycles.
- **Platform integrations:** notifications through webhooks/events and platform UIs (for example Home Assistant) to be evaluated once Core is supported upstream, not in the first PR.

## 11. Final considerations

**The proposal is architecturally promising, but not yet verified end-to-end.** Inspecting the repository showed that rate limiting, budgets, caching and alerting already have operational paths based on Redis/Valkey that can be separated from analytics storage in ClickHouse. That is a good foundation for an upstream Core profile without duplicating business logic — and it means edge installations get the same governance as a data center deployment.

The two main pieces of work are: **(1) decoupling the ClickHouse/Celery initialization and services from the LLM path and from alerts; (2) making SQLite genuinely supported, which is the heaviest part today: Alembic migrations (verified to fail), ORM mappings, the connection pool and the `psycopg2` dependency**. Decoupling from ClickHouse looks less invasive than expected, because the engine is lazy and startup probably does not depend on it being reachable. The semantic cache adds a further requirement: real vector search support, validated on ARM64.

**Recommended decision:** propose to the Radicalbit team an RFC for an opt-in Core profile aimed at edge installations, with Full unchanged, and proceed with incremental, test-driven PRs. Do not declare the solution ready for any edge device, including the Home Assistant Yellow reference, before a real startup and functional and performance tests.

**Success criterion:** a deployment made of gateway + SQLite + Valkey Bundle, running on a single edge node, that serves LLM requests with routing, guardrails, rate/token/budget limiting, exact/semantic caching and alerting, with no runtime dependency on PostgreSQL, ClickHouse, the OTel Collector or the metrics-worker.
