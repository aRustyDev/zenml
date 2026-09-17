# 0001 — The ZenML gate taxonomy

Measured against `cc064f100`. Every claim below carries a `file:line` and was verified by
running the check, not by reading around it.

## 1. There is one gating idiom, repeated six times

`src/zenml/config/server_config.py` declares six pluggable subsystems in the block at
`:343-349` (which also contains `reportable_resources` at `:345` — a list, not a source). Each
source is an `Optional[str]` dotted path; each has a derived `*_enabled` property that means
nothing more than *"is that string set?"*. All six default to `None`.

| Source field (`:343-349`) | `*_enabled` property | Initializer | Fallback when unset? |
|---|---|---|---|
| `rbac_implementation_source` | `:695` | `zen_server/utils.py:142` | no |
| `feature_gate_implementation_source` | `:704` | `zen_server/utils.py:170` | no |
| `workload_manager_implementation_source` | `:713` | `zen_server/utils.py:201` | **no** — bare `if source :=`, no `else` |
| `resource_pool_implementation_source` | `:722` | `zen_server/utils.py:368` | no — and never auto-wired, even on Pro |
| `stream_broker_implementation_source` | `:731` | `zen_server/utils.py:259` | **no** — `if not source: return` (`:271`) |
| `snapshot_run_dispatcher_implementation_source` | *(none)* | `zen_server/utils.py:434` | **YES** — `else: LocalSnapshotRunDispatcher()` (`:458`) |

The last row is the exception that proves the pattern, and it is easy to miss: it is the only
one of the six that ships a working default.

Because `ServerConfiguration` is populated from `ZENML_SERVER_<FIELD>` (`constants.py:274`,
loader `server_config.py:829-846`), **any of the six can be pointed at an arbitrary class by
environment variable.** That is the lever this audit exists to evaluate.

## 2. The master switch

`deployment_type == ServerDeploymentType.CLOUD` → `is_pro_server` (`server_config.py:820-826`;
enum at `models/v2/misc/server_models.py:26-39`). When true, `get_server_config()`
(`:851-885`) **force-overwrites** the config — this is not a default, it is an override:

- `auth_scheme` → `AuthScheme.EXTERNAL`
- `rbac_implementation_source` → `zenml.zen_server.rbac.zenml_cloud_rbac.ZenMLCloudRBAC` (`:865-867`)
- `feature_gate_implementation_source` → `...zenml_cloud_feature_gate.ZenMLCloudFeatureGateInterface` (`:868`)
- `reportable_resources` → `DEFAULT_REPORTABLE_RESOURCES` (`:869`)

Deploy-time exposure: `helm/values.yaml:86` `zenml.pro.enabled`, with
`zenml.featureGateImplementationSource` at `:304` and its `rbacImplementationSource` sibling.

## 3. Four tiers, verified by subclass census

The single discriminating question is: *does a concrete implementation exist in the OSS tree?*
Answered with `grep -rn "<Interface>)" src/zenml --include="*.py"`:

### Tier A — Pro-only. No *self-contained* implementation exists in OSS.

The precise claim matters. For RBAC and the feature gate, a concrete class **does ship in the OSS
wheel** — what is missing is a usable one. Both are thin HTTP clients to the Pro control plane and
cannot even be constructed without Pro credentials: `ZenMLCloudRBAC.__init__` calls
`cloud_connection()` (`zenml_cloud_rbac.py:37`), which builds `ServerProConfiguration` from the
`ZENML_SERVER_PRO_*` env block (`cloud_utils.py:34`, `server_config.py:926-951`). So the gap is
*wiring plus a remote service*, not source code. Resource pools are the stricter case: no concrete
class exists at all.

