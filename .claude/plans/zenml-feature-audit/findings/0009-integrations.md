# 0009 — Integrations

Repo @ `cc064f100`. Source root `src/zenml/`. Framework reference: the six nullable dotted-path
gates on `ServerConfiguration` (`src/zenml/config/server_config.py:343-349`).

## 1. Surface

| Layer | Path |
|---|---|
| Base class + metaclass | `src/zenml/integrations/integration.py` — `IntegrationMeta:28` (registers on class creation, `:46`), `Integration:50`, `check_installation:60`, `get_requirements:97`, `get_uninstall_requirements:114`, `activate:137`, `flavors:141` |
| Registry | `src/zenml/integrations/registry.py` — `IntegrationRegistry:29`, `register_integration:64`, `_initialize:75` (directory scan), `activate_integrations:99`, `list_integration_names:140`, `select_integration_requirements:149`, `is_installed:225`, `get_installed_integrations:253` |
| Name constants | `src/zenml/integrations/constants.py` — 72 string constants; `LLAMA_INDEX` **commented out at `:47`** while `src/zenml/integrations/llama_index/` still ships (`__init__.py`, `materializers/`) |
| Packages | `src/zenml/integrations/` — **67 directories**, **66** carrying an `Integration` subclass. `skypilot/` is the one exception: a shared base package with no subclass (verified by scanning every `__init__.py`) |
| Flavor registration | `src/zenml/stack/flavor_registry.py` — `register_flavors:46`, `builtin_flavors:56-99` (15 flavors), `integration_flavors:102-113`, `register_builtin_flavors:114`, `register_integration_flavors:145` |
| Packaging | `pyproject.toml:74` `[project.optional-dependencies]` — extras through `:175+` |
| CLI | `src/zenml/cli/integration.py` (545 lines) — `integration:56`, `list_integrations:61`, `get_requirements:79`, `export_requirements:152`, install/uninstall at `:259+` |
| Store hook | `SqlZenStore._sync_flavors` (`src/zenml/zen_stores/sql_zen_store.py:1875-1877`) — "purge all in-built and integration flavors from the DB and sync" |

## 2. Tier verdict — **Tier B**, with an important caveat

**Tier B, not A.** Every integration is fully implemented in-tree; the gate is dependency
presence, not edition. `Integration.check_installation` (`integration.py:60`) walks
`get_requirements()` and its transitive dependencies through
`requirement_installed`/`get_dependencies` (`utils/package_utils.py`), returning `False` on the
first miss (`integration.py:75,83`). No `ServerConfiguration` field, no
`deployment_type == CLOUD` branch, and no entitlement call touches `integrations/`.

**The Tier-B "enable via env var" half of the definition does not hold here.** Claim verified:

> There is NO env var that disables integration activation.

Evidence: `grep -rn "ZENML_[A-Z_]*INTEGRATION[A-Z_]*" src/zenml/ --include="*.py"` returns
**zero** matches; `grep -n "environ|handle_bool_env_var|ENV_"` across
`integrations/registry.py`, `integrations/integration.py` and `stack/flavor_registry.py` returns
**zero** matches. `activate_integrations()` is invoked unconditionally at every entrypoint
(`entrypoints/step_entrypoint_configuration.py:188`,
`entrypoints/pipeline_entrypoint_configuration.py:32`,
`pipelines/pipeline_definition.py:1396`, `pipelines/dynamic/entrypoint_configuration.py:75`,
`deployers/server/app.py:1173`, `deployers/server/entrypoint_configuration.py:145`,
`services/local/local_daemon_entrypoint.py:82`, `services/container/entrypoint.py:62`).
The only dial is what is installed in the Python environment.

**Surprise: flavor *registration* ignores `check_installation` entirely.**
`FlavorRegistry.register_integration_flavors` (`flavor_registry.py:145-152`) iterates
`integration_registry.integrations.items()` and calls `integration.flavors()` on every one —
there is no installation check in that loop, unlike `activate_integrations`
(`registry.py:118`). So a ZenML server writes **all 68 integration flavor rows** to its DB
regardless of which extras are installed. Flavor *listing* is therefore edition- and
install-independent; only instantiation fails later.

## 3. Gate inventory

