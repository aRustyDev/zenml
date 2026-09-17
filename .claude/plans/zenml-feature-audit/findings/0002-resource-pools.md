# 0002 — Resource Pools (`zen_stores/resource_pools/`)

Repo `cc064f100`. All paths relative to repo root. Every claim carries `file:line`.

---

## 1. Surface

The most complete Pro-only surface in the repo: **~3,000 lines of models, schemas, routers, CLI and migration shipped in OSS around a hole where the implementation should be.**

| Layer | Path | Size |
|---|---|---|
| ABCs | `src/zenml/zen_stores/resource_pools/store_interface.py:39` `ResourcePoolsStoreInterface` (13 abstract methods, `:45`–`:209`) and `:217` `ResourcePoolsSQLStoreInterface` (+2 more, `:231`, `:242`) | 253 |
| Package init | `src/zenml/zen_stores/resource_pools/__init__.py` | — |
| Pydantic models | `models/v2/core/resource_pool.py` (256), `resource_pool_subject_policy.py` (234), `resource_request.py` (304) | 794 |
| SQL schemas | `zen_stores/schemas/resource_pool_schemas.py` (`:49` Queue, `:107` Allocation, `:176` Pool, `:339` PoolResource), `resource_pool_policy_schemas.py` (`:48` SubjectPolicy, `:233` SubjectPolicyResource), `resource_request_schemas.py` (`:48` Request, `:246` RequestResource) — **8 table classes** | 891 |
| Migration | `zen_stores/migrations/versions/b0ba2c3800e3_add_resource_pools.py` — `revision = "b0ba2c3800e3"` (`:14`), `down_revision = "0.94.2"` (`:15`). Creates 8 tables: `resource_pool` (`:23`), `resource_pool_resource` (`:42`), `resource_pool_subject_policy` (`:62`), `resource_pool_subject_policy_resource` (`:113`), `resource_request` (`:135`), `resource_pool_allocation` (`:196`), `resource_pool_queue` (`:231`), `resource_request_resource` (`:282`) | 348 |
| REST routers | `routers/resource_pools_endpoints.py` (5 ops), `resource_pool_subject_policies_endpoints.py` (5 ops), `resource_requests_endpoints.py` (3 ops) — registered **unconditionally** at `zen_server_api.py:351-353` | — |
| CLI | `cli/resource_pool.py` (608 lines, 9 commands, group at `:39-40`), `cli/resource_request.py` (80 lines, 3 commands, group at `:17-18`); both star-imported at `cli/__init__.py:2581-2582` | 688 |
| Client SDK | `client.py:2292-2560` — 9 methods (`create/get/list/update/delete_resource_pool`, `create/get/update/delete_resource_pool_subject_policy`) | — |
| REST store | `zen_stores/rest_zen_store.py:1341-1538+` — full client-side pass-through | — |
| SQL store facade | `zen_stores/sql_zen_store.py:4086-4272` — 13 delegating methods | — |
| Enum | `enums.py:669` `ResourceRequestStatus` (7 states: pending/allocated/preempting/preempted/cancelled/rejected/released) | — |
| API paths | `constants.py:510-512` `/resource_pools`, `/resource_pool_subject_policies`, `/resource_requests` | — |

---

## 2. Tier verdict — **Tier A (hardest case in the audit)**

**Two ABCs, zero concrete classes.** Verified:

```
$ grep -rn "ResourcePoolsSQLStoreInterface" --include="*.py" .
src/zenml/zen_server/utils.py:72,99,353,379,382     # type annotations + loader
src/zenml/zen_stores/sql_zen_store.py:23,1194,1233,1251   # attribute type
src/zenml/zen_stores/resource_pools/store_interface.py:217 # the ABC itself
```

No subclass anywhere in the repo. The only other reference to the base ABC is `ZenStoreInterface(ResourcePoolsStoreInterface, ABC)` at `zen_stores/zen_store_interface.py:183` — another ABC, which merely propagates the 13 abstract methods into the store contract.

**How the hole is plugged at runtime.** `SqlZenStore._resource_pools` defaults to `None` (`sql_zen_store.py:1194`). The `resource_pools` property raises when unset:

