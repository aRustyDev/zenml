# 0012 — Model Management

Repo @ `cc064f100`. Source root `src/zenml/`. Framework reference: the six nullable dotted-path
gates on `ServerConfiguration` (`src/zenml/config/server_config.py:343-349`);
`DEFAULT_REPORTABLE_RESOURCES = ["project","pipeline","pipeline_run","model"]` (`constants.py:465`).

> ## Vocabulary — FOUR distinct things, not one
>
> | # | Concept | `StackComponentType` | Root |
> |---|---|---|---|
> | **(a)** | **ZenML Model / Model Control Plane** — a *namespace* grouping artifacts, metadata and runs | none (a server entity, not a stack component) | `model/`, `models/v2/core/model*.py`, `zen_stores/schemas/model_schemas.py`, `routers/models_endpoints.py` |
> | **(b)** | **Model Registry** — a stack component wrapping an external registry | `MODEL_REGISTRY = "model_registry"` (`enums.py:229`) | `model_registries/`, `integrations/mlflow/model_registries/` |
> | **(c)** | **Model Deployer / served models** — a stack component that serves an ML model | `MODEL_DEPLOYER = "model_deployer"` (`enums.py:226`) | `model_deployers/`, `services/`, `routers/service_endpoints.py` |
> | **(d)** | **Deployer** — pipeline-as-a-service; **NOT** model serving | `DEPLOYER = "deployer"` (`enums.py:230`) | `deployers/` (`BaseDeployer:89`, `BaseDeployerFlavor:1104`, `type` → `:1114`) |
>
> (d) is the newest and the easiest to confuse with (c). `BaseDeployer` is documented as
> "a ZenML deployment registry … keep track of all externally running **pipeline**"
> (`deployers/base_deployer.py:102-104`) and it resolves its component via
> `snapshot.stack.components[StackComponentType.DEPLOYER]` (`:241-243`). It has nothing to do
> with `model_deployers/`. This dossier covers (a), (b) and (c); (d) is named only to separate it.

---

# Part A — ZenML Model / Model Control Plane

## A1. Surface

| Layer | Path |
|---|---|
| Client SDK | `src/zenml/model/model.py` — `Model:47`, `id:100`, `model_id:125`, `number:138`, `stage:163`, `load_artifact:183`, `get_artifact:210`, `get_model_artifact:234`, `get_data_artifact:258`, `get_deployment_artifact:282`, `set_stage:306`, `log_metadata:318`, `run_metadata:343`, `delete_artifact:371`, `_get_or_create_model:507`, `_get_model_version:562`, `_get_or_create_model_version:625` |
| Helpers | `src/zenml/model/utils.py` — `log_model_metadata:36`, `link_artifact_version_to_model_version:84`, `link_artifact_to_model:103`, `link_service_to_model:139` |
| Lazy loading | `src/zenml/model/lazy_load.py` — `ModelVersionDataLazyLoader:27`, `_root_validator:52`, `_get_model_response:82` |
| Domain models | `models/v2/core/model.py` (`ModelRequest:62`, `ModelUpdate:117`, `ModelResponseBody:136`, `ModelResponseMetadata:140`, `ModelResponseResources:184`, `ModelResponse:198`), `model_version.py`, `model_version_artifact.py`, `model_version_pipeline_run.py` |
| SQL schemas | `zen_stores/schemas/model_schemas.py` — `ModelSchema:80`, `ModelVersionSchema:321`, `ModelVersionArtifactSchema:605`, `ModelVersionPipelineRunSchema:686` |
| REST | `routers/models_endpoints.py` (`router:56`, `create_model:76`, `list_models:105`, `get_model:134`, `update_model:159`, `delete_model:186`), `routers/model_versions_endpoints.py` (`router:73`, `model_version_artifacts_router:262`, `model_version_pipeline_runs_router:384`) |
| CLI | `src/zenml/cli/model.py` (780 lines) — `model:78`, `register_model:181`, `update_model:311`, `delete_model:371`, `version:400`, `update_model_version:475`, artifact-link listings `:643,677,711,745` |
| Enums/constants | `ModelStages` (`enums.py:455-462`: `NONE`, `STAGING`, `PRODUCTION`, `ARCHIVED`, `LATEST`); `LATEST_MODEL_VERSION_PLACEHOLDER = "__latest__"` (`constants.py:616`) |
| RBAC | `ResourceType.MODEL = "model"` (`rbac/models.py:57`), `MODEL_VERSION` (`:58`); `Action.PROMOTE = "promote"` (`rbac/models.py:42`, comment "# Models" at `:41`) |
| Migrations | `4d688d8f7aff_rename_model_version_to_model.py`, `14d687c8fa1c_rename_model_config_to_model_version.py`, `4f66af55fbb9_rename_model_config_model_to_model_.py`, `3b68abe58f44_add_model_watchtower_entities.py`, `ec6307720f92_simplify_model_version_links.py`, `a1237ba94fd8_add_model_version_producer_run_unique_.py`, `5cc3f41cf048_add_save_models_to_registry.py`, `bf2120261b5a_add_configured_model_version_id.py`, `904464ea4041_add_pipeline_model_run_unique_constraints.py`, `6f707b385dc1_fix_model_artifacts.py` |

