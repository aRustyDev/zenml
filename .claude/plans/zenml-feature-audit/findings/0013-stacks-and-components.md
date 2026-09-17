# 0013 — Stacks & Stack Components

Repo @ `cc064f100`. Source root `src/zenml/`. Framework: the six nullable dotted-path gates on
`ServerConfiguration` (`config/server_config.py:343-349`) and the nine mechanisms in
`analysis/0001-gate-taxonomy.md` §5. Neighbours: **0008** (service connectors — the connector
linkage is derived there, not here) and **0009** (integrations — the flavor *inventory* is
enumerated there, not here).

## 1. Surface

| Layer | Path |
|---|---|
| Stack aggregate | `stack/stack.py` (`Stack:92`, `from_model:204`, `from_components:271`, `from_components_v2:439`, `validate:1334`, `submit_pipeline:1468`) — 1,624 lines |
| Component base | `stack/stack_component.py` (`StackComponentConfig`, `StackComponent`, `from_model:405`, `get_connector:577`) — 818 lines |
| Flavor abstraction | `stack/flavor.py` (`Flavor:31`, `from_model:135`, `to_model:175`, `validate_flavor_source:273`) |
| Flavor registry | `stack/flavor_registry.py` (`register_flavors:46`, `builtin_flavors:56-99`, `integration_flavors:102-113`, `register_builtin_flavors:115`, `register_integration_flavors:145`) |
| Validator | `stack/stack_validator.py` (`StackValidator:28`, `validate:56`) — 84 lines, no gates |
| Helpers | `stack/utils.py` (`validate_stack_component_config:29`, `warn_if_config_server_mismatch:109`, `get_flavor_by_name_and_type_from_zen_store:145`) |
| Secret mixin | `stack/authentication_mixin.py` (`AuthenticationConfigMixin:25`, `get_authentication_secret:56`) |
| Cloud deploy wizard | `stack_deployments/` — `stack_deployment.py:34` (`ZenMLCloudStackDeployment`, 6 abstract classmethods), `{aws,gcp,azure}_stack_deployment.py`, `utils.py:30` (`STACK_DEPLOYMENT_PROVIDERS`), `constants.py` (4 Terraform version pins). 1,327 lines |
| Domain models | `models/v2/core/stack.py:61,165,172,297,462` · `component.py:60,104,136,143,246,396` · `flavor.py:41,110,245,412` |
| SQL schemas | `zen_stores/schemas/stack_schemas.py` (`StackCompositionSchema:55`, `StackSchema:100`) · `component_schemas.py:53` · `flavor_schemas.py:40` |
| REST routers | `routers/stacks_endpoints.py` (5 ops) · `stack_components_endpoints.py` (6 ops + `types_router:55`) · `flavors_endpoints.py` (6 ops) · `stack_deployment_endpoints.py` (3 ops) — mounted unconditionally at `zen_server_api.py:317,340-343` |
| CLI | `cli/stack.py` (2,414 lines, 18 commands) · `cli/stack_components.py` (1,848 lines — a **generator**: `register_all_stack_component_cli_commands:1622` loops `for component_type in StackComponentType`, emitting ~12 subcommands × 15 types) |
| Type enum | `StackComponentType` (`enums.py:214-231`) — 15 types; `.plural:233` special-cases `container_registries`, `model_registries`, `sandboxes` |
| Migrations | 13 stack/component/flavor revisions; the load-bearing one is `3b1776345020_remove_workspace_from_globals.py` |

## 2. Tier verdict — **Open**

Ships and works in full; **zero Pro coupling exists in the source**. Verified negatively:

```
grep -rn "is_pro_server|workload_manager|check_entitlement|report_usage|report_decrement|
          ServerDeploymentType|feature_gate"  stack/ stack_deployments/
          routers/{stacks,stack_components,flavors,stack_deployment}_endpoints.py
          cli/stack.py cli/stack_components.py          → 0 matches
```