| Subsystem | Interface | Shipped subclass | Usable in OSS? |
|---|---|---|---|
| RBAC | `rbac/rbac_interface.py:25` | `ZenMLCloudRBAC` (`rbac/zenml_cloud_rbac.py:32`) | No — HTTP proxy, needs Pro plane |
| Feature gate | `feature_gate/feature_gate_interface.py:20` | `ZenMLCloudFeatureGateInterface` (`zenml_cloud_feature_gate.py:61`) | No — same |
| Resource pools | `zen_stores/resource_pools/store_interface.py:39` | **none** — `ResourcePoolsSQLStoreInterface` (`:217`) is itself abstract, 15 `@abstractmethod`s | No — nothing to wire |

Resource pools are the starkest case: full CLI (`cli/resource_pool.py`), models, two routers, two
schemas and a migration (`b0ba2c3800e3_add_resource_pools.py`) all ship — complete scaffolding
around an absent engine. Every endpoint returns **HTTP 501** (`sql_zen_store.py:1243-1247` →
`zen_server/exceptions.py:95`).

Three things make resource pools anomalous even among Tier A, all verified:

- **`resource_pool_implementation_source` is never assigned anywhere** — not by the Pro override
  block, not by Helm. It appears only as its declaration (`server_config.py:347`), its property
  (`:728`) and its loader (`zen_server/utils.py:375`). Unlike RBAC and the feature gate, *even a Pro
  server* must supply it as an environment variable. `grep -rin resourcepool helm/` returns nothing.
- **Its loader swallows failure** (`zen_server/utils.py:384-385`): a mistyped source logs a warning
  and continues, so the symptom surfaces later as silent 501s rather than a startup error.
- **`initialize_resource_pool_store()` is called twice** at startup — `zen_server_api.py:182` and
  `:187`. Harmless today because the loader is idempotent, but it is a genuine duplicate.

### Tier B — OSS implementation exists in-tree but is **not wired**. Opt-in by env var.

Tier B splits in two, and the distinction decides whether the env var is actually worth setting.
**"A class exists" is not the same as "the capability exists"** — each candidate was read, not
just located.

**B1 — genuine implementation.** Stream broker. `RedisStreamsBroker`
(`streaming/brokers/redis_streams.py:84`) is 286 lines with a real `publish`/`read`/`latest_id`/
`delete_stream`/`close` surface and **zero stub returns**; `InMemoryBroker`
(`streaming/brokers/memory.py:77`) covers the single-process case. Neither is wired —
`initialize_streaming` returns early at `utils.py:271`. Setting
`stream_broker_implementation_source` (plus the `server-streaming` extra and a Redis) is a real
path to live event streaming.

**B2 — degenerate dev shim.** Workload manager. `InMemoryWorkloadManager`
(`zen_server/pipeline_execution/in_memory_workload_manager.py:30`) is referenced **nowhere else in
the repo**, and reading it shows it is not a production implementation:

| Method | Reality |
|---|---|
| `run()` (`:33`) | Works, but **ignores the `image` argument entirely** — rewrites `command[0]` to `sys.executable` (`:57`) and shells out with `subprocess` on the server itself. No containerization. |
| `build_and_push_image()` (`:74`) | **Returns `""`** (`:95`). Cannot build an image. |
| `get_logs()` (`:105`) | **Returns `""`** (`:113`). No log retrieval. |
| `delete_workload()` (`:97`), `log()` (`:115`) | `pass` |

So enabling it flips `workload_manager_enabled` and unblocks the gated endpoints, but yields
server-side execution that cannot containerize and returns no logs. Treat B2 as a testing shim,
**not** as OSS parity with the Pro workload manager. For the practical consequence — that
server-side snapshot and template execution is effectively Tier A on OSS — see dossiers 0004 and
0006.

### Tier D — OSS default-on, with an override hook.

| Subsystem | Default | Override |
|---|---|---|
| Snapshot run dispatcher | `LocalSnapshotRunDispatcher` (`snapshot_run_dispatcher.py:57`), instantiated at `utils.py:458` | `snapshot_run_dispatcher_implementation_source` |

