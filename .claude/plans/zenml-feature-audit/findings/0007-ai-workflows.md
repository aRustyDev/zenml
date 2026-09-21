# 0007 — "AI workflows" → the pipeline-serving surface

Repo `cc064f100`. Every claim carries `file:line`.

---

## 0. Term mapping — stated explicitly

**"AI workflows" is not a ZenML codebase term.** Zero hits in `src/zenml/`. It resolves to the
*serving* surface: a pipeline provisioned as an always-on HTTP service, plus the dynamic-pipeline
runtime, sandboxes and live streaming that agentic workloads need.

Evidence for the mapping:
- `docs/book/how-to/deployment/deployment.md:40` — **"LLM Agent Workflows**: Build intelligent
  agents that combine multiple AI capabilities like intent analysis, retrieval-augmented
  generation (RAG), and response synthesis."
- `docs/book/how-to/deployment/deployment.md:30-32` — "Use **Deployments** when you need an
  always-on HTTP service (inference APIs, agents, interactive workflows)."
- Twelve agent-shaped examples: `examples/{agent_comparison, agent_framework_integrations,
  agent_outer_loop, agentic_hitl_pipeline, deploying_agent, hierarchical_doc_search_agent,
  rlm_document_analysis, sandbox_harbor, sandbox_pydantic_ai, software_factory, weather_agent,
  baseten_robot_policy}`.

**Vocabulary trap.** `deployers/` (this dossier) ≠ `pipeline_deployment` (the legacy wire name for a
snapshot, `zen_server/rbac/models.py:61-62`) ≠ `ServerDeploymentType` (Pro/OSS master switch,
`config/server_config.py:820`). Separate routers: `routers/deployment_endpoints.py` (this area),
`routers/pipeline_deployments_endpoints.py` (deprecated snapshot alias, `:81,:94,:143`),
`routers/pipeline_snapshot_endpoints.py` (snapshots).

---

## 1. Surface

| Layer | Path |
|---|---|
| Deployer component | `src/zenml/deployers/base_deployer.py` (1128): `BaseDeployerSettings:77`, `BaseDeployerConfig:85`, `BaseDeployer:89`, `BaseDeployerFlavor:1104` |
| Concrete deployers | `src/zenml/deployers/docker/docker_deployer.py` (`do_provision_deployment:268`, `do_deprovision_deployment:565`), `src/zenml/deployers/local/local_deployer.py` (`:199`, `:471`), `src/zenml/deployers/containerized_deployer.py` |
| Serving app | `src/zenml/deployers/server/app.py` (1182): `BaseDeploymentAppRunner:81`, `BaseDeploymentAppRunnerFlavor:957`, `build_asgi_app:1057`, `start_deployment_app:1082`; FastAPI impl `deployers/server/fastapi/app.py:61`; `runtime.py` (170), `service.py` (764), `adapters.py` (240), `extensions.py` (49: `BaseAppExtension:23`), `entrypoint_configuration.py` |
| Built-in dashboard | `src/zenml/deployers/server/dashboard/{index.html,assets/dashboard.css,assets/dashboard.js}` |
| Deployer helpers | `src/zenml/deployers/utils.py`: `get_deployment_input_schema:47`, `get_deployment_output_schema:73`, `get_deployment_invocation_example:99`, `invoke_deployment:136`, `deployment_snapshot_request_from_source_snapshot:310`, `load_deployment_requirements:436` |
| Settings | `src/zenml/config/deployment_settings.py`: `AppExtensionSpec:302`, `DeploymentSettings` docstring `:628-650` |
| Sandboxes | `src/zenml/sandboxes/`: `BaseSandboxConfig:50`, `BaseSandbox:63`, `BaseSandboxFlavor:195` (`base.py`); `LocalSandbox:395`/`LocalSandboxFlavor:481` (`local_sandbox.py`); `DockerSandbox:516`/`DockerSandboxFlavor:683` (`docker_sandbox.py`); `process.py`, `session.py`, `snapshot.py` |
| Dynamic pipelines | `src/zenml/pipelines/dynamic/{pipeline_definition,entrypoint_configuration}.py`; runtime `src/zenml/execution/pipeline/dynamic/` (10 modules: `runner`, `compilation`, `future_registry`, `inputs`, `outputs`, `pipeline_output_utils`, `run_context`, `invocation_dependency_graph`, `interactive_input_utils`, `utils`) |
| Streaming (producer) | `src/zenml/streaming/__init__.py` → `publish`, `flush`; `src/zenml/streaming/publishing.py` (`_StreamPublisher:102`, `publish:329`, `flush:315`) |
| Streaming (server) | `src/zenml/zen_server/streaming/{broadcaster,sse,redis_client,run_end_handler,types}.py`; brokers `brokers/{base,frames,memory,redis_streams}.py` |
| Container engines | `src/zenml/container_engines/{base,docker_engine,podman_engine,factory}.py` |
| Model / schema / router | `models/v2/core/deployment.py` (511), `zen_stores/schemas/deployment_schemas.py` (308, `DeploymentSchema:61`, `url:94`, `snapshot_id:111`, `deployer_id:123`), `routers/deployment_endpoints.py` (5 endpoints: `create:64`, `list:87`, `get:118`, `update:145`, `delete:172`) |
| CLI | `src/zenml/cli/deployment.py` (737): `list:74`, `describe:113`, `provision:171`, `deprovision:280`, `delete:435`, `refresh:591`, `invoke:612`, `logs:698`; plus `zenml pipeline snapshot deploy` (`cli/pipeline.py:1686`) |