No `*_implementation_source` field governs any of it, no router is wrapped in a
`if server_config().workload_manager_enabled:` block (contrast `run_templates_endpoints.py:232`),
and no stack resource appears in `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`). Stacks are
therefore **unmetered on both editions** — Pro does not cap how many you create.

What is missing in OSS is only authorization. All 41+5 RBAC calls early-return
(`rbac/utils.py:75,110,243,272,316,381,812,839,860`), and `get_allowed_resource_ids` returns
`None` = "full access" at `rbac/utils.py:381-382`, so list endpoints are unfiltered. **No stack
endpoint calls `verify_admin_status_if_no_rbac` (`zen_server/utils.py:793`)** — the fallback used
by `users_`, `service_accounts_` and `secrets_endpoints.py`. OSS has *no* authorization here at
all, not even the `is_admin` degradation.

### 2.1 `stack_deployment_endpoints.py` — **OSS, definitively**

The question posed to this dossier. Answer: the cloud stack-deployment wizard is **entirely
open source**, and the four things that could have gated it are all absent.

| Candidate gate | Reality |
|---|---|
| `is_pro_server` / `ServerDeploymentType.CLOUD` | absent from the router and from all of `stack_deployments/` |
| `workload_manager_enabled` | absent — the router is registered unconditionally (`zen_server_api.py:340`) |
| `check_entitlement` / HTTP 402 | absent |
| A Pro-hosted provisioning service | absent — the router calls `get_stack_deployment_class(provider)` (`utils.py:37`) **in-process** and returns a URL the user's own browser opens |

All three provider classes ship (`utils.py:30-34`, enum `enums.py:603-609`) with full CloudFormation
and Terraform config generation (`aws_stack_deployment.py:241`, `gcp_…:…`, `azure_…:…`).
`GET /stack-deployment/config` mints a fresh **6-hour** access token
(`stack_deployment_endpoints.py:130-136`, `STACK_DEPLOYMENT_API_TOKEN_EXPIRATION = 60 * 6`,
`constants.py:634`) scoped to the caller, and the provisioned Terraform/CFN stack registers itself
back with that token.

Two real constraints, neither of them editions:

1. **Server-only, not local.** `SqlZenStore.get_stack_deployment_info:11948`,
   `get_stack_deployment_config:11968` and `get_stack_deployment_stack` all
   `raise NotImplementedError("Stack deployments are not supported by local ZenML deployments.")`.
   The CLI pre-empts this at `cli/stack.py:1915-1922` — and its error text nudges toward
   "a managed ZenML Pro server instance … https://www.zenml.io/pro". **That sentence is marketing,
   not a gate**: a self-hosted OSS server satisfies the condition. The same nudge appears on the
   `--provider`/`--connector` path of `zenml stack register` (`cli/stack.py:331-341`).
2. **ZenML-hosted external artifacts.** The AWS path templates
   `https://zenml-cf-templates.s3.eu-central-1.amazonaws.com/aws-ecr-s3-sagemaker.yaml`
   (`aws_stack_deployment.py:267`); the Terraform path pins the public registry modules
   `zenml-io/zenml-stack/{aws,gcp,azure}` (`:309`, `gcp_…:290`, `azure_…:295`) and the
   `zenml-io/zenml` provider (`aws_stack_deployment.py:293`). Public, but network-dependent —
   an air-gapped OSS server cannot use the wizard.

### 2.2 Flavor registration ignores `check_installation()` — confirmed, with the consequence

`FlavorRegistry.register_integration_flavors` (`flavor_registry.py:145`) iterates
`integration_registry.integrations.items()` at `:152` and calls `integration.flavors()` at `:154`
with **no installation check anywhere in `:145-179`** — unlike `activate_integrations`, whose gate
is `if integration.check_installation():` at `integrations/registry.py:117`. (Dossier 0009 §2 and
taxonomy §8 cite `:118`; the line is **117**.) Failures are swallowed per-integration by a bare
`except Exception` → `logger.warning` at `:175-179`.