## A2. Tier verdict — **Tier C**

**Tier C.** The Model Control Plane is fully implemented in OSS — `ModelSchema` (`model_schemas.py:80`),
`ModelVersionSchema` (`:321`), the routers, the `Model` SDK class (`model/model.py:47`), the CLI.
What is Pro-only is **metering and authorization**, and both fail open:

- `MODEL` ∈ `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`), set only for Pro
  (`server_config.py:869`). `report_decrement(ResourceType.MODEL, ...)` on delete
  (`models_endpoints.py:203`) and `check_entitlement` on create
  (`rbac/endpoint_utils.py:102`, reached via `verify_permissions_and_create_entity` at
  `models_endpoints.py:94`) both return immediately in OSS
  (`feature_gate/endpoint_utils.py:30,54`).
- `Action.PROMOTE` (`rbac/models.py:42`) is checked when a model version's `stage` changes
  (`model_versions_endpoints.py:224-226`). In OSS `verify_permission_for_model` early-returns
  (`rbac/utils.py:316`), so **anyone can promote anything to `production`**.

No Pro-only model class exists. Verified: `grep`ing `models_endpoints.py`,
`model_versions_endpoints.py`, `model_schemas.py` and `model/` for `is_pro_server`,
`*_implementation_source`, or a Pro import returns nothing.

**The implicit-creation injection is the interesting bit.** `RBACSqlZenStore`
(`zen_server/rbac/rbac_sql_zen_store.py:47`) overrides exactly three model methods to add
permission and entitlement checks to the paths where a model or model version is created
*implicitly* by a pipeline step rather than by an explicit API call:

| Override | Line | Injects |
|---|---|---|
| `_get_or_create_model` | `:96` | `verify_permission(MODEL, CREATE, project_id)` `:115-119`, `check_entitlement(MODEL)` `:120`; on denial falls back to `get_model_by_name_or_id` and re-raises the *original* permission error if the model genuinely does not exist (`:130-141`); `report_usage` on create (`:144-146`), `verify_permission_for_model(..., READ)` otherwise (`:148`) |
| `_get_model_version` | `:152` | `verify_permission_for_model(model_version, READ)` (`:174`) |
| `_get_or_create_model_version` | `:178` | same shape as `_get_or_create_model` |

**`RBACSqlZenStore` is selected on *every* server, OSS included** —
`BaseZenStore.get_store_class` returns it whenever `ENV_ZENML_SERVER` is in the environment
(`zen_stores/base_zen_store.py:156-162`), with no edition test. So the Pro code path *executes*
on OSS; only its leaf checks no-op. That is a meaningful correction to a naive reading of
"Tier A = Pro-only code path".

## A3. Gate inventory (Part A)

