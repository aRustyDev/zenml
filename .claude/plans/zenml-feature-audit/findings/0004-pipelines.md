# 0004 — Pipelines (author, compile, execute)

Repo `cc064f100`. Every claim carries `file:line`. Paths relative to repo root.

---

## 1. Surface

| Layer | Path |
|---|---|
| Authoring SDK | `src/zenml/pipelines/pipeline_definition.py` (2029 ln), `src/zenml/pipelines/pipeline_decorator.py`, `src/zenml/steps/base_step.py`, `src/zenml/steps/step_decorator.py` |
| Dynamic authoring | `src/zenml/pipelines/dynamic/pipeline_definition.py`, `src/zenml/pipelines/dynamic/entrypoint_configuration.py` |
| Compilation | `src/zenml/config/compiler.py` (820 ln), `src/zenml/config/pipeline_configurations.py`, `src/zenml/config/step_configurations.py`, `src/zenml/pipelines/compilation_context.py` |
| Build | `src/zenml/pipelines/build_utils.py` |
| Execution (client) | `src/zenml/orchestrators/base_orchestrator.py` (1011 ln), `step_launcher.py`, `step_runner.py`, `step_run_utils.py`, `cache_utils.py`, `dag_runner.py` |
| Execution (runtime) | `src/zenml/execution/context.py`, `src/zenml/execution/pipeline/utils.py`, `src/zenml/execution/pipeline/dynamic/{runner,compilation,outputs,run_context}.py` |
| Entrypoints | `src/zenml/entrypoints/{base_entrypoint_configuration,pipeline_entrypoint_configuration,step_entrypoint_configuration,entrypoint}.py` |
| Server exec | `src/zenml/zen_server/pipeline_execution/{utils,snapshot_run_dispatcher,workload_manager_interface,in_memory_workload_manager,runner_entrypoint_configuration}.py` |
| REST routers | `routers/pipelines_endpoints.py` (231), `runs_endpoints.py` (1005), `steps_endpoints.py` (355), `pipeline_builds_endpoints.py` (175), `pipeline_snapshot_endpoints.py` (411), `pipeline_deployments_endpoints.py` (legacy alias, `deprecated=True` at :81,:94,:143) |
| SQL schemas | `zen_stores/schemas/{pipeline_schemas,pipeline_run_schemas,pipeline_snapshot_schemas,pipeline_build_schemas,step_run_schemas}.py` |
| Migration | `migrations/versions/8ad841ad9bfe_pipeline_snapshots.py:25` — `rename_pipeline_deployment_to_pipeline_snapshot()` |
| CLI | `src/zenml/cli/pipeline.py` (1904 ln); snapshot subcommands at `:1535 create`, `:1645 run`, `:1686 deploy`, `:1829 list`, `:1874 delete` |

**Vocabulary trap.** `ResourceType.PIPELINE_SNAPSHOT = "pipeline_deployment"` (`zen_server/rbac/models.py:61-62`, comment: *"We keep this name for backwards compatibility"*). `ResourceType.DEPLOYMENT = "deployment"` (`:63`) is the *serving* component (area 0007). Three separate routers exist and are not interchangeable.

---

## 2. Tier verdict

**Split verdict — this is the area's central finding.**

| Sub-capability | Tier | Evidence |
|---|---|---|
| Author / compile / build / run **from the client** | **OSS, ungated** | `pipeline_definition.py` `__call__`→`_run`; `LocalOrchestrator` registered as a builtin flavor (`stack/flavor_registry.py:84`). No gate anywhere on this path. |
| Run a snapshot **server-side** | **Tier A** | The only `WorkloadManagerInterface` implementation reachable by OSS is `InMemoryWorkloadManager` (`zen_server/pipeline_execution/in_memory_workload_manager.py:30`) and it is a **stub** — see below. No production implementation exists in-tree. |
| `InMemoryWorkloadManager` as the Tier-B escape hatch | **Tier B, but degenerate** | Referenced from **nowhere** in the repo: `grep -rn "InMemoryWorkloadManager"` over `src/`, `tests/`, `helm/`, `docs/` returns only its own definition at `in_memory_workload_manager.py:30`. Not in `helm/values.yaml`, not in any test. |
| Snapshot run **dispatch** | **Not gated at all — contradicts the framework brief** | `LocalSnapshotRunDispatcher` is the **default** when no source is configured: `zen_server/utils.py:446` reads the source, `:458` falls back to `dispatcher = LocalSnapshotRunDispatcher()`. Tested at `tests/unit/zen_server/test_snapshot_run_dispatcher.py:50`. |