This works because `Integration.flavors()` (`integrations/integration.py:141`) returns *flavor
classes*, and `src/zenml/integrations/AGENTS.md:9` mandates **"NEVER import integration libraries
at the top level in flavor files"** — the third-party import is deferred into the
`implementation_class` property (`AGENTS.md:20-21`). `Flavor.from_model` reinforces this by passing
`validate_component_classes=False` (`stack/flavor.py:152`), so even loading a flavor never touches
the heavy import.

**Consequence, stated precisely:**

| Stage | Behaviour |
|---|---|
| `GET /flavors`, `zenml <type> flavor list` | Returns **all** integration flavors. Install-independent, edition-independent, identical on every server built from the same image |
| `POST /components` (register) | **Succeeds.** `validate_stack_component_config` is called with `validate_custom_flavors=False` (`stack_components_endpoints.py:113-120`, also `sql_zen_store.py:3768-3775`), so a config for an uninstalled flavor is persisted |
| `Stack.from_model` → **`StackComponent.from_model`** | **Fails here.** `flavor.implementation_class(...)` at **`stack/stack_component.py:429`** raises `ImportError`, caught at `:446`, and the message branches on `integration_registry.is_installed(...)` at `:453`: "requirements broken, reinstall" vs "integration not installed, `zenml integration install <x>`" (`:455-477`) |

So the failure is deferred from *listing* to *first client-side use*, and the diagnostic is good.
The cost is a dashboard/CLI flavor list that advertises ~68 integration flavors the local
environment cannot instantiate.

### 2.3 Built-in flavor inventory (15, from `flavor_registry.py:82-98`)

| `StackComponentType` | N | Built-in flavors |
|---|---|---|
| `CONTAINER_REGISTRY` (`enums.py:220`) | 5 | Default, Azure, DockerHub, GCP, GitHub |
| `ORCHESTRATOR` (`:227`) | 2 | Local, LocalDocker |
| `DEPLOYER` (`:230`) | 2 | Docker, Local |
| `LOG_STORE` (`:225`) | 2 | Datadog, Otel |
| `SANDBOX` (`:231`) | 2 | Docker, Local |
| `ARTIFACT_STORE` (`:219`) | 1 | Local |
| `IMAGE_BUILDER` (`:224`) | 1 | Local |
| **8 remaining types** — `ALERTER`, `ANNOTATOR`, `DATA_VALIDATOR`, `EXPERIMENT_TRACKER`, `FEATURE_STORE`, `MODEL_DEPLOYER`, `STEP_OPERATOR`, `MODEL_REGISTRY` | **0** | none — an installed integration is mandatory (inventory in dossier 0009 §4) |

`LOG_STORE` is the inverse of the others: served **only** by built-ins, with a further unregistered
fallback — `Stack.validate_log_store` (`stack/stack.py:1413`) injects `ArtifactLogStore`
(`:1418-1422`) when none is configured. `ArtifactLogStore` has **no `Flavor` class**
(`log_stores/artifact/` contains only `artifact_log_store.py` and `artifact_log_exporter.py`), so
it is invisible to `zenml log-store flavor list`.

## 3. Gate inventory