---

## 2. Tier verdict

| Sub-area | Tier | Evidence |
|---|---|---|
| **Deployers / serving** | **Full OSS, ungated** | `DockerDeployerFlavor` and `LocalDeployerFlavor` are **built-in flavors**: `stack/flavor_registry.py:70` imports them, `:92` and `:94` register them. `routers/deployment_endpoints.py` contains **no** `check_entitlement`, **no** `workload_manager_enabled` guard, **no** `server_config()` reference at all — its only gating import is RBAC (`:38-45`). Provisioning happens **client-side**: `cli/deployment.py:259` → `Client().provision_deployment` (`client.py:3649`) → `deployer.provision_deployment` (`base_deployer.py:458`) → `do_provision_deployment` (`:632`). The server never runs a deployer. |
| **Sandboxes** | **Full OSS, ungated** | `DockerSandboxFlavor` and `LocalSandboxFlavor` registered as built-ins: `stack/flavor_registry.py:80` import, `:95-96` registration |
| **Dynamic pipelines** | **Full OSS, ungated** | `submit_dynamic_pipeline` is implemented by the OSS built-ins — `orchestrators/local/local_orchestrator.py:193`, `orchestrators/local_docker/local_docker_orchestrator.py:305` — so `supports_dynamic_pipelines` (`base_orchestrator.py:539-548`) is `True` for a stock local stack. Client check `pipelines/dynamic/pipeline_definition.py:329-333` |
| **Streaming** | **Tier B — VERIFIED** | Two concrete `StreamBroker` implementations ship in-tree: `InMemoryBroker` (`zen_server/streaming/brokers/memory.py:77`) and `RedisStreamsBroker` (`zen_server/streaming/brokers/redis_streams.py:84`). Neither is loaded by default (`config/server_config.py:348` default `None`; `streaming_enabled` `:731-737`). Redis is documented and Helm-exposed |
| **Container engines** | **OSS + mechanism 1** | `container_engines/factory.py:30` auto-probes Docker then Podman (`:46-57`), raising `RuntimeError` if the requested engine is missing (`:60`) or none is available (`:66`) |

### Streaming Tier-B verification (explicit)