| # | Mechanism | Location | Effect |
|---|---|---|---|
| **4** | **Entitlement gate** | `check_entitlement` at `rbac/endpoint_utils.py:102,188` and `rbac_sql_zen_store.py:120`; `report_usage` at `rbac/endpoint_utils.py:107,195` and `rbac_sql_zen_store.py:144`; `report_decrement(ResourceType.MODEL)` at `models_endpoints.py:203`. All short-circuit on `feature_gate_enabled` (`server_config.py:704`) at `feature_gate/endpoint_utils.py:30,42,54` | Pro: models metered → `SubscriptionUpgradeRequiredError` (`exceptions.py:163`) → **HTTP 402** (`zen_server/exceptions.py:79`). OSS: unlimited |
| **3** | **Pluggable implementation source** | `rbac_implementation_source` (`server_config.py:343`) → `rbac_enabled` (`:695`); Pro sets it to `ZenMLCloudRBAC` (`server_config.py:865-867`). Concrete class at `zen_server/rbac/zenml_cloud_rbac.py:32` — `check_permissions:39`, `list_allowed_resource_ids:81`, `update_resource_membership:124`, `delete_resources:152` | Governs `Action.PROMOTE` (`model_versions_endpoints.py:226`), model READ/UPDATE/DELETE, and the three `RBACSqlZenStore` overrides |
| 3 | Pluggable implementation source | `feature_gate_implementation_source` (`server_config.py:344`) → `feature_gate_enabled` (`:704`); Pro sets `ZenMLCloudFeatureGateInterface` (`server_config.py:868`), concrete at `zen_server/feature_gate/zenml_cloud_feature_gate.py:61` (`check_entitlement:68` raising at `:82`, `report_event:94`) | As above |
| 5 | Deployment-type gate | `is_pro_server` (`server_config.py:820`) drives the whole `get_server_config` Pro block (`:851-885`) that installs both sources above | Indirect only — no model-specific `is_pro_server` branch exists |
| 8 | Schema availability | 10 model migrations (§A1) incl. **three renames**; skippable via `ZENML_DISABLE_DATABASE_MIGRATION` (`sql_zen_store.py:1551-1557`) | `4d688d8f7aff_rename_model_version_to_model.py` / `14d687c8fa1c_rename_model_config_to_model_version.py` / `4f66af55fbb9_...` are a churn cluster — a half-applied chain leaves the router pointing at absent tables |
| 9 | Deprecation | The legacy `workspace_router` (imported from `projects_endpoints.py`) is mounted at `models_endpoints.py:69` and `model_versions_endpoints.py:86`, both flagged `deprecated=True` (`models_endpoints.py:72`, `model_versions_endpoints.py:89`) | Legacy `/workspaces/{project}/models[...]` paths still answer and are marked deprecated. No `@experimental` decorator exists in the repo |
| 2 | Env-var boolean | none model-specific | — |
| 1 | Dependency presence | none for (a) — it is core, not an integration | — |
| 6 | Helm value | none | — |
| 7 | Three-state cascade | none for (a) | — |

## A4. Two dead symbols

1. **`LATEST_MODEL_VERSION_PLACEHOLDER = "__latest__"` (`constants.py:616`) has zero consumers.**
   `grep -rn "LATEST_MODEL_VERSION_PLACEHOLDER\|__latest__" src/ tests/` returns **only the
   definition line**. "Latest" is instead expressed as `ModelStages.LATEST` (`enums.py:462`),
   checked at `model/model.py:583`.
2. **`save_models_to_registry` is stored but never acted on.** Declared at
   `models/v2/core/model.py:108,130,178,312`, `model/model.py:79`, persisted at
   `model_schemas.py:119,227,257`, plumbed through `client.py:8150,8182,8215,8255` and
   `cli/model.py:49,191`, with its own migration (`5cc3f41cf048_add_save_models_to_registry.py`).
   But `grep -rn "save_models_to_registry" src/zenml/{orchestrators,steps,artifacts,model_registries}/`
   returns **nothing** — no runtime path reads it. It is the only declared bridge between
   concept (a) and concept (b), and it is inert.

