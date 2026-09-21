# 0011 — Artifact Management

Repo @ `cc064f100`. Source root `src/zenml/`. Framework reference: the six nullable dotted-path
gates on `ServerConfiguration` (`src/zenml/config/server_config.py:343-349`).

## 1. Surface

| Layer | Path |
|---|---|
| Client SDK — artifacts | `src/zenml/artifacts/` — `utils.py` (`should_use_artifact_store_cache:876`, `instantiate_artifact_store:891`, `load_artifact_store:916`), `artifact_config.py`, `external_artifact.py`, `external_artifact_config.py`, `unmaterialized_artifact.py`, `in_memory_cache.py` (`InMemoryArtifactCache:23`), `preexisting_data_materializer.py:37` |
| Materializers | `src/zenml/materializers/` — 13 modules. `base_materializer.py` (auto-register at `:60-62`, `SKIP_REGISTRATION:121`), `materializer_registry.py` (`MaterializerRegistry:26`, `register_materializer_type:34`, `get_default_materializer:103`), `built_in_materializer.py`, `cloudpickle_materializer.py:51`, `path_materializer.py:70`, `in_memory_materializer.py:35`, `dataclass_materializer.py:64`, + numpy/pandas/pydantic/uuid/structured_string/service |
| Artifact stores | `src/zenml/artifact_stores/` — `base_artifact_store.py` (path-guard decorator `__init__:64`, `_validate_path:91`, enforcement `:107-108`, wired at `:150,152`), `local_artifact_store.py` |
| Server cache | `src/zenml/zen_server/artifact_store_cache.py` — `_CacheEntry:35`, `ArtifactStoreCache:43`, `get_or_create:62`, `clear:101`, `_is_expired:121`, `_instantiate:136`, `_maybe_evict:149`, `_safe_cleanup:158` |
| Server download | `src/zenml/zen_server/download_utils.py` — `verify_artifact_is_downloadable:32`, `create_artifact_archive:74`, `download_snapshot_code_archive:122`, `verify_file_is_downloadable:142` |
| REST routers | `routers/artifact_endpoint.py` (`artifact_router:44`), `routers/artifact_version_endpoints.py` (`artifact_version_router:80`, `get_artifact_visualization:350`, `get_artifact_download_token:394`, `prune_artifact_versions:323`), `routers/curated_visualization_endpoints.py` (`router:21`) |
| SQL schemas | `zen_stores/schemas/artifact_schemas.py` (`ArtifactSchema:76`, `ArtifactVersionSchema:276`), `artifact_visualization_schemas.py` (`ArtifactVisualizationSchema:40`), `curated_visualization_schemas.py` (`CuratedVisualizationSchema:46`) |
| CLI | `src/zenml/cli/artifact.py` (439 lines) — `artifact:36`, `list_artifacts:45`, `version:118`, `prune_artifacts:352` |
| Config cascade | `config/pipeline_configurations.py:58-59`, `config/pipeline_run_configuration.py:48,53`, `config/step_configurations.py:165,170`, `config/compiler.py:196-197,231-242` |
| Migrations | `41b28cae31ce_make_artifacts_workspace_scoped.py`, `7834208cc3f6_artifact_project_scoping.py`, `6f707b385dc1_fix_model_artifacts.py` |

## 2. Tier verdict — **Tier B**, with a Tier-C authorization overlay

**Tier B.** Nothing in artifact management is Pro-gated. There is no
`*_implementation_source` field for artifacts, no `check_entitlement` call anywhere under
`artifacts/`, `materializers/`, `artifact_stores/`, or the three artifact routers, and
`ARTIFACT`/`ARTIFACT_VERSION` are **absent** from `DEFAULT_REPORTABLE_RESOURCES`
(`constants.py:465`) — so artifacts are unmetered on **both** editions.

The concrete implementations all ship: `ArtifactStoreCache` (`zen_server/artifact_store_cache.py:43`),
`LocalArtifactStore` (`artifact_stores/local_artifact_store.py`), 5 integration artifact-store
flavors (S3, GCP, Azure, B2, DigitalOcean Spaces — see dossier 0009), 13 built-in materializers
plus every integration materializer. What varies is **configuration**, via five independent
boolean/int knobs — hence Tier B.