```python
# sql_zen_store.py:1243-1247
if not self._resource_pools:
    raise NotImplementedError("Resource pool functionality is not enabled")
return self._resource_pools
```

`NotImplementedError → HTTP 501` (`zen_server/exceptions.py:95`). Every one of the 13 facade methods (`sql_zen_store.py:4086-4272`) funnels through that property, so **in OSS all 13 REST endpoints return 501** even though FastAPI advertises them in the OpenAPI schema.

The wiring is self-registering: `ResourcePoolsSQLStoreInterface.__init__` does `store.resource_pools = self` (`store_interface.py:228`) against the setter at `sql_zen_store.py:1249-1258`. A Pro implementation therefore attaches itself simply by being constructed.

**Two additional Tier-B-looking booleans that are *not* Tier B:** `SqlZenStore.resource_pools_enabled` (`sql_zen_store.py:1224-1230`) returns `self._resource_pools is not None`, and gates the two live orchestration hooks (§6). It is a downstream consequence of the same nullable source, not a separate switch.

---

## 3. Gate inventory

| # | Mechanism | Where |
|---|---|---|
| **3** | Pluggable implementation source | field `resource_pool_implementation_source: Optional[str] = None` — `config/server_config.py:347`; property `resource_pool_enabled` — `:722-728`; loader `initialize_resource_pool_store()` — `zen_server/utils.py:368-389`, validating `expected_class=ResourcePoolsSQLStoreInterface` (`:380-383`); accessor `resource_pool_store()` — `:353-365` (raises `RuntimeError` when unset, `:363`) |
| **3b** | **Loader swallows failure** | `zen_server/utils.py:384-385` catches `(ModuleNotFoundError, KeyError)` and only `logger.warning("Unable to load resource pool store source.")`. A typo'd source silently yields an unpooled server — no startup failure, later 501s |
| **4** | Entitlement gate | `check_entitlement(feature=RESOURCE_POOL_FEATURE)` at `routers/resource_pools_endpoints.py:74` (create) and `:159` (update), and `routers/resource_pool_subject_policies_endpoints.py:74` (create) and `:156` (update). Feature name `RESOURCE_POOL_FEATURE = "resource_pool"` — `constants.py:703`. Short-circuits in OSS at `feature_gate/endpoint_utils.py:30`. On Pro over quota: `SubscriptionUpgradeRequiredError` (`exceptions.py:163`) → **402** (`zen_server/exceptions.py:79`) |
| **5** | Deployment-type gate | **absent for resource pools.** `get_server_config()` (`server_config.py:851-885`) force-sets `rbac_implementation_source` (`:865`) and `feature_gate_implementation_source` (`:868`) on a Pro server but **never sets `resource_pool_implementation_source`**. So even on a Pro server the source must arrive as `ZENML_SERVER_RESOURCE_POOL_IMPLEMENTATION_SOURCE` — see §"Contradiction" below |
| **6** | Helm value | **none.** `grep -n "resourcePool" helm/values.yaml` → no hits. `helm/templates/_environment.tpl` maps only `rbacImplementationSource` (`:226-227`). Deploying pools via the OSS chart needs a raw `extraEnvVars`-style override |
| **2** | Env-var boolean | not used here. The lever is a dotted path (`ZENML_SERVER_RESOURCE_POOL_IMPLEMENTATION_SOURCE`) derived from `server_config.py:834-841`, not `handle_bool_env_var` (`constants.py:131`) |
| **8** | Schema availability | the 8 tables ship in OSS via `b0ba2c3800e3_add_resource_pools.py` and are created by `alembic upgrade head` for **every** deployment. Suppressible only with `ENV_ZENML_DISABLE_DATABASE_MIGRATION` (`sql_zen_store.py:1551-1553`), which disables all migrations. Net: **every OSS database carries 8 permanently empty resource-pool tables** |
| **1** | Dependency presence | none — no optional package |
| **9** | Deprecation | none |
| — | RBAC coupling | `ResourceType.RESOURCE_POOL = "resource_pool"` and `RESOURCE_POOL_SUBJECT_POLICY` — `rbac/models.py:74-75`; both non-project-scoped (`:95-96`); mapped to schemas at `rbac/utils.py:731,755`. In OSS these permission checks fail open (dossier 0001 §2) |