| # | Mechanism | Location | Effect |
|---|---|---|---|
| **1** | **Dependency presence (dominant gate)** | `Integration.check_installation` (`integration.py:60-94`) via `requirement_installed` + `get_dependencies`; consumed at `registry.py:118` | Integration is not *activated* (materializers, connectors not eagerly registered) |
| 1 | Dependency presence — silent directory scan | `IntegrationRegistry._initialize` (`registry.py:75-97`) `importlib.import_module` per directory; `except ImportError: logger.exception(...)` then `continue` (`:95-96`) | A package that fails to import is **dropped from the registry**; it logs but never raises |
| 1 | Dependency presence — broad activation catch | `registry.py:121-134` catches bare `Exception` (not just `ImportError`/`OSError`, despite the docstring at `:105-109`), logs and continues | Broken installs degrade to "feature missing from auto-discovery" |
| 1 | Platform gate — python version | `great_expectations/__init__.py:34` — `"great-expectations>=0.17.15,<1.0; python_version < '3.14'"` (a PEP 508 marker inside `REQUIREMENTS`) | On Python ≥3.14 the requirement resolves empty → integration reports installed but GE is absent |
| 1 | Platform gate — darwin/arm64 | `mlx/__init__.py:23-38` `_is_supported_platform()` (linux `:33`, darwin+arm64 `:35-37`), overriding `check_installation` at `:48-49`; `get_requirements` at `:59-65` returns `[]` off-platform | MLX is hard-disabled on Intel macOS and on Windows |
| 1 | Platform gate — darwin/arm64 | `tensorflow/__init__.py:38` skips `import tensorflow_io` on Apple Silicon; `get_requirements:55-56` swaps to `tensorflow-macos>=2.12,<2.15` | Different wheel and no `tensorflow_io` on Apple Silicon |
| 2 | Env-var boolean | **ABSENT for integration activation** — see §2. (`handle_bool_env_var` lives at `constants.py:131` but is never called from `integrations/`) | No runtime kill switch |
| 3 | Pluggable implementation source | Flavors are dotted paths resolved through `Flavor.to_model` → `FlavorResponse`; custom flavors registered via `zenml flavor register` | Third-party flavors are first-class — this is the extension seam, not a gate |
| 4 | Entitlement gate | **ABSENT.** No `check_entitlement` import anywhere under `integrations/` or `stack/flavor_registry.py`; no flavor/component type in `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`) | Unlimited stack components on both editions |
| 5 | Deployment-type gate | **ABSENT** | — |
| 6 | Helm value | **ABSENT** — no integration toggles in `helm/values.yaml`; the server image's installed extras are the only lever | Changing available integrations means changing the image |
| 7 | Three-state cascade | **ABSENT** | — |
| 8 | Schema availability | `FlavorSchema` rows written by `register_integration_flavors` (`flavor_registry.py:145`), driven from `SqlZenStore._sync_flavors` (`sql_zen_store.py:1875-1877`); suppressed when `ZENML_DISABLE_DATABASE_MIGRATION` short-circuits `_run_migrations` (`sql_zen_store.py:1551-1557`) | Flavor table can be stale/absent |
| 9 | Deprecation | `LLAMA_INDEX` commented out at `integrations/constants.py:47` while the package directory remains; no `@experimental` decorator exists anywhere in the repo | Soft-retirement by constant removal — the directory still imports during `_initialize` |

## 4. Category breakdown (integration-supplied flavors)

Derived by parsing every `integrations/*/flavors/*.py` for `class <X>Flavor(<Base>)` and mapping
the base class to its `StackComponentType` (`enums.py:214-231`). **68 flavors, 14 of 15 types.**

| Type (`enums.py`) | N | Flavors |
|---|---|---|
| `ORCHESTRATOR` (`:227`) | **17** | Airflow, AzureML, Databricks, HyperAI, Kubeflow, Kubernetes, Lightning, Modal, SSH, Sagemaker, Skypilot{AWS,Azure,GCP,Kubernetes,Lambda}, Tekton, Vertex |
| `STEP_OPERATOR` (`:228`) | **11** | AzureML, Baseten, Databricks, KubernetesSpark, Kubernetes, Modal, RunAI, SSH, Sagemaker, Spark, Vertex |
| `EXPERIMENT_TRACKER` (`:222`) | 6 | Comet, MLFlow, Neptune, Trackio, Vertex, Wandb |
| `MODEL_DEPLOYER` (`:226`) | 6 | BentoML, Databricks, HuggingFace, MLFlow, Seldon, VLLM |
| `ARTIFACT_STORE` (`:219`) | 5 | Azure, B2, DigitalOceanSpaces, GCP, S3 (B2 and DO subclass `S3ArtifactStoreFlavor`) |
| `ANNOTATOR` (`:218`) | 4 | Argilla, LabelStudio, Pigeon, Prodigy |
| `DATA_VALIDATOR` (`:221`) | 4 | Deepchecks, Evidently, GreatExpectations, Whylogs |
| `DEPLOYER` (`:230`) — pipeline-as-a-service | 4 | AWS, GCP, HuggingFace, Kubernetes |
| `IMAGE_BUILDER` (`:224`) | 3 | AWS, GCP, Kaniko |
| `ALERTER` (`:217`) | 2 | Discord, Slack |
| `CONTAINER_REGISTRY` (`:220`) | 2 | AWS, DigitalOcean |
| `SANDBOX` (`:231`) | 2 | Kubernetes, Modal |
| `FEATURE_STORE` (`:223`) | 1 | Feast |
| `MODEL_REGISTRY` (`:229`) | 1 | **MLFlow only** |
| `LOG_STORE` (`:225`) | **0** | No integration supplies one |