## A5. OSS degradation (Part A)

| Capability | OSS behaviour | Substitute |
|---|---|---|
| Stage promotion control (`Action.PROMOTE`) | `verify_permission_for_model` fails open (`rbac/utils.py:316`) at `model_versions_endpoints.py:226` | None — `models_endpoints.py` never calls `verify_admin_status_if_no_rbac` (`zen_server/utils.py:793`) |
| Model/model-version read isolation | `dehydrate_response_model` (`rbac/utils.py:110`) and `batch_verify_permissions_for_models` (`:243`) fail open | Project scoping only — `MODEL` and `MODEL_VERSION` **are** project-scoped (`rbac/models.py:57-58`, not in the exclusion list `:89-102`) |
| Implicit model creation in steps | `RBACSqlZenStore._get_or_create_model` runs but its checks no-op (`rbac_sql_zen_store.py:115-120`) | — |
| Model quotas | Unmetered (`feature_gate/endpoint_utils.py:30`) | — (a strict OSS gain) |
| Model sharing | `update_resource_membership` returns at `rbac/utils.py:812` | Everything is visible project-wide |

---

# Part B — Model Registry (stack component)

## B1. Surface

| Layer | Path |
|---|---|
| Abstraction | `src/zenml/model_registries/base_model_registry.py` — `RegisteredModel`/`ModelVersion` value types (`custom_attributes:74`, `model_dump:86`), `BaseModelRegistryConfig:171`, `BaseModelRegistry:175`, `BaseModelRegistryFlavor:486` (`type` → `MODEL_REGISTRY` at `:490`) |
| Abstract surface | **14 `@abstractmethod`s**: `register_model:192`, `delete_model:214`, `update_model:229`, `get_model:253`, `list_models:268`, `register_model_version:288`, `delete_model_version:316`, `update_model_version:333`, `list_model_versions:361`, `get_model_version:418`, `load_model_version:436`, `get_model_uri_artifact_store:458`, plus `config_class:499` and `implementation_class:509` on the flavor. Only `get_latest_model_version:391` is concrete |
| Implementations | `src/zenml/integrations/mlflow/model_registries/mlflow_model_registry.py` — **the only one** |
| Flavor | `integrations/mlflow/flavors/mlflow_model_registry_flavor.py` |
| CLI | `src/zenml/cli/model_registry.py` (659 lines) — `register_model_registry_subcommands:33`, attaches to the `model-registry` group at `:39`, resolves the active stack component at `:54-63` |
| REST | **none.** There is no `model_registries_endpoints.py`; model registries are reached only through the stack-component API |

## B2. Tier verdict — **Tier B (dependency-gated), and the thinnest surface in the audit**

Not Pro-gated at all: no `check_entitlement`, no `*_implementation_source`, no `is_pro_server`
reference anywhere under `model_registries/`. The only gate is **mechanism 1**: MLflow must be
installed (`integrations/mlflow/__init__.py`, checked by `Integration.check_installation`,
`integrations/integration.py:60`).