**Tier-C overlay.** All three routers gate through RBAC — `artifact_endpoint.py:74,99,125,151,173`;
`artifact_version_endpoints.py:148,171,197,223,258,285,290,294,337,379,422`;
`curated_visualization_endpoints.py:94,95,127,159,188`. In OSS every one returns immediately
(`rbac/utils.py:75,110,243,272,316,381`). `ARTIFACT` and `ARTIFACT_VERSION` **are**
project-scoped (`rbac/models.py:53-54`, not in the exclusion list at `:89-102`), so the SQL
project filter still applies — but user-level authorization does not.

## 3. Gate inventory

| # | Mechanism | Location | Default | Effect |
|---|---|---|---|---|
| 2 | Env-var boolean — **server-side local file access** | `ENV_ZENML_SERVER_ALLOW_LOCAL_FILE_ACCESS` (`constants.py:238-240`) → `handle_bool_env_var` (`constants.py:131`) at `artifact_stores/base_artifact_store.py:75`; **only when `ENV_ZENML_SERVER` is in the env** (`:74`), else unconditionally `True` (`:79`) | `False` on the server, `True` on clients | `_validate_path` (`:91`) raises `IllegalOperationError` (`:107-108` → HTTP 403, `zen_server/exceptions.py:74`) for any non-remote path. Enforced on every `PathType` argument of every filesystem method (`:150,152`) |
| 2 | Env-var boolean — **path materializer** | `ENV_ZENML_DISABLE_PATH_MATERIALIZER` (`constants.py:271`) evaluated **at class-definition time** into `PathMaterializer.SKIP_REGISTRATION` (`materializers/path_materializer.py:70-72`) | `False` (materializer active) | Setting it removes `pathlib.Path` from the materializer registry; `base_materializer.py:60-62` is the auto-registration hook that reads it |
| 2 | Env-var boolean — **JSON encoding** | `ENV_ZENML_MATERIALIZER_ALLOW_NON_ASCII_JSON_DUMPS` (`constants.py:268-270`) → module-level constant at `built_in_materializer.py:57-59`, consumed as `ensure_ascii=not …` at `:647` | `False` (ASCII-escape) | Purely an on-disk encoding choice for built-in JSON artifacts |
| 2/3 | Server config int/bool — **artifact store cache** | `ServerConfiguration.artifact_store_cache_enabled` (`server_config.py:409`, documented `:272`); read only through `artifacts/utils.py:876-888`, which first requires `ENV_ZENML_SERVER` (`:882-883`) | `True` | When on, `instantiate_artifact_store` (`artifacts/utils.py:891`) returns a **process-shared** instance from `artifact_store_cache()`; callers must **not** call `cleanup()` (`:897-899`). When off, a fresh `StackComponent.from_model` per request |
| 2 | Server config int — **download size limit** | `ServerConfiguration.file_download_size_limit` (`server_config.py:430`) ← `DEFAULT_ZENML_SERVER_FILE_DOWNLOAD_SIZE_LIMIT` (`constants.py:406`); enforced at `download_utils.py:62-70` and `:160-167` | see note | `IllegalOperationError` → HTTP 403 |
| **7** | **Three-state `Optional[bool]` cascade** | Declared on step (`config/step_configurations.py:165,170`), pipeline (`config/pipeline_configurations.py:58-59`), and run config (`config/pipeline_run_configuration.py:48,53`); propagated by `Compiler` (`config/compiler.py:196-197`, run-level override at `:231-242`); resolved by `is_setting_enabled` (`orchestrators/utils.py:88-109`) at `orchestrators/step_runner.py:349-356` | `None` at every level → **True** (`orchestrators/utils.py:109`) | `enable_artifact_metadata` / `enable_artifact_visualization`. Precedence: step (`:105-106`) → pipeline (`:107-108`) → default `True` (`:109`) |
| 1 | Dependency presence | Artifact-store flavors are integration-gated (`integrations/{s3,gcp,azure,b2,digitalocean}`) via `Integration.check_installation` (`integrations/integration.py:60`) | — | No S3 extra → no `s3` artifact store |
| 8 | Schema availability | `41b28cae31ce_make_artifacts_workspace_scoped.py`, `7834208cc3f6_artifact_project_scoping.py`, `6f707b385dc1_fix_model_artifacts.py`; all skippable via `ZENML_DISABLE_DATABASE_MIGRATION` (`sql_zen_store.py:1551-1557`) | — | Project-scoping columns absent → `ProjectScopedFilter.apply_filter`'s `getattr(table, "project_id")` (`models/v2/base/scoped.py:451`) fails |
| 4 | Entitlement gate | **ABSENT.** No `check_entitlement` import in `artifact_endpoint.py`, `artifact_version_endpoints.py`, `curated_visualization_endpoints.py`; `ARTIFACT`/`ARTIFACT_VERSION` not in `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`) | — | Unmetered on both editions |
| 5 | Deployment-type gate | **ABSENT** | — | — |
| 6 | Helm value | **ABSENT** for artifacts — unlike `zenml.enableImplicitAuthMethods` (`helm/values.yaml:328`), none of the five knobs above has a chart field; they must be set as raw env vars | — | Operational gap |
| 9 | Deprecation | No `deprecated=True` on any artifact route; `artifact_endpoint.py` does **not** import `workspace_router` | — | — |