### Tier C — the behavioural consequence: gates that fail **open**

Tier A subsystems being absent does not close the door; it removes the doorman. This inverts the
naive "Pro unlocks features" expectation.

- `feature_gate/endpoint_utils.py` short-circuits all three entry points on
  `if not server_config().feature_gate_enabled: return` — `check_entitlement` (`:30`),
  `report_usage` (`:42`), `report_decrement` (`:54`). **OSS is unmetered.**
- `rbac/utils.py` early-returns at `:75, 110, 243, 272, 316, 381, 812, 839, 860`. **Every
  permission check passes.**
- Authorization degrades to the binary `UserSchema.is_admin` column, funnelled through
  `verify_admin_status_if_no_rbac()` (`zen_server/utils.py:793`, falls open at `:810`).

## 3.5 Two things the tier model does not capture

### Helm exposes only half the sources

Mechanism 6 is narrower than "deploy-time exposure of mechanism 3". Only **three** of the six
sources have a Helm key (`grep -rn ImplementationSource helm/`):

| Source | Helm key | Template |
|---|---|---|
| `rbac_implementation_source` | `zenml.auth.rbacImplementationSource` (`values.yaml:297`) | `_environment.tpl:226-227` |
| `feature_gate_implementation_source` | `zenml.auth.featureGateImplementationSource` (`values.yaml:304`) | `_environment.tpl:229-230` |
| `stream_broker_implementation_source` | `zenml.streaming.streamBrokerImplementationSource` (`values.yaml:66`) | `_environment.tpl:251-252` |

`workload_manager`, `resource_pool` and `snapshot_run_dispatcher` have **no Helm key at all** — raw
env var only. That the stream broker *does* have one, with `RedisStreamsBroker` named in the
commented example at `values.yaml:66`, is corroborating evidence that Redis streaming is the
intended, supported OSS path (Tier B1) rather than an accident of packaging.

### The six loaders fail in two opposite directions

This decides whether a misconfiguration is loud or silent, and the split is not documented anywhere:

| Loader | On load failure | Consequence of a typo |
|---|---|---|
| `initialize_workload_manager` (`utils.py:201`) | `logger.warning`, continues (`:215-217`) | **Silent.** You stay in OSS mode believing the feature is on. |
| `initialize_resource_pool_store` (`utils.py:368`) | `logger.warning`, continues (`:384-385`) | **Silent.** Surfaces later as HTTP 501s. |
| `initialize_streaming` (`utils.py:259`) | raises `RuntimeError` (`:283+`) | Loud — server fails to start. |
| `initialize_snapshot_run_dispatcher` (`utils.py:434`) | raises `RuntimeError` (`:459-465`) | Loud. |

When enabling a Tier B capability, **verify the `*_enabled` property actually flipped** rather than
trusting a clean startup — for two of the six, a clean startup proves nothing.

## 4. Entitlement metering — narrow and fully enumerated

`DEFAULT_REPORTABLE_RESOURCES = ["project", "pipeline", "pipeline_run", "model"]`
(`constants.py:465`). Three feature names exist:
`RUN_TEMPLATE_TRIGGERS_FEATURE_NAME = "template_run"` (`:466`), `SCHEDULE_FEATURE = "schedule"`
(`:702`), `RESOURCE_POOL_FEATURE = "resource_pool"` (`:703`).

Complete gated call set (obtained via `gitnexus context()` on `check_entitlement`, then confirmed
by grep): `trigger_endpoints.py:141,256,337` · `resource_pools_endpoints.py:74,159` ·
`resource_pool_subject_policies_endpoints.py:74,156` · `runs_endpoints.py:757` ·
`run_templates_endpoints.py:290` · `pipeline_snapshot_endpoints.py:405`. Decrement reporting at
`models_endpoints.py:203`, `pipelines_endpoints.py:231`, `projects_endpoints.py:233`.