### Duplicate initialization

`initialize_resource_pool_store()` is called **twice** during startup — `zen_server_api.py:182` and again `:187`. Idempotent (it reassigns a module global), but it means a Pro implementation's `__init__` — which mutates `store.resource_pools` (`store_interface.py:228`) — runs twice per boot.

---

## 4. OSS degradation

**Lost entirely:**

- Capacity accounting. `ResourcePoolResponse.capacity: Dict[str, PositiveInt]` (`models/v2/core/resource_pool.py:92`), `occupied_resources` (`:137`), `queue_length` (`:131`).
- Queueing and admission control. `ResourceRequestStatus` (`enums.py:669-679`) with `PENDING/ALLOCATED/REJECTED`, and `ExecutionStatus.QUEUED` assignment at `sql_zen_store.py:12459-12461`.
- Preemption. `preemptible` flag on the request (`sql_zen_store.py:12456`) and `PREEMPTING`/`PREEMPTED` states (`enums.py:673-674`).
- Per-component quotas. `ResourcePoolSubjectPolicy`, keyed on `component_id` (`models/v2/core/resource_pool_subject_policy.py:47`) — i.e. per-orchestrator / per-step-operator limits.
- Resource release on step completion (`sql_zen_store.py:12898-12901`).

**What substitutes:** nothing inside ZenML. Two orchestration hooks simply do not fire:

| Hook | Guard | Effect when off |
|---|---|---|
| `sql_zen_store.py:12435-12443` | `self.resource_pools_enabled and step_schema.status in {INITIALIZING, PROVISIONING, RUNNING} and step_run.resource_requester` | no `ResourceRequestRequest` created → step never queues, runs immediately |
| `sql_zen_store.py:12895-12901` | `(execution_status.is_finished or == RETRYING) and self.resource_pools_enabled` | no release call |

`resource_requester` is still populated in OSS — `orchestrators/step_run_utils.py:301-303` sets it to the step operator's or orchestrator's component id, and it is a real field on `StepRunRequest` (`models/v2/core/step_run.py:180`). So OSS writes the requester column and then ignores it. Concurrency limiting falls entirely to the orchestrator backend (Kubernetes quotas, SageMaker limits, etc.).

**What the user actually sees:** `zenml resource-pool list` is registered (`cli/__init__.py:2581`) and reaches `client.list_resource_pools` (`client.py:2341`) → `RestZenStore.list_resource_pools` (`rest_zen_store.py:1378`) → **HTTP 501**. A first-class CLI command that cannot succeed on any OSS server.

---

## 5. Self-host path

Tier A. There is **no env var that turns this on with in-tree code** — `ZENML_SERVER_RESOURCE_POOL_IMPLEMENTATION_SOURCE` demands a class that does not exist in this repo.

Writing one means implementing **15 abstract methods**:

| Group | Methods | Interface lines |
|---|---|---|
| Pools | create / get / list / update / delete | `store_interface.py:45,58,73,89,103` |
| Subject policies | create / get / list / update / delete | `:111,124,138,154,168` |
| Requests | get / list / delete | `:178,193,209` |
| SQL-only | `release_step_run_resources(session, step_run_id)`, `create_resource_request(session, request) -> Response \| None` | `:231,242` |

The last two take a live SQLAlchemy `Session` (`:232`, `:243`) — they run **inside** the step-run transaction at `sql_zen_store.py:12448` and `:12899`, so an implementation must be transaction-correct against ZenML's own session, not merely a separate service.

Size of the job: the schemas, models, migration, routers, CLI and REST plumbing are all done for you (~3,000 lines, §1). What is missing is the **scheduler**: a multi-key capacity allocator with per-subject policy limits, a reserved non-preemptible share, a priority queue, preemption victim selection, orphan reconciliation, and capacity-change requeueing. The scope is legible from the skipped test file (§7) — 31 test functions naming exactly those behaviours. Realistically a substantial engineering project (thousands of lines, heavy concurrency risk), not a weekend port.