| # | Mechanism | Location | Effect |
|---|---|---|---|
| **1** | **Dependency presence — deferred, not gating registration** | `register_integration_flavors` (`flavor_registry.py:145-179`) has **no** `check_installation()`, unlike `registry.py:117`; enforced instead at `stack_component.py:429` → `:446-477` | Flavors listed everywhere; `StackComponent.from_model` raises `ImportError` on first instantiation |
| 1 | Dependency presence — server-side leniency | `validate_stack_component_config(..., validate_custom_flavors=False)` at `stack_components_endpoints.py:119` and `sql_zen_store.py:3774`; returns `None` at `stack/utils.py:80-81` | The **server** persists components whose flavor it cannot import. Deliberate (comment at `stack_components_endpoints.py:118`) |
| **2** | **Env-var boolean** | `ENV_ZENML_SKIP_STACK_VALIDATION` (`constants.py:236`) → `handle_bool_env_var` at `stack/stack.py:1352`, default `False` | Skips *all* of `validate()`: image-builder default, log-store default, every `StackValidator`, and the secret existence check (`:1356-1362`) |
| **2** | **Env-var boolean** | `ENV_ZENML_SKIP_IMAGE_BUILDER_DEFAULT` (`constants.py:235`) → `stack/stack.py:1377-1379`, default `False` | Suppresses auto-injection of a `temporary_default` `LocalImageBuilder` when the stack requires one (`:1371-1376`). The unit suite relies on it (`tests/unit/stack/conftest.py:33-35`) |
| 2 | Env-var boolean (inherited) | `ENV_ZENML_ENABLE_IMPLICIT_AUTH_METHODS` (`constants.py:242`) fires through `StackComponent.get_connector` (`stack_component.py:577`) → `Client().get_service_connector_client` (`:620`) | A connector-backed component fails with `RuntimeError` (`:644-648`) wrapping the connector's `AuthorizationException`. **Derived in dossier 0008 §3** |
| 3 | Pluggable implementation source | Flavors are dotted paths (`Flavor.to_model` → `source_utils.resolve(...).import_path`, `flavor.py:210`), resolved by `validate_flavor_source` (`:273`) via `source_utils.load` (`:300`) | The public extension seam: `zenml <type> flavor register <dotted.path>`. Not an edition gate — third-party flavors are first-class |
| **4** | **Entitlement gate** | **ABSENT.** Zero `check_entitlement`/`report_usage` imports in all four routers; `stack`, `stack_component`, `flavor` absent from `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`) | Unlimited stacks, components and custom flavors on **both** editions |
| **5** | **Deployment-type gate** | **ABSENT.** Only the blanket `is_pro_server` config override (`server_config.py:851-885`) applies | — |
| 5′ | Store-type gate (not edition) | `SqlZenStore.get_stack_deployment_{info,config,stack}` (`sql_zen_store.py:11948,11968,11992`) raise `NotImplementedError`; pre-empted at `cli/stack.py:1915-1922` and `:331-341` by `client.zen_store.is_local_store()` | The deploy wizard needs a **server** (OSS or Pro), not a local SQLite store |
| **6** | **Helm value** | **ABSENT.** `grep -n "stack\|flavor\|component" helm/values.yaml` → 0 matches | Nothing about stacks is deploy-time configurable |
| 7 | Three-state `Optional[bool]` cascade | **ABSENT** | — |
| 8 | Schema availability | 13 revisions (`3b1776345020`, `9306debdfe0c_extending_stack_composition`, `bea8a6ce3015_port_flavors_into_database`, …); flavor rows written by `SqlZenStore._sync_flavors` (`sql_zen_store.py:1873,1875-1877`) on every store init; suppressed when `ZENML_DISABLE_DATABASE_MIGRATION` short-circuits `_run_migrations` (`:1551-1557`) | An un-migrated DB yields an empty `flavor` table → every `POST /components` fails at `get_flavor_by_name_and_type_from_zen_store` (`stack/utils.py:166-170`) |
| 9 | Deprecation | `workspace_router` imported from `projects_endpoints.py` (`stacks_endpoints.py:44`, `stack_components_endpoints.py:42`), mounted at `stacks_endpoints.py:64,142` and `stack_components_endpoints.py:68,134`, each `deprecated=True` with the comment "only kept for dashboard compatibility and can be removed after the migration" | See §4.1 — the legacy path segment is **inert** |

## 4. OSS degradation