> **Discrepancy worth reporting:** `constants.py:406` reads
> `DEFAULT_ZENML_SERVER_FILE_DOWNLOAD_SIZE_LIMIT = 2 * 1024 * 1024 * 1024  # 20 GB`. The value is
> **2 GiB**; the comment says 20 GB. One of the two is wrong, and the value is what
> `download_utils.py:62` enforces.

> **Asymmetry worth reporting:** `ZENML_SERVER_ALLOW_LOCAL_FILE_ACCESS` carries the
> `ZENML_SERVER_` prefix (`constants.py:274`) and so *looks* like a `ServerConfiguration`
> field — but it is **not** declared on `ServerConfiguration`. It is read directly via
> `handle_bool_env_var` at `base_artifact_store.py:75`, i.e. it is a mechanism-2 gate wearing a
> mechanism-3 name. Anyone auditing `server_config.py` alone will miss it.

## 4. OSS degradation

| Capability | OSS behaviour | Substitute |
|---|---|---|
| Per-artifact authorization | Every check fails open (`rbac/utils.py:75,110,243,272,316,381`), including the download-token path (`artifact_version_endpoints.py:422`) and the prune path (`:337`) | None — the artifact routers never call `verify_admin_status_if_no_rbac` (`zen_server/utils.py:793`) |
| Artifact visualization curation | Fully present in OSS: `CuratedVisualizationSchema:46`, the router (`curated_visualization_endpoints.py`), `display_name`/`display_order`/`layout_size` (`curated_visualization_schemas.py:77-80`) | — (no degradation) |
| Artifact store caching | Present and **on by default** (`server_config.py:409`) | — |
| Artifact metadata/visualization capture | Present; the cascade defaults to enabled (`orchestrators/utils.py:109`) | — |
| Artifact quotas | Unmetered on both editions | — (a strict parity point) |
| Local-path artifacts served by the server | **Blocked by default** (`base_artifact_store.py:74-79`) | `ZENML_SERVER_ALLOW_LOCAL_FILE_ACCESS=true` — a deliberate path-traversal/SSRF mitigation, not an edition gate |

Net: **artifact management is the closest area in this audit to true OSS/Pro parity.** The only
thing OSS loses is authorization, which is the blanket Tier-C condition, not an
artifact-specific one.

## 5. Self-host path (Tier B)

```
# server
ZENML_SERVER_ARTIFACT_STORE_CACHE_ENABLED=true     # server_config.py:409 (already the default)
ZENML_SERVER_FILE_DOWNLOAD_SIZE_LIMIT=<bytes>      # server_config.py:430; default 2 GiB
ZENML_SERVER_ALLOW_LOCAL_FILE_ACCESS=true          # constants.py:238 → base_artifact_store.py:75
                                                   # ONLY for a trusted single-tenant server

# client / step runtime
ZENML_DISABLE_PATH_MATERIALIZER=true               # constants.py:271 → path_materializer.py:70
ZENML_MATERIALIZER_ALLOW_NON_ASCII_JSON_DUMPS=true # constants.py:268 → built_in_materializer.py:647
```

Cascade knobs are code-level, not env-level:
`@pipeline(enable_artifact_metadata=..., enable_artifact_visualization=...)`
(`pipelines/pipeline_decorator.py:60,272`), `@step(...)`
(`steps/step_decorator.py:70-71,190-191`), or `PipelineRunConfiguration`
(`config/pipeline_run_configuration.py:48,53`).

The two env vars whose value is read **at import time** —
`ZENML_DISABLE_PATH_MATERIALIZER` (`path_materializer.py:70`, a `ClassVar` initializer) and
`ZENML_MATERIALIZER_ALLOW_NON_ASCII_JSON_DUMPS` (`built_in_materializer.py:57`, a module
constant) — cannot be changed after the module is imported. They must be set in the process
environment before `zenml` is imported, which matters for step containers built by the image
builder.