**One concrete implementation for a 14-method abstract interface** — `MLFlowModelRegistry` is it
(confirmed in dossier 0009's flavor census: `MODEL_REGISTRY` count = 1). By comparison
`ORCHESTRATOR` has 17 and `STEP_OPERATOR` 11. Anyone self-hosting against a non-MLflow registry
writes all 14 methods themselves.

## B3. Gate inventory (Part B)

| # | Mechanism | Location | Effect |
|---|---|---|---|
| 1 | Dependency presence | `MLFlowIntegration.check_installation` via `integrations/integration.py:60-94`; flavor row still written regardless by `flavor_registry.py:145-152` (see dossier 0009 §2) | Flavor is *listed* even when MLflow is absent; instantiation then fails |
| 3 | Pluggable implementation source | `BaseModelRegistryFlavor.implementation_class` (`base_model_registry.py:509`) — the standard flavor seam | Third-party registries are first-class |
| 4/5/6/7/9 | — | **all absent** | Full parity |
| 8 | Schema availability | `5cc3f41cf048_add_save_models_to_registry.py` (the inert flag, §A4) | — |

---

# Part C — Model Deployer / served models

## C1. Surface

| Layer | Path |
|---|---|
| Abstraction | `src/zenml/model_deployers/base_model_deployer.py` — `BaseModelDeployerConfig:45`, `BaseModelDeployer:49`, **5 `@abstractmethod`s** at `:235,268,366,429,482`, `BaseModelDeployerFlavor:574` (`type` → `MODEL_DEPLOYER` at `:584`, `implementation_class:596`) |
| Service runtime | `src/zenml/services/` — `service.py`, `service_endpoint.py`, `service_monitor.py`, `service_status.py`; `local/` (`local_service.py`, `local_service_endpoint.py`, `local_daemon_entrypoint.py:82` calls `activate_integrations()`); `container/` (`container_service.py`, `container_service_endpoint.py`, `entrypoint.py:62` likewise) |
| Materializer | `src/zenml/materializers/service_materializer.py` |
| Domain model / schema | `models/v2/core/service.py`; `ResourceType.SERVICE = "service"` (`rbac/models.py:67`) — project-scoped |
| REST | `routers/service_endpoints.py` — `router:46`, `create_service:66`, `list_services:95`, `get_service:124`, `update_service:151`, `delete_service:178` |
| CLI | `src/zenml/cli/served_model.py` (429 lines) — `register_model_deployer_subcommands:37`, attaches to the `model-deployer` group at `:39`, `list_models:125` |
| Implementations | 6 flavors: BentoML, Databricks, HuggingFace, MLFlow, Seldon, VLLM (dossier 0009) |
| Link to (a) | `model/utils.py:139` `link_service_to_model` |

## C2. Tier verdict — **Tier C** (same overlay as Part A), **Tier B** for the component itself

The `services` REST surface is RBAC-gated at `service_endpoints.py:84,111,139,165,187`; in OSS
all fail open (`rbac/utils.py:75,110,243,272,316,381`). `SERVICE` is **not** in
`DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`), so services are unmetered on both editions.
The stack component itself is purely dependency-gated (mechanism 1), like Part B.

## C3. Gate inventory (Part C)

| # | Mechanism | Location | Effect |
|---|---|---|---|
| 3 | Pluggable implementation source | `rbac_implementation_source` (`server_config.py:343`) gates the five router checks | OSS: no authorization on served-model records |
| 1 | Dependency presence | 6 model-deployer integrations, each behind `check_installation` (`integrations/integration.py:60`) | Missing extra → flavor listed but unusable |
| 3 | Pluggable implementation source | `BaseModelDeployerFlavor.implementation_class` (`base_model_deployer.py:596`) | Extension seam |
| 4 | Entitlement gate | **absent** — `SERVICE ∉ DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`) | Unmetered on both |
| 9 | Deprecation | `service_endpoints.py:59` mounts the legacy `workspace_router`, flagged `deprecated=True` at `:62` | Same as Part A |
| 5/6/7/8 | — | absent | — |

---

## 5. Self-host path

| Part | Path |
|---|---|
| A — enforcement | Tier C: supply an `RBACInterface` (`zen_server/rbac/rbac_interface.py`) and point `ZENML_SERVER_RBAC_IMPLEMENTATION_SOURCE` (prefix `constants.py:274`) at it. The four methods to write are those `ZenMLCloudRBAC` provides (`zenml_cloud_rbac.py:39,81,124,152`). That alone activates `Action.PROMOTE` enforcement (`model_versions_endpoints.py:226`) and the three `RBACSqlZenStore` overrides (`rbac_sql_zen_store.py:96,152,178`) |
| A — metering | Supply a `FeatureGateInterface` (`zen_server/feature_gate/feature_gate_interface.py`) at `ZENML_SERVER_FEATURE_GATE_IMPLEMENTATION_SOURCE`, and set `ZENML_SERVER_REPORTABLE_RESOURCES` (parsed by the validator at `server_config.py:634-636`). Only then do `check_entitlement`/`report_usage`/`report_decrement` do anything |
| B | `zenml integration install mlflow`, then `zenml model-registry register … --flavor=mlflow`. For any other backend: implement 14 abstract methods on `BaseModelRegistry` (`base_model_registry.py:192-458`) plus a flavor |
| C | `zenml integration install <bentoml\|seldon\|mlflow\|huggingface\|databricks\|vllm>`; or implement 5 abstract methods on `BaseModelDeployer` (`base_model_deployer.py:235,268,366,429,482`) |