Violation path: `SubscriptionUpgradeRequiredError` (`exceptions.py:163`) → **HTTP 402**
(`zen_server/exceptions.py:79`).

## 5. The nine gating mechanisms

Use these labels verbatim in every dossier's gate inventory.

| # | Mechanism | Anchor |
|---|---|---|
| 1 | Dependency presence | `Integration.check_installation()` (`integrations/integration.py:59-94`) |
| 2 | Env-var boolean | `handle_bool_env_var` (`constants.py:131`), ~34 read sites |
| 3 | Pluggable implementation source | the six above |
| 4 | Entitlement gate | `check_entitlement` → HTTP 402 |
| 5 | Deployment-type gate | `ServerDeploymentType.CLOUD` ⇒ forced Pro config |
| 6 | Helm value | deploy-time exposure of 2/3/5 — **only 3 of the 6 sources**, see §3.5 |
| 7 | Three-state `Optional[bool]` cascade | pipeline → step `enable_*` |
| 8 | Schema availability | alembic revision + `DISABLE_DATABASE_MIGRATION` (`sql_zen_store.py:1552`) |
| 9 | Deprecation | `utils/deprecation_utils.py:29`; **no experimental/beta decorator exists** |

## 6. Vocabulary traps

Handle these before analysing any area; each one silently halves a grep.

- `ResourceType.PIPELINE_SNAPSHOT = "pipeline_deployment"` (`rbac/models.py:62`) — symbol and wire
  value disagree.
- **"deployment" means three things**: the `deployers/` pipeline-serving component; the legacy wire
  name for a snapshot; and `ServerDeploymentType`. Three routers coexist —
  `deployment_endpoints.py`, `pipeline_deployments_endpoints.py`, `pipeline_snapshot_endpoints.py`.
- **"model" means three things** — Pydantic model, ML model, ZenML Model namespace (`CLAUDE.md`) —
  plus a fourth trap: `StackComponentType.DEPLOYER` is pipeline serving, not model serving.
- **"workspace" means two things**: the pre-rename name for Project (migration
  `12eff0206201_rename_workspace_to_project.py`) and a ZenML Pro workspace (a deployed server).
  `projects_endpoints.py` still serves a fully `deprecated=True` `/workspaces` router.
- `ResourceType.USER` is **commented out** (`rbac/models.py:79`) — users are not an RBAC resource.
- **Run Templates are deprecated**, superseded by Pipeline Snapshots
  (`run_templates_endpoints.py:72,85,122`).
- The EventSource/Action plugin system was **removed**, not evolved
  (`97109a4a8d26_remove_triggers.py`); one polymorphic `Trigger` with three flavors replaced it.

## 7. Correction against the originating plan

The plan classified `snapshot_run_dispatcher` as Tier B (exists but unwired). It is not — it has an
`else` fallback at `utils.py:458` and is **on by default**. Tier D was added to hold it. The other
five rows of the plan's table stand as written.

## 8. Two branches that run the other way

The audit's framing is "what does OSS lose". Two verified branches invert it:

- **Pro loses the implicit default project.** `SqlZenStore._default_project_enabled`
  (`sql_zen_store.py:14114-14129`) returns `False` when `is_pro_server` — the only `is_pro_server`
  branch outside `server_config.py`.
- **OSS multi-project is limited by the dashboard, not the server** (`cli/utils.py:2760-2784`) — the
  only front-end gate found anywhere in this audit.

Also note: `RBACSqlZenStore` is selected whenever `ENV_ZENML_SERVER` is set
(`zen_stores/base_zen_store.py:156-162`) with **no edition test**, so the "Pro" store class runs on
every OSS server; only its leaf permission checks no-op. And `FlavorRegistry.register_integration_flavors`
(`stack/flavor_registry.py:145-152`) calls `.flavors()` for **every** integration with no
`check_installation()` gate — unlike `activate_integrations` (`registry.py:118`) — so every server
writes all integration flavor rows regardless of installed extras.