| Fact | Line |
|---|---|
| Config field, default `None` | `config/server_config.py:348` |
| `streaming_enabled` ⇔ source set | `config/server_config.py:731-737` |
| Loader returns early when unset | `zen_server/utils.py:269-271` (`if not source: return`) |
| Accessors raise when unset | `zen_server/utils.py:250-255` (message names the exact config key), `:258-263` |
| Routes 501 when unset | `routers/runs_endpoints.py:767-778` `streaming_enabled()` dependency → `HTTP_501_NOT_IMPLEMENTED`; attached at `:810` (publish) and `:905` (SSE subscribe) |
| Documented self-host path | `docs/book/getting-started/deploying-zenml/live-event-streaming.md:38,49,66` |
| Helm value present (commented) | `helm/values.yaml:65-77` (`streamBrokerImplementationSource:66`, `heartbeatSeconds:69`, `maxSubscribersPerStream:72`, `broadcasterIdleGraceSeconds:77`) |
| `InMemoryBroker` is dev-only *by its own comment* | `zen_server/streaming/brokers/memory.py:88-90`: *"Streams are retained for the lifetime of the process. Running this class in long-running processes like a production server will cause OOM errors."* |
| Redis is an **optional extra** | `pyproject.toml:106` `server-streaming = ["redis>=7.1.0,<8.0.0"]`; import guarded at `zen_server/streaming/redis_client.py:25-30` with a named `ImportError` message |

**⇒ Streaming is genuinely Tier B: a production-grade OSS implementation (`RedisStreamsBroker`) exists and is documented. That is a materially different situation from the workload manager (0004), whose only in-tree implementation is a stub.**

---

## 3. Gate inventory

| # | Mechanism | Where | Effect |
|---|---|---|---|
| 3 | Pluggable implementation source | `config/server_config.py:348` `stream_broker_implementation_source`; loader `zen_server/utils.py:259-…`; lifespan `zen_server/zen_server_api.py:191` | Streaming off by default |
| 3 | Pluggable source (user-level, **not** a tier gate) | `config/deployment_settings.py:302` `AppExtensionSpec`; loaded at `deployers/server/app.py:651-680` (`load_sources()` `:661`, `resolve_extension_handler()` `:662`, subclass check `:667-672`); `deployment_app_runner_flavor` documented `config/deployment_settings.py:648-650` | Users extend the serving app in OSS |
| 1 | Dependency presence | `pyproject.toml:106` `redis` extra; guarded import `zen_server/streaming/redis_client.py:25-30`. Container engines probed at `container_engines/factory.py:46-57` | Redis broker unusable without the extra |
| 2 | Env-var boolean | `execution/pipeline/dynamic/interactive_input_utils.py:46` gates interactive input; `execution/context.py:66` `ENV_ZENML_PREVENT_EXECUTION_CONTEXT_CACHING` | Runtime behaviour, not tiering |
| 4 | Entitlement gate | **NONE** in `routers/deployment_endpoints.py`. Verified: `grep -n "check_entitlement" routers/deployment_endpoints.py` → no match. `ResourceType.DEPLOYMENT` (`rbac/models.py:63`) is not in `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465` = `["project","pipeline","pipeline_run","model"]`) | Deployments are neither gated nor metered |
| 5 | Deployment-type gate | `config/server_config.py:820` `is_pro_server` | Only supplies the Pro sources |
| 6 | Helm value | `helm/values.yaml:66` streaming source; `:73` `heartbeatSeconds`, `:76` `maxSubscribersPerStream`, `:80` broadcaster idle grace. **No Helm value for deployers or sandboxes** — they are client-side | |
| 8 | Schema availability | `migrations/versions/4e1972485075_endpoint_artifact_deployment_artifact.py`, `19f27d5b234e_add_build_and_deployment_tables.py`; skippable via `ENV_ZENML_DISABLE_DATABASE_MIGRATION` (`zen_stores/sql_zen_store.py:1551-1557`) | |
| 7 | Three-state cascade | Not applicable — `enable_*` flags do not extend to deployments | |
| 9 | Deprecation | none in this area | |

Runtime knobs, all OSS-settable: `config/server_config.py:350` `streaming_heartbeat_seconds` (30.0), `:351` `streaming_max_subscribers_per_stream` (100), `:352-354` `streaming_broadcaster_idle_grace_seconds` (30.0).

---

## 4. OSS degradation

**Almost none.** This is the strongest OSS area of the four.