## 6. Blast radius

| Direction | Target |
|---|---|
| `_validate_path` decorator | Wraps **every** `PathType` parameter of **every** filesystem method on `BaseArtifactStore` (`base_artifact_store.py:80-88` builds the arg index; `:150,152` apply it). A change here touches every artifact store flavor at once |
| `instantiate_artifact_store` | `artifacts/utils.py:891` — called by `load_artifact_store:916`, which `download_utils.py:52` and the visualization endpoint both use. Toggling the cache changes instance lifetime for every server-side artifact read |
| Cache lifecycle contract | `artifacts/utils.py:894-899` — "callers must not call `cleanup()`". Any caller that does will close a store other requests are using. `_safe_cleanup` (`artifact_store_cache.py:158`) and `clear:101` are the only sanctioned teardown |
| Materializer auto-registration | `base_materializer.py:60-62` registers on class creation unless `SKIP_REGISTRATION` is set. Nine classes set it: `preexisting_data_materializer.py:37`, `pytorch/base_pytorch_materializer.py:31`, `dataclass_materializer.py:64`, `in_memory_materializer.py:35`, `cloudpickle_materializer.py:51`, `path_materializer.py:70` (env-driven), and the base itself (`:121`); `huggingface_t5_materializer.py:32` sets it **False** to force registration |
| Cascade | `is_setting_enabled` (`orchestrators/utils.py:88`) is shared by the artifact-metadata, artifact-visualization and (elsewhere) cache settings — changing its precedence changes all of them |
| Project scoping | `ProjectScopedFilter.apply_filter` (`models/v2/base/scoped.py:413,451`) applies to artifact filters; `_set_filter_project_id` (`sql_zen_store.py:14481`) is called from the artifact list paths among its 22 sites |
| Curated visualizations | `CuratedVisualizationSchema` (`curated_visualization_schemas.py:46`) has `ON DELETE CASCADE` FKs to both `project` (`:60-67`) and `artifact_visualization` (`:68-76`), with a uniqueness constraint over `(artifact_visualization_id, resource_id, resource_type)` (`:50-56`). Deleting a project silently removes curated visualizations (`project_schemas.py:132`) |
| Download | `verify_artifact_is_downloadable` (`download_utils.py:32`) calls `artifact_store.exists` then `.size` — two remote round-trips per download token, both through the cached store |

## 7. Test coverage

| Path | Covers |
|---|---|
| `tests/unit/artifacts/test_base_artifact.py`, `test_utils.py` | Artifact model + `artifacts/utils.py` helpers |
| `tests/unit/artifact_stores/test_base_artifact_store.py` | **The `_validate_path` guard** — the security-relevant gate |
| `tests/unit/artifact_stores/test_local_artifact_store.py` | Local store |
| `tests/unit/materializers/` — 10 files | `test_base_materializer.py`, `test_built_in_materializer.py`, `test_cloudpickle_materializer.py`, `test_dataclass_materializer.py`, `test_in_memory_materializer.py`, `test_materializer_registry.py`, **`test_path_materializer.py`**, `test_pydantic_materializer.py`, `test_structured_string_materializer.py`, `test_uuid_materializer.py` |
| `tests/unit/zen_server/test_artifact_store_cache.py` | The server-side cache |
| `tests/unit/models/test_artifact_models.py` | Artifact pydantic models |
| `tests/integration/functional/artifacts/`, `tests/integration/functional/materializers/` | End-to-end |
| `tests/integration/functional/cli/test_artifact.py` | `zenml artifact` CLI |

**This is the best-covered of the five areas** — it is the only one with dedicated unit tests for
its gates (`test_base_artifact_store.py` for local-file access, `test_path_materializer.py` for
the skip-registration env var, `test_artifact_store_cache.py` for the cache).

The mechanism-7 cascade resolver is covered too: `tests/unit/orchestrators/test_utils.py:116`
(`test_is_setting_enabled`, exercising all combinations at `:127-176`).

**Remaining gap:** nothing asserts the *run-level override* leg of the cascade
(`config/compiler.py:231-242`, where a run config stamps `enable_artifact_metadata` onto every
step), and there is no unit test for `verify_artifact_is_downloadable`'s size limit
(`download_utils.py:62-70`) or for the `file_download_size_limit` default's 2 GiB / "20 GB"
discrepancy.