| Capability | OSS behaviour | Substitute |
|---|---|---|
| Per-stack / per-component authorization | Every check fails open; `get_allowed_resource_ids` → `None` (`rbac/utils.py:381-382`) so `list_stacks`/`list_stack_components` are unfiltered | **None.** Unlike users/secrets, there is no `verify_admin_status_if_no_rbac` fallback anywhere in this area |
| `PATCH /flavors/sync` | `verify_permission(FLAVOR, UPDATE)` at `flavors_endpoints.py:190` fails open, then calls the **private** `zen_store()._sync_flavors()` (`:191`) — a server-wide purge-and-re-register (`sql_zen_store.py:1875-1877`) | None. **Any authenticated user, including a service account, can rewrite the whole flavor table** |
| Stack/component deletion | `verify_permissions_and_delete_entity` (`stacks_endpoints.py:261`, `stack_components_endpoints.py:272`) fails open | Only the hardcoded default-object guards survive: `IllegalOperationError` on the `default` stack (`sql_zen_store.py:11761,11843`) and `default` component (`:3917,3996`) |
| Resource-pool attachment for components | `ResourcePoolSubjectPolicySchema.component_id` (`resource_pool_policy_schemas.py:82-89`, `ondelete="CASCADE"`) and `ResourceRequestSchema.component_id` (`resource_request_schemas.py:67-71`) are FKs to `stack_component`. The whole subsystem is **HTTP 501** in OSS (dossier 0002) | None — Pro-only |
| Sharing / resource membership | `update_resource_membership` and the delete hooks early-return (`rbac/utils.py:812,839,860`) | Everything is effectively shared — codified by `tests/.../test_zen_store.py:3069` `test_stacks_are_accessible_by_other_users`, which asserts a *second* user sees the first user's stack (and skips on SQL stores: "SQL Zen Stores do not support stack scoping", `:3073-3074`) |
| Cloud deploy wizard | **Not degraded** — see §2.1 | — |
| Flavors for 8 of 15 component types | Requires an installed integration (§2.3) | `zenml integration install <name>` — dossier 0009 §6 |

### 4.1 Project scoping: stacks are **global**, deliberately

`ResourceType.STACK` (`rbac/models.py:72`), `STACK_COMPONENT` (`:73`) and `FLAVOR` (`:56`) are all
in the **exclusion list** of `is_project_scoped()` (`rbac/models.py:83-102`, entries at
`:90,93,94`) — alongside `SECRET`, `SERVICE_CONNECTOR`, `TAG`, `SERVICE_ACCOUNT`, `PROJECT` and the
two resource-pool types. `Resource.validate_project_id` (`:160-185`) *rejects* a `project_id` on
these types outright (`:179-183`).

This is not an RBAC-layer convenience — it is the storage model. `grep -n project` over
`stack_schemas.py`, `component_schemas.py`, `flavor_schemas.py` and over
`models/v2/core/{stack,component,flavor}.py` returns **zero matches** in all six files. All three
models derive from `UserScoped*` (`stack.py:61,462` · `component.py:104,396` · `flavor.py:41,412`),
never `ProjectScoped*`.

It was removed on purpose. Migration `3b1776345020_remove_workspace_from_globals.py` (2025-02-13)
drops `workspace_id` from `flavor`, `secret`, `service_connector`, `stack` and `stack_component`
(`:43-72`), refuses to run if more than one workspace exists (`:28-41`), and is explicitly
**irreversible** (`:75-81`).

**What that means for multi-tenancy:** a ZenML server has exactly one global stack namespace. Two
projects cannot hold same-named, differently-configured stacks — the DB unique constraints are
global (`flavor_schemas.py:51-57`: `unique_flavor_name_and_type`). On Pro, per-stack isolation has
to come from RBAC resource membership on a *global* resource; **in OSS, with RBAC absent, there is
no isolation mechanism at all** — every project sees and can mutate every stack. Stack-level
tenancy therefore requires one server per tenant, not one project per tenant.

The legacy `/workspaces/{project_name_or_id}/stacks` routes make this look otherwise and are
**inert**: `project_name_or_id` is declared at `stacks_endpoints.py:73,150` and
`stack_components_endpoints.py:77,145` and is **never read in any handler body** — only the route
path and the docstring mention it.