### The hard block, traced end to end

1. `RestZenStore.run_snapshot` (`zen_stores/rest_zen_store.py:1985`) POSTs `POST /pipeline_snapshots/{id}/runs`; on `MethodNotAllowedError` it raises `RuntimeError("Running a snapshot is not supported for this server.")` (`:2006-2010`).
2. That route is registered **only** under `if server_config().workload_manager_enabled:` (`routers/pipeline_snapshot_endpoints.py:351`). Same guard on `POST /run_templates/{id}/runs` (`routers/run_templates_endpoints.py:232`) and `POST /runs/{id}/replay` (`routers/runs_endpoints.py:703`).
3. `workload_manager_enabled` ⇔ `workload_manager_implementation_source is not None` (`config/server_config.py:713-719`), default `None` (`:346`).
4. `SqlZenStore.run_snapshot` raises `NotImplementedError("Running a snapshot is not possible with a local store.")` **unconditionally** (`zen_stores/sql_zen_store.py:5640,5653-5656`). So a direct-SQL client has no path either.
5. Even if the routes existed, `execute_snapshot_run` (`zen_server/pipeline_execution/utils.py:471`) ends in `_build_and_run` (`:560`, defined `:1151`), whose first act is `workload_manager().build_and_push_image(...)` (`:1182`), and `workload_manager()` raises `RuntimeError("Workload manager component not initialized")` when unset (`zen_server/utils.py:196-197`).

**⇒ A default OSS ZenML server cannot run pipelines server-side. The routes do not exist.**

### Does `InMemoryWorkloadManager` close the gap?

Partially, and unsafely.

- `build_and_push_image` **returns the empty string** and builds nothing (`in_memory_workload_manager.py:76-98`, `return ""` at `:98`).
- `run` discards the image and rewrites the entrypoint: `command_to_run[0] = sys.executable` (`:58`), then `subprocess.run(...)` (`:61`) or `subprocess.Popen(...)` (`:75`) **inside the server process's own Python interpreter**.

So setting the env var *does* register the routes and *does* execute runs — but with no container isolation, in the API server's own venv, meaning the snapshot's stack requirements must already be installed in the server image. It is a development shim, and its own module docstring is a copy-paste of the interface's (`:14`). `validate_snapshot_for_server_execution` still rejects local components (`pipeline_execution/utils.py:1056` → `pipelines/run_utils.py:181,189`), so the stack must be fully remote regardless.

---

## 3. Gate inventory