| Capability | OSS status |
|---|---|
| Deploy a pipeline as an HTTP service, locally or on Docker | ✅ works — built-in flavors, client-side provisioning |
| Invoke it (`zenml deployment invoke`, `cli/deployment.py:612`) | ✅ |
| Auto-generated request/response schemas + example (`deployers/utils.py:47,73,99`) | ✅ |
| Built-in serving dashboard (`deployers/server/dashboard/`) | ✅ |
| Custom endpoints / middleware / app extensions / alternate app-runner flavor | ✅ — user-configured via `DeploymentSettings` |
| Dynamic (agentic) pipelines on a local stack | ✅ |
| Sandboxed code execution, local or Docker | ✅ |
| **Live event streaming** | ⚠️ **off by default**; SSE and publish endpoints return **501** (`routers/runs_endpoints.py:773-778`). Client degrades gracefully: `_StreamPublisher._send_batch` catches `NotImplementedError`/`MethodNotAllowedError` and calls `_disable_publishing()` (`streaming/publishing.py:279-281`), which logs once — *"Streaming is disabled on the server. Further publishes will be dropped for the rest of this process."* (`:234,240-243`) — and drops silently thereafter. So `zenml.streaming.publish(...)` in agent code is a **no-op**, not an error. |
| Remote/managed deployers (Kubernetes, cloud runtimes) | Only `local` and `docker` ship as built-ins; anything else must come from an integration or be written |
| Server-side execution of the deployed snapshot's *pipeline runs* | ❌ — inherits 0004's hard block. Deployments serve HTTP; they do not go through `run_snapshot` |

`SqlZenStore.publish_run_events` raises `NotImplementedError` unconditionally (`zen_stores/sql_zen_store.py:5676-5688`, *"the local store has no broker"*), so streaming needs a server regardless of tier.

---

## 5. Self-host path

**Deployers / sandboxes / dynamic pipelines:** nothing to enable. Register the stack component and go:
`zenml deployer register … --flavor=docker`, `zenml sandbox register … --flavor=docker`.

**Streaming (Tier B):**

```bash
# Production
ZENML_SERVER_STREAM_BROKER_IMPLEMENTATION_SOURCE=\
zenml.zen_server.streaming.brokers.redis_streams.RedisStreamsBroker
ZENML_REDIS_BROKER_URL=redis://...        # via server.environmentSecretKeyRefs
pip install "zenml[server-streaming]"      # pyproject.toml:106

# Development only — unbounded in-process growth (memory.py:88-90)
ZENML_SERVER_STREAM_BROKER_IMPLEMENTATION_SOURCE=\
zenml.zen_server.streaming.brokers.memory.InMemoryBroker
```

Helm equivalent at `helm/values.yaml:65-77` (commented out by default). Broker settings load from
env prefixes: `ZENML_IN_MEMORY_BROKER_` (`brokers/memory.py:31`) and
`ZENML_REDIS_STREAMS_BROKER_` (`brokers/redis_streams.py:42`).