### 4.2 A Pro-side field leaks into the OSS schema

`StackComponentSchema.update` excludes `"attach_resource_pools"` and `"detach_resource_pools"`
from its `model_dump` (`component_schemas.py:219-220`). Neither field exists on `ComponentUpdate`
(`models/v2/core/component.py:143-179`), and `grep -rn "attach_resource_pools" src/zenml/`
returns **only those two lines**. Pydantic silently ignores unknown `exclude` keys, so the
exclusion is a no-op — a vestigial hook for a Pro `ComponentUpdate` subclass carrying resource-pool
attachment. Same shape as `update_trigger_snapshot_dispatch_state` (FINDINGS.md "Dead symbols").

## 5. Self-host path

**There is no Tier-A gap and nothing to implement.** The only levers are packaging and two
validation escape hatches, all OSS:

```
pip install "zenml[connectors-aws,s3fs,sagemaker]"   # 8 of 15 component types need an integration
zenml integration install kubernetes mlflow          # cli/integration.py:259+

ZENML_SKIP_STACK_VALIDATION=true          # constants.py:236 → stack/stack.py:1352
ZENML_SKIP_IMAGE_BUILDER_DEFAULT=true     # constants.py:235 → stack/stack.py:1377
ZENML_ENABLE_IMPLICIT_AUTH_METHODS=true   # only if components use connectors — dossier 0008 §5
```

Both skip flags are **escape hatches, not features** — they weaken correctness checks. Neither has
a Helm key; neither has an `*_enabled` property to assert against.

To use the cloud deploy wizard, the one requirement is that the client talk to a **remote server**
(`cli/stack.py:1915`), which a self-hosted OSS deployment satisfies. To add a flavor:
`zenml <type> flavor register <dotted.path>` (`cli/stack_components.py:1044`) — no server change,
no manifest edit.

What OSS cannot get without writing code is the RBAC layer that would make the global stack
namespace safe for more than one trusted team (dossier 0001).

## 6. Blast radius

| Direction | Target |
|---|---|
| `Stack` (upstream) | `gitnexus impact` — **270** symbols, risk **CRITICAL** (8 direct, 262 at depth 2). Direct importers: `client.py`, `stack/__init__.py`, `cli/stack.py`, plus 4 Databricks/MLflow/HuggingFace integration modules |
| `StackComponent` (upstream) | **421** symbols, risk **CRITICAL** (40 direct). Modules hit directly: Annotators (8), Alerter (7), Orchestrators (6), Experiment trackers (5), Artifact stores (4), Deployers (3), Step operators (3) — i.e. every component family |
| Flavor → DB (write path) | `SqlZenStore._sync_flavors` (`sql_zen_store.py:1875`) → `FlavorRegistry.register_flavors:46` → `register_builtin_flavors:115` + `register_integration_flavors:145`, both wrapped in `analytics_disabler()` (`:121,151`). Runs **on every store init** and again on every `PATCH /flavors/sync` |
| Component → connector (lateral) | `StackComponentSchema.connector_id` FK with `ondelete="SET NULL"` (`component_schemas.py:107-114`) → deleting a connector silently **unlinks** every component using it; the component survives and fails at `get_connector` (`stack_component.py:583`). Dossier 0008 §6 |
| Component → secrets (lateral) | `secrets` relationship via `SecretResourceSchema` (`component_schemas.py:120-128`); `AuthenticationMixin.get_authentication_secret` (`authentication_mixin.py:56`) raises `KeyError` when the named secret is gone |
| Component → resource pools (lateral, Pro) | CASCADE FK from `ResourcePoolSubjectPolicySchema` (`resource_pool_policy_schemas.py:82-89`): deleting a component deletes its pool policies |
| Stack → snapshots/builds (downstream) | `StackSchema.builds` and `.snapshots` relationships (`stack_schemas.py:140-143`) — a stack is referenced by historical pipeline builds and snapshots |
| Flavor → component (join, not FK) | `StackComponentSchema.flavor_schema` joins on `(flavor, type)` **strings**, not a foreign key (`component_schemas.py:100-105`); `sql_zen_store.py:4716` raises `IllegalOperationError` when deleting a flavor still in use — the only integrity guard |
| CLI surface generation | `register_all_stack_component_cli_commands` (`cli/stack_components.py:1622-1627`) loops `StackComponentType`, so **adding one enum member adds ~12 CLI commands**; `type.plural` (`enums.py:233-246`) also drives `import_module(f"zenml.{component_type.plural}")` in the `explain` command (`cli/stack_components.py:983`) and the docs URL (`flavor.py:232`) — a bad plural breaks all three |
| Stack deployment → connectors | `get_stack` (`stack_deployment.py:175-236`) matches on labels `zenml:provider` / `zenml:deployment` (`:220-224`) and requires `orchestrator.connector` to be set (`:228`) — a deployed stack is only discoverable if the wizard-created connector survived |
| Odd coupling | `SqlZenStore._create_stack_component` reaches into `get_stack_deployment_class` to warn about Skypilot regions for flavors `vm_gcp`/`vm_azure` (`sql_zen_store.py:3786-3800`), flagged in-place with `# TODO: this sooo does not belong here!` (`:3785`) |