| # | Mechanism | Where | Effect |
|---|---|---|---|
| 3 | Pluggable implementation source | `config/server_config.py:346` field; `:713-719` `workload_manager_enabled`; loader `zen_server/utils.py:201-220`; accessor raises at `:196-197`; wired in lifespan `zen_server/zen_server_api.py:186` | Master switch for all server-side execution |
| 3 | Pluggable implementation source | `config/server_config.py:349` `snapshot_run_dispatcher_implementation_source`; loader `zen_server/utils.py:434-466` | **Defaults to local** (`:458`), not a Pro gate |
| 4 | Entitlement gate | `routers/pipeline_snapshot_endpoints.py:405`, `routers/run_templates_endpoints.py:290`, `routers/runs_endpoints.py:757` — all `check_entitlement(feature=RUN_TEMPLATE_TRIGGERS_FEATURE_NAME)`, constant `constants.py:466` = `"template_run"` | **Fails open in OSS**: `feature_gate/endpoint_utils.py:30-31` `if not server_config().feature_gate_enabled: return` |
| 4 | Usage reporting | `zen_server/pipeline_execution/utils.py:454-457` `report_usage(RUN_TEMPLATE_TRIGGERS_FEATURE_NAME, ...)`; decrement `routers/pipelines_endpoints.py:223-231` | No-ops in OSS (`feature_gate/endpoint_utils.py:41-42`, `:53-54`) |
| 5 | Deployment type | `config/server_config.py:820` `is_pro_server` ⇔ `deployment_type == CLOUD`; overrides applied in `get_server_config()` (`:851-885`) | Forces EXTERNAL auth + cloud sources |
| 6 | Helm value | `helm/values.yaml:297` `rbacImplementationSource`, `:304` `featureGateImplementationSource`, `:66` `streamBrokerImplementationSource` | **`workloadManagerImplementationSource` is absent from `helm/values.yaml`** — only reachable via `ZENML_SERVER_WORKLOAD_MANAGER_IMPLEMENTATION_SOURCE` |
| 7 | Three-state `Optional[bool]` cascade | see §3a | pipeline → step resolution |
| 2 | Env-var boolean (`handle_bool_env_var`, `constants.py:131`) | `orchestrators/base_orchestrator.py:311` `ENV_ZENML_PREVENT_CLIENT_SIDE_CACHING`; `utils/logging_utils.py:405` `ENV_ZENML_DISABLE_PIPELINE_LOGS_STORAGE`, `:428` `ENV_ZENML_DISABLE_STEP_LOGS_STORAGE`; `steps/base_step.py:713` run-without-stack; `execution/context.py:66` `ENV_ZENML_PREVENT_EXECUTION_CONTEXT_CACHING` | Local kill-switches, not tiering |
| 1 | Dependency presence | `container_engines/factory.py:30-69` auto-probes Docker then Podman, raising `RuntimeError` if neither is available (`:65-69`) | Affects containerized orchestrators/builds |
| 8 | Schema availability | `zen_stores/sql_zen_store.py:1551-1557` `_run_migrations` skipped on `skip_migrations` or `ENV_ZENML_DISABLE_DATABASE_MIGRATION` | Same in both tiers |
| 9 | Deprecation | `routers/pipeline_deployments_endpoints.py:81,94,143` `deprecated=True` (legacy wire name for snapshots) | See 0006 |

### 3a. Mechanism 7 — the `enable_*` cascade, precisely

Declared **nullable at both levels**:

| Flag | Pipeline level | Step level |
|---|---|---|
| `enable_cache` | `config/pipeline_configurations.py:57` | `config/step_configurations.py:161` |
| `enable_artifact_metadata` | `:58` | `:165` |
| `enable_artifact_visualization` | `:59` | `:170` |
| `enable_step_logs` | `:60` | `:175` |
| `enable_pipeline_logs` | `:64` | **no step-level twin** |

Resolution function — `orchestrators/utils.py:88-110`:

```
if is_enabled_on_step is not None:      return is_enabled_on_step     # :104-105
if is_enabled_on_pipeline is not None:  return is_enabled_on_pipeline # :106-107
return True                                                          # :109
```

So `None` ≠ `False`; the third state is "inherit, else default-on".

Cascade is applied twice, at two different times:

1. **Run-config push-down**, `config/compiler.py:195-201` sets the pipeline-level values from `PipelineRunConfiguration`, then `:226-249` *overwrites every step invocation* whenever the run-level value `is not None` (four loops: `:227-229`, `:232-236`, `:239-243`, `:246-250`). Note `enable_pipeline_logs` is pushed to the pipeline (`:201`) but has no step loop — it is pipeline-scoped only.
2. **Runtime resolution** via `is_setting_enabled`: `orchestrators/step_runner.py:350-353` (metadata), `:354-357` (visualization); `orchestrators/step_run_utils.py:163`, `:258`, `:503` (cache); `utils/logging_utils.py:426-433` (step logs, after the env-var kill-switch at `:428`).

`enable_pipeline_logs` resolves in `utils/logging_utils.py:405` (env-var kill-switch) with no `is_setting_enabled` call — there is nothing to cascade from.

**None of these five flags is tiered.** They are the same in OSS and Pro.

### 3b. Orchestrator capability gates

Four live on **`BaseOrchestratorConfig`** (class at `base_orchestrator.py:94`) and are overridden per flavor:

| Property | Line | Base default | Consumed at |
|---|---|---|---|
| `is_synchronous` | `:122` | `False` (`:128`) | flavor-specific |
| `is_schedulable` | `:131` | `False` (`:137`) | `pipelines/pipeline_definition.py:984` → `ValueError` "does not support scheduling" |
| `supports_client_side_caching` | `:140` | `True` (`:146`) | `base_orchestrator.py:317` |
| `handles_step_retries` | `:149` | `False` (`:155`) | `base_orchestrator.py:535` → `retry=not self.config.handles_step_retries` |

Four live on **`BaseOrchestrator`** (class at `:158`) and are **introspective**, not declarative — they return "did the subclass override the method?":

| Property | Line | Test |
|---|---|---|
| `supports_dynamic_pipelines` | `:539-548` | `submit_dynamic_pipeline.__func__ is not BaseOrchestrator.submit_dynamic_pipeline` |
| `can_run_isolated_steps` | `:552-560` | same idiom on `submit_isolated_step` |
| `supports_schedule_updates` | `:928-937` | same idiom on `update_schedule` |
| `supports_schedule_deletion` | `:940-949` | same idiom on `delete_schedule` |

`is_schedulable` is overridden `True` by 12 integration flavors (airflow `:132`, azureml `:85`, databricks `:162`, hyperai `:83`, kubeflow `:228`, kubernetes `:395`, lightning `:105`, modal `:62`, sagemaker `:277`, ssh `:57`, vertex `:185`). Built-in `LocalOrchestrator` / `LocalDockerOrchestrator` do **not** — local stacks cannot schedule at all, in either tier.

`submit_dynamic_pipeline` **is** implemented by the OSS built-ins (`orchestrators/local/local_orchestrator.py:193`, `orchestrators/local_docker/local_docker_orchestrator.py:305`) plus six integrations. Dynamic pipelines are therefore fully available in OSS; the client-side check is `pipelines/dynamic/pipeline_definition.py:329-333`.

---

## 4. OSS degradation

| Lost | Substitute |
|---|---|
| `POST /pipeline_snapshots/{id}/runs` — trigger a run from the dashboard/API | Run from a client that has the code: `python run.py`, or `zenml pipeline snapshot run` **against a Pro server only** (`cli/pipeline.py:1663` → `Client().trigger_pipeline` → `rest_zen_store.run_snapshot` → `RuntimeError` on OSS) |
| `POST /runs/{id}/replay` (`runs_endpoints.py:703-765`) | Re-run the pipeline from source |
| `POST /run_templates/{id}/runs` (`run_templates_endpoints.py:232-324`) | deprecated anyway — see 0006 |
| Resume of a failed dynamic run — `resume_run` refuses outright: `IllegalOperationError("Resuming runs is only possible when the workload manager is enabled.")` (`pipeline_execution/utils.py:613-616`) | none |
| Server-side runner logs — `runs_endpoints.py:626` guards log retrieval on `workload_manager_enabled`, `:646` calls `workload_manager().get_logs` | Step logs still land in the artifact store via `logging_utils` |
| Metering | **Nothing is lost** — Tier C. OSS is *unmetered*: `check_entitlement` returns immediately (`feature_gate/endpoint_utils.py:30-31`), so `template_run` is unlimited **where the route exists at all** |

The `check_entitlement(RUN_TEMPLATE_TRIGGERS_FEATURE_NAME)` calls in this area are effectively dead code in OSS for a second reason: all three sit *inside* `if workload_manager_enabled:` blocks, so on a default OSS server they are unreachable rather than merely fail-open.

---

## 5. Self-host path

Tier A for production. The Tier-B shim exists but is not production-viable.

```bash
ZENML_SERVER_WORKLOAD_MANAGER_IMPLEMENTATION_SOURCE=\
zenml.zen_server.pipeline_execution.in_memory_workload_manager.InMemoryWorkloadManager
```

Loader: `zen_server/utils.py:208-220`. Note `:215-217` swallows `ModuleNotFoundError`/`KeyError` with only `logger.warning("Unable to load workload manager source.")` — **a typo in the source string leaves the server silently in OSS mode**, routes absent, no startup failure. Contrast `initialize_snapshot_run_dispatcher`, which re-raises (`:459-465`).