Built-in (non-integration) flavors, 15 total (`flavor_registry.py:82-98`):
`LocalArtifactStore`, `LocalOrchestrator`, `LocalDockerOrchestrator`,
`Default/Azure/DockerHub/GCP/GitHub ContainerRegistry`, `LocalImageBuilder`,
`DockerDeployer`, `LocalDeployer`, `DatadogLogStore`, `OtelLogStore`, `DockerSandbox`,
`LocalSandbox`. Note `LOG_STORE` is served **entirely** by built-ins.

`MODEL_REGISTRY` having exactly one implementation is the thinnest surface in the matrix — see
dossier 0012.

## 5. OSS degradation

**None attributable to edition.** Nothing in `integrations/` consults `is_pro_server`
(`server_config.py:820`), `rbac_enabled`, `feature_gate_enabled`, or any `*_implementation_source`.
An OSS server and a Pro workspace running the same image expose the same 68+15 flavors.

What a user actually loses, and the substitute:

| Loss | Cause | Substitute |
|---|---|---|
| An integration's materializers/connectors not auto-registered | `check_installation` returned `False` (`registry.py:118`) | `zenml integration install <name>` (`cli/integration.py:259+`) or install the extra directly |
| An integration silently missing from `zenml integration list` | `_initialize` swallowed an `ImportError` (`registry.py:95-96`) | Raise log level; the exception *is* logged via `logger.exception` |
| MLX on Intel macOS / Windows | `mlx/__init__.py:48-49` hard-returns `False` | none — platform limit |
| `tensorflow_io` filesystem plugins on Apple Silicon | `tensorflow/__init__.py:38` | none — upstream wheel gap |
| Great Expectations on Python ≥3.14 | marker at `great_expectations/__init__.py:34` | pin Python <3.14 |
| LlamaIndex as a named integration | constant commented out at `integrations/constants.py:47` | the `llama_index/materializers/` package still exists and can be imported directly |

## 6. Self-host path

There is no Tier-A gap and no Tier-B env var. The self-host lever is **packaging**:

```
pip install "zenml[connectors-aws,sagemaker,s3fs]"     # pyproject.toml:111-155
zenml integration install kubernetes mlflow            # cli/integration.py:259+
zenml integration export-requirements --installed-only # cli/integration.py:152
```

To add a new integration one writes `integrations/<name>/__init__.py` with an `Integration`
subclass (auto-registered by `IntegrationMeta.__new__`, `integration.py:46`) plus a
`flavors/` package; `_initialize`'s directory scan (`registry.py:75-90`) picks it up with no
central manifest edit — only `integrations/constants.py` needs the name constant for the CLI.

## 7. Blast radius

| Direction | Target |
|---|---|
| Registration (upstream) | `IntegrationMeta.__new__` (`integration.py:38-47`) registers on *class creation*, so `_initialize`'s bare `importlib.import_module` per directory (`registry.py:87`) is the whole discovery mechanism |
| Flavor → DB | `register_integration_flavors` (`flavor_registry.py:145`) → `store.create_flavor` / `update_flavor`, wrapped in `analytics_disabler()` (`:151`). Called from `SqlZenStore._sync_flavors` (`sql_zen_store.py:1877`) on every store init |
| Activation (downstream) | `activate_integrations` (`registry.py:99`) runs in **8** entrypoints (listed in §2) — a slow or broken integration import taxes every pipeline compile and every step container start |
| Failure containment | `registry.py:121-134` catches bare `Exception` around `integration.activate()`; the docstring at `:105-109` explicitly warns that later on-demand imports may still fail independently — i.e. failures are deferred, not eliminated |
| Requirements resolution | `select_integration_requirements:149` / `select_uninstall_requirements:187` feed `zenml integration export-requirements` and Docker image builds — a wrong `get_requirements` override silently changes built images |
| Cross-package coupling | `skypilot/` is imported as a base by `skypilot_{aws,azure,gcp,kubernetes,lambda}/`; it alone has no `Integration` subclass, so it never appears in `list_integration_names` |
| Service connectors (dossier 0008) | `register_builtin_service_connectors` (`service_connector_registry.py:205`) re-imports AWS/GCP/Azure/Kubernetes/HyperAI connector modules **independently** of the integration registry, with its own `try/except ImportError` — two parallel discovery paths over the same packages |

## 8. Test coverage

| Path | Covers |
|---|---|
| `tests/unit/integrations/test_registry.py` | The registry itself |
| `tests/unit/integrations/{b2,digitalocean,gcp,kubernetes,mlflow,ssh}/` | **6** of 67 packages |
| `tests/unit/test_flavor.py` | Flavor model round-trip |
| `tests/integration/functional/cli/test_integration.py` | `zenml integration` CLI (`NOT_AN_INTEGRATION`, `INTEGRATIONS` fixtures) |

**Gaps:** there is no unit test for `Integration.check_installation` platform overrides
(`mlx/__init__.py:48`, `tensorflow/__init__.py:38`); no test asserting that
`register_integration_flavors` registers flavors for *uninstalled* integrations (the §2
surprise); and 61 of 67 integration packages have no `tests/unit/` mirror at all — consistent
with `CLAUDE.md`'s stated exception for code that integrates with external services.