Unlike the workload manager, `initialize_streaming` **raises** rather than warns if the source is
bad (`zen_server/utils.py:259-263` docstring: *"Raises RuntimeError: If the configured broker class
can't be loaded or its connectivity probe fails"*) — a typo fails startup loudly, which is the
correct behaviour and the opposite of `initialize_workload_manager` (`:215-217`).

**Remote deployer:** subclass `BaseDeployer` (`base_deployer.py:89`) and implement the four
abstract hooks — `do_provision_deployment:955`, `do_get_deployment_state:1011`,
`do_get_deployment_state_logs:1033`, `do_deprovision_deployment:1060` — plus a
`BaseDeployerFlavor` (`:1104`, abstract `implementation_class` at `:1126`). No server-side change
needed; the component runs client-side.

---

## 6. Blast radius

```
zenml deployment provision            (cli/deployment.py:210)
  └─ Client().provision_deployment    (client.py:3649)
       ├─ deployers/utils.py:310 deployment_snapshot_request_from_source_snapshot
       ├─ orchestrators/utils.py get_config_environment_vars      ◄── cross-module
       └─ deployer.provision_deployment  (base_deployer.py:458)
            └─ do_provision_deployment   (:632 → docker_deployer.py:268 | local_deployer.py:199)
                 └─ container_engines/factory.py:30 get_container_engine   [docker only]
                      └─ container image runs: deployers/server/entrypoint_configuration.py
                           └─ start_deployment_app (server/app.py:1082)
                                └─ build_asgi_app (:1057) → FastAPIDeploymentAppRunner (fastapi/app.py:61)
                                     ├─ install_extensions (app.py:651)  ◄── user AppExtensionSpec
                                     └─ service.py:… → dynamic/static pipeline execution per request

zenml.streaming.publish(...)          (streaming/publishing.py:329)
  └─ _resolve_publish_context :68  ── step context OR DynamicPipelineRunContext.get() :91
  └─ _StreamPublisher.publish :119 → background thread → _send_batch :259
       └─ zen_store.publish_run_events :275
            ├─ SqlZenStore  → NotImplementedError   (sql_zen_store.py:5680)
            └─ RestZenStore → POST (rest_zen_store.py:2464) → routers/runs_endpoints.py:806
                 → Depends(streaming_enabled) :810 → 501 if broker unset
                 → broker.publish → SSE subscribers (routers/runs_endpoints.py:897, streaming/sse.py)
```

**Hops that matter:**
- `streaming/publishing.py:76-78` imports `DynamicPipelineRunContext` from
  `execution/pipeline/dynamic/run_context.py` — **streaming is wired into the dynamic runtime**, so
  agent code can publish from outside a step. Changing the dynamic run context breaks streaming.
- `deployers/utils.py` reuses `orchestrators/utils.get_config_environment_vars` — the deployed app
  authenticates back to the server with the same env-var contract as an orchestrator.
- `DeploymentSchema.snapshot_id` (`schemas/deployment_schemas.py:111`) and `.deployer_id` (`:123`)
  make a deployment a *join* of a snapshot and a stack component. `sql_zen_store.py:4011-4017`
  refuses to delete a deployer stack component while deployments reference it.
- `StreamEndEventHandler` (`zen_server/streaming/run_end_handler.py:33`) is registered **inside**
  `initialize_streaming` (`zen_server/utils.py:322-325`) — so with streaming off, the
  `EventDispatcher` has **zero** handlers, which is also why `sql_zen_store.py:13219`'s
  `has_handlers()` short-circuit fires (see 0005 §2).
- `AppExtensionSpec.load_sources()` executes user-named code inside the deployed container
  (`deployers/server/app.py:661`) — a supply-chain surface, not a tiering one.

---

## 7. Test coverage

| Area | Tests |
|---|---|
| Serving app / runtime | `tests/unit/deployers/server/`: `test_service.py`, `test_runtime.py`, `test_parameter_flow.py`, `test_service_outputs.py`, `conftest.py` |
| Sandboxes | `tests/unit/sandboxes/`: `test_base_sandbox.py`, `test_local_sandbox.py`, `test_docker_sandbox.py` |
| Dynamic pipelines | `tests/unit/pipelines/dynamic/`: `test_pipeline.py`, `test_outputs.py`, `test_entrypoint_configuration.py`; runtime `tests/unit/execution/pipeline/dynamic/` |
| Streaming (producer) | `tests/unit/streaming/test_publishing.py`, `test_stream_event.py` |
| Streaming (server) | `tests/unit/zen_server/streaming/`: `test_broadcaster.py` (uses `InMemoryBroker` + `InMemoryBrokerSettings`, `:31-32,39-40`, and subclasses it as a fault-injector `:180-189`), `test_broker_frames.py`, `test_sse.py`, `conftest.py` |
| Container engines | `tests/unit/container_engine/` |

**Gaps:** no unit tests for `deployers/base_deployer.py`, `docker_deployer.py`, or `local_deployer.py`
(there is no `tests/unit/deployers/test_*.py` — only the `server/` subdir), and none for
`routers/deployment_endpoints.py`. Deployer provisioning is exercised only through the examples and
integration/CI, consistent with `CLAUDE.md`'s exception for external-service integrations.
`RedisStreamsBroker` likewise has no unit test — `test_broadcaster.py` covers the broker contract via
`InMemoryBroker` only.