What a real implementation must provide (`pipeline_execution/workload_manager_interface.py`): `run` (`:34`), `build_and_push_image` (`:61`), `delete_workload` (`:88`), `get_logs` (`:97`), `log` (`:109`). In practice: a Kubernetes Job/pod launcher plus an image builder with registry push. There is no such class in this repo.

Not exposed in Helm — you must inject the env var via `server.environmentVariables` yourself; unlike `rbacImplementationSource` (`helm/values.yaml:297`) and `featureGateImplementationSource` (`:304`) there is no first-class value.

---

## 6. Blast radius

**Inbound to `run_snapshot` (`pipeline_execution/utils.py:287`)** — exactly three callers, all inside `workload_manager_enabled` blocks: `routers/runs_endpoints.py:759`, `routers/pipeline_snapshot_endpoints.py:407`, `routers/run_templates_endpoints.py:317`.

**Outbound chain:**
```
run_snapshot :287
 ├─ prepare_snapshot_run :387 ── create_placeholder_run + report_usage :454
 ├─ sync ──► execute_snapshot_run :471
 └─ async ─► snapshot_run_dispatcher().submit  :353
              └─ LocalSnapshotRunDispatcher.submit (snapshot_run_dispatcher.py:59)
                   └─ snapshot_executor().submit(execute_snapshot_run) (:69)
                        └─ execute_snapshot_run :471
                             ├─ validate_snapshot_for_server_execution :1009
                             │    └─ validate_stack_is_runnable_from_server (run_utils.py:158)
                             │         └─ rejects is_custom flavors :181, is_local components :189
                             ├─ build_runner_environment :1066
                             ├─ build_runner_dockerfile :1116
                             └─ _build_and_run :1151
                                  ├─ workload_manager().build_and_push_image :1182  ◄── HARD DEP
                                  ├─ workload_manager().log :1190
                                  └─ workload_manager().run :1201
```

**Cross-module hops that matter:**
- `pipeline_execution/utils.py:66` imports `validate_stack_is_runnable_from_server` from `pipelines/run_utils.py` — the server reuses the **client** validator, so client and server agree on "runnable".
- `RunnerEntrypointConfiguration` (`pipeline_execution/runner_entrypoint_configuration.py`) is the contract between the server and the runner container; changing it changes the image ABI.
- Concurrency ceiling: `BoundedThreadPoolExecutor` (`pipeline_execution/utils.py:145`), sized by `max_concurrent_snapshot_runs` (`config/server_config.py:355`); overflow raises `MaxConcurrentTasksError` → `SnapshotRunQueueFullError` (`snapshot_run_dispatcher.py:31`), and `run_snapshot:355-366` **deletes the placeholder run and the copied snapshot** on rejection.
- `report_usage` at `:454` fires on the *prepare* step, before execution — a queue-full rejection reports a usage it then rolls back only in the DB, not in the feature gate.

---

## 7. Test coverage

| Area | Tests |
|---|---|
| Compiler / cascade | `tests/unit/config/test_compiler.py`, `test_pipeline_configurations.py`, `test_step_configurations.py` |
| Orchestrator capability gates | `tests/unit/orchestrators/test_base_orchestrator.py`, `test_step_launcher.py`, `test_step_runner.py`, `test_cache_utils.py`, `test_utils.py` (covers `is_setting_enabled`) |
| Pipelines | `tests/unit/pipelines/{test_base_pipeline,test_build_utils,test_run_utils,test_schedule}.py`; dynamic at `tests/unit/pipelines/dynamic/` (3 files) |
| Steps | `tests/unit/steps/` (7 files) |
| Execution runtime | `tests/unit/execution/test_context.py`, `tests/unit/execution/pipeline/dynamic/` |
| Server execution | `tests/unit/zen_server/test_pipeline_execution_utils.py`, `tests/unit/zen_server/test_snapshot_run_dispatcher.py` |
| Entrypoints | `tests/unit/entrypoints/` |

**Gap:** `InMemoryWorkloadManager` has **zero** tests — it is referenced nowhere outside its own definition. `test_snapshot_run_dispatcher.py:50` asserts the *local* dispatcher is the default, confirming §2's contradiction of the framework brief.