Sharpest hint at what's expected: `create_resource_request` returns `ResourceRequestResponse | None` and the docstring says "or None if the feature is not enabled" (`store_interface.py:252`) — the Pro implementation itself has an internal off-switch beyond the source-loading one.

---

## 6. Blast radius

**Direct callees of the hole** — 13 `SqlZenStore` methods (`sql_zen_store.py:4086-4272`), each a one-line delegation to `self.resource_pools.<x>`, each raising 501 in OSS.

**Callers:**

| From | To | Line |
|---|---|---|
| 3 routers (13 endpoints) | `zen_store().<method>` | `resource_pools_endpoints.py:78,108,135,164-165,185-186`; `resource_pool_subject_policies_endpoints.py:78,106,132,161-162,182-183`; `resource_requests_endpoints.py:64,90,109` |
| `RestZenStore` | HTTP | `rest_zen_store.py:1341-1538+` |
| `Client` | store | `client.py:2292-2560` |
| CLI | `Client` | `cli/resource_pool.py`, `cli/resource_request.py` |

**Cross-module hops that matter:**

1. **Into the step-run write path.** `sql_zen_store.py:12435-12464` sits inside step-run creation and can flip `step_schema.status` to `QUEUED` (`:12459`). This is the only place resource pools change *pipeline execution semantics*, and it reads `step_config.config.resource_settings.merged_requested_resources()` (`:12446`) plus `.preemptible` (`:12456`) — both live on the OSS `ResourceSettings`, so the settings surface is shared.
2. **Into step-run update.** `sql_zen_store.py:12895-12901`, release on finish/retry.
3. **Into `orchestrators/step_run_utils.py:301-303`**, which populates `resource_requester` from `self.stack.step_operator.id` or `self.stack.orchestrator.id` — unconditional, OSS included.
4. **Into RBAC.** Two `ResourceType` members (`rbac/models.py:74-75`) and two schema-map entries (`rbac/utils.py:731,755`). Removing resource pools would break the mapping's exhaustiveness.
5. **Into the ZenStore contract.** `zen_store_interface.py:183` makes the 13 methods part of `ZenStoreInterface` — every store implementation must at least declare them.

**No RBAC on resource requests.** `resource_requests_endpoints.py` imports none of the `verify_permissions_and_*` wrappers and calls `zen_store()` directly (`:64`, `:90`, `:109`) behind only `Security(authorize)` (`:51`). The pools and policies routers *do* use the wrappers. So resource-request read and delete are authenticated but never authorized, on Pro as well as OSS.

---

## 7. Test coverage

There is **no** `tests/unit/zen_stores/resource_pools/` directory. Coverage lives in two files:

| File | Status |
|---|---|
| `tests/unit/zen_stores/test_resource_request_pool_lifecycle.py` | **2,020 lines, 31 test functions — entirely skipped.** `tests/unit/zen_stores/test_resource_request_pool_lifecycle.py:30`: `pytest.skip("Resource pool lifecycle tests are disabled.", allow_module_level=True)` |
| `tests/integration/integrations/kubernetes/deployers/test_kubernetes_deployer.py` | incidental mention only |

`grep -rn "resource_pool" tests --include="*.py"` → 19 hits across those 2 files.

**This is the single most informative artefact in the audit.** The skipped suite names the exact allocator semantics the missing implementation must provide — `test_request_rejected_if_exceeds_pool_total_capacity` (`:183`), `test_non_preemptible_request_rejected_if_exceeds_reserved_share` (`:261`), `test_preemption_skips_non_preemptible_victims` (`:731`), `test_orphaned_requests_are_cancelled_before_allocation` (`:896`), `test_decreasing_pool_capacity_keeps_allocated_and_rebuilds_queue` (`:1101`), `test_reconciliation_repairs_occupied_resource_counters` (`:1877`), and 25 more. It imports `SqlZenStore` (`:28`) and the concrete schemas (`:23-27`), i.e. it was written to run against an implementation bound to `store.resource_pools` — which OSS does not have. The suite is a specification for Tier A work, checked into OSS and disabled.