## 6. Blast radius

| Direction | Target |
|---|---|
| `RBACSqlZenStore` selection | `base_zen_store.py:156-162` — returns `RBACSqlZenStore` for **any** server (`ENV_ZENML_SERVER` set), OSS included. Any change to the three model overrides (`rbac_sql_zen_store.py:96,152,178`) affects OSS servers' code path even though the checks no-op |
| Implicit model creation | `Model._get_or_create_model` (`model/model.py:507`) and `_get_or_create_model_version` (`:625`) are called from every step that declares a `model=` — so a permission error surfaces at step-run time, not at API-call time. The store override deliberately re-raises the *permission* error rather than the `KeyError` (`rbac_sql_zen_store.py:130-141`) to keep that diagnosable |
| Stage promotion | `Model.set_stage` (`model/model.py:306,316`) → `ModelVersionUpdate.stage` → `update_model_version` (`model_versions_endpoints.py:222-226`) → `Action.PROMOTE`. Three layers, one RBAC check |
| Artifact/run links | `model_version_artifacts_router:262` and `model_version_pipeline_runs_router:384` (`model_versions_endpoints.py`) both verify against the **parent model version** (`:348,373`), not the link row — deleting links requires `UPDATE` on the version |
| Project cascade | `ProjectSchema.models` (`project_schemas.py:120`) and `.model_versions` (`:124`) — deleting a project destroys both (dossier 0010) |
| Concept bridge (a)↔(b) | `save_models_to_registry` is the only declared bridge and is inert (§A4) — treat (a) and (b) as **unconnected** subsystems for planning purposes |
| Concept bridge (a)↔(c) | `link_service_to_model` (`model/utils.py:139`) — the one live connection |
| (c)↔(d) collision risk | `model_deployers/` and `deployers/` are separate `StackComponentType`s (`enums.py:226` vs `:230`) with separate CLI groups (`model-deployer` at `cli/served_model.py:39`; the deployer CLI elsewhere). Naming a variable `deployer` in either subsystem is the trap |

## 7. Test coverage

| Path | Covers |
|---|---|
| `tests/unit/models/test_model_models.py` | (a) pydantic models |
| `tests/unit/model_registries/test_base_model_registry.py` | (b) — the abstract base only |
| `tests/unit/services/test_service.py` | (c) service runtime |
| `tests/unit/integrations/mlflow/` | The single registry implementation |
| `tests/unit/models/test_service_models.py` | (c) pydantic models |
| `tests/integration/functional/model/test_model_version.py` | (a) end-to-end |
| `tests/integration/functional/models/` | (a) |
| `tests/integration/functional/cli/test_model.py`, `test_model_registry.py` | CLIs for (a) and (b) |
| `tests/unit/zen_server/test_tag_resource_rbac.py` | The `RBACSqlZenStore` tag overrides (`:50,75`) — **not** the three model overrides |

**Gaps:**
- `RBACSqlZenStore._get_or_create_model` / `_get_model_version` / `_get_or_create_model_version`
  (`rbac_sql_zen_store.py:96,152,178`) have **no** unit test; the sibling tag overrides do
  (`tests/unit/zen_server/test_tag_resource_rbac.py`). The subtle error-substitution logic at
  `:130-141` is untested.
- `Action.PROMOTE` enforcement (`model_versions_endpoints.py:224-226`) is untested.
- No unit test covers `model/lazy_load.py` (`ModelVersionDataLazyLoader:27`).
- Nothing asserts that `save_models_to_registry` has an effect — consistent with §A4's finding
  that it has none.