## 7. Test coverage

| Path | Tests | Covers |
|---|---|---|
| `tests/unit/stack/test_stack.py` | 15 | Composition, `requirements`, validator dispatch, `requires_remote_server`, submission, run/step metadata, Docker builds |
| `tests/unit/stack/test_stack_component.py` | 8 | Config immutability, extras, secret-reference resolution and non-serialization |
| `tests/unit/stack/test_stack_validator.py` | 2 | `required_components` and `custom_validation_function` |
| `tests/unit/stack/test_utils.py` | 1 | `validate_stack_component_config` rejects extra keys |
| `tests/unit/test_flavor.py` | 8 | `validate_flavor_source`, docs URLs, `CustomFlavorImportError`, and `test_flavor_from_model_skips_implementation_class_validation:185` — which pins the `validate_component_classes=False` behaviour at `flavor.py:152` |
| `tests/unit/cli/test_stack_component_config_help.py` | 22 | The generated `--help` config-schema renderer |
| `tests/unit/models/{test_component_models,test_flavor_models}.py` | 1 + 4 | Model round-trips |
| `tests/integration/functional/cli/test_stack.py` / `test_stack_components.py` | 22 + 16 | CLI CRUD |
| `tests/integration/functional/zen_stores/test_zen_store.py:2663-3105` | 17 | Store-level CRUD, default-object guards, recursive delete, cross-user visibility (`:3069`) |

**Gaps, in order of exposure:**

1. **`stack_deployments/` has zero tests.** `find tests -iname "*stack_deployment*"` and
   `grep -rn "StackDeployment" tests/` both return nothing — 1,327 lines that generate
   CloudFormation/Terraform config and mint a 6-hour API token (`stack_deployment_endpoints.py:135`)
   are entirely untested.
2. **No test asserts that `register_integration_flavors` registers flavors for *uninstalled*
   integrations** — the §2.2 behaviour. (Dossier 0009 §8 records the same gap.) An accidental
   `check_installation()` gate would silently shrink every server's flavor table and pass CI.
3. **Neither skip flag is tested as a gate.** `ENV_ZENML_SKIP_IMAGE_BUILDER_DEFAULT` is *used* by
   `tests/unit/stack/conftest.py:33-35` to make fixtures work, never asserted; nothing anywhere
   references `ENV_ZENML_SKIP_STACK_VALIDATION`, so an inverted default would not be caught.
4. **`PATCH /flavors/sync`** — the one destructive, unguarded, server-wide endpoint in this area —
   has no test at any level.
