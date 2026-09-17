# 0010 — Projects

Repo @ `cc064f100`. Source root `src/zenml/`. Framework reference: the six nullable dotted-path
gates on `ServerConfiguration` (`src/zenml/config/server_config.py:343-349`);
`DEFAULT_REPORTABLE_RESOURCES = ["project","pipeline","pipeline_run","model"]` (`constants.py:465`).

> **Vocabulary.** "Workspace" means two unrelated things: (a) the pre-rename name for *Project*,
> still serving a fully deprecated REST router; (b) a **ZenML Pro workspace**, i.e. a deployed
> server (`ServerProConfiguration.workspace_id/workspace_name`, `server_config.py:862,871-880`).
> Only (a) is this dossier's subject; (b) appears only where the two collide.

## 1. Surface

| Layer | Path |
|---|---|
| REST — current | `src/zenml/zen_server/routers/projects_endpoints.py` — `router:67` (prefix `API + VERSION_1 + PROJECTS`, `constants.py:513`) |
| REST — legacy | same file, `workspace_router:61` (prefix `WORKSPACES`, `constants.py:563`), **every** mount flagged `deprecated=True` at `:78,115,144,177,210,240` |
| Endpoints | `list_projects:85`, `create_project:122`, `get_project:151`, `update_project:184`, `delete_project:217`, `get_project_statistics:247` |
| Domain model | `src/zenml/models/v2/core/project.py` — `ProjectRequest:36` (`name:39`, `display_name:47`, name-derivation validator `:75-90`), `ProjectUpdate:98`, `ProjectResponseBody:129`, `ProjectResponse:156` |
| Scoping mixins | `src/zenml/models/v2/base/scoped.py` — `ProjectScopedRequest:89`, `ProjectScopedResponseBody:315`, `ProjectScopedResponseMetadata:321`, `ProjectScopedResponseResources:325`, `ProjectScopedResponse:338`, `ProjectScopedFilter:393`, `apply_filter:413`, scope predicate `:451` |
| SQL schema | `src/zenml/zen_stores/schemas/project_schemas.py` — `ProjectSchema:57` (`NamedSchema`, `table=True`), `project_metadata:70`, **18 cascade relationships** `:80-149` |
| Store | `src/zenml/zen_stores/sql_zen_store.py` — `_set_filter_project_id:14481` (**22 call sites**), `_default_project_enabled:14114`, project CRUD `~:13998-14111` |
| Default-project name | `base_zen_store.py:448-454` — `os.getenv(ENV_ZENML_DEFAULT_PROJECT_NAME, DEFAULT_PROJECT_NAME)` (`constants.py:221`, `:359`) |
| CLI | `src/zenml/cli/project.py` (228 lines) — `project:35`, `list_projects:44`, `register_project:107` |
| RBAC | `src/zenml/zen_server/rbac/models.py` — `ResourceType.PROJECT:77`, `is_project_scoped:83` |
| Migrations | `3944116bbd56_rename_project_to_workspace.py` (rev `3944116bbd56`, down `0.32.1`) → … → `12eff0206201_rename_workspace_to_project.py` (rev `12eff0206201`, down `cbc6acd71f92`); plus `2e695a26fe7a_add_user_default_workspace.py`, `1f1105204aaf_add_user_default_project.py`, `3b1776345020_remove_workspace_from_globals.py`, `41b28cae31ce_make_artifacts_workspace_scoped.py`, `7834208cc3f6_artifact_project_scoping.py`, `cbc6acd71f92_add_workspace_display_name.py`, `b53125d4536d_add_project_metadata.py` |

The rename was made **twice, in opposite directions**: project→workspace at `0.32.1`, then
workspace→project many revisions later. `2e695a26fe7a_add_user_default_workspace.py` and
`1f1105204aaf_add_user_default_project.py` are the same round trip for the user's default.

## 2. Tier verdict — **Tier C**, with a Pro-only *behaviour inversion*

**Tier C.** Projects are fully implemented in OSS: `ProjectSchema` (`project_schemas.py:57`),
the `/projects` router, the CLI, and 22 `_set_filter_project_id` call sites in
`sql_zen_store.py` all ship and work. What OSS loses is **enforcement**, not capability:

- Every project endpoint routes through `verify_permissions_and_*`
  (`projects_endpoints.py:103,134,166,198,226,260`); in OSS each of those bottoms out in an
  early return (`rbac/utils.py:75,110,243,272,316,381`) — so any authenticated user can create,
  read, rename and **delete** any project.
- `ResourceType.PROJECT` is explicitly **not** project-scoped (`rbac/models.py:99` in the
  exclusion list of `is_project_scoped:83`) — projects are global resources.
- `PROJECT` **is** in `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`), and
  `report_decrement(ResourceType.PROJECT, ...)` is called on delete
  (`projects_endpoints.py:233`). That call returns immediately in OSS
  (`feature_gate/endpoint_utils.py:54`), and `check_entitlement` on create
  (`rbac/endpoint_utils.py:102`, via `verify_permissions_and_create_entity`) returns at
  `feature_gate/endpoint_utils.py:30`. **OSS project creation is unmetered; Pro is metered.**
- `projects_endpoints.py` never calls `verify_admin_status_if_no_rbac` (`zen_server/utils.py:793`),
  so the OSS admin fallback does **not** apply to projects.

No Pro-only project class exists: `grep`ing the router and store finds no
`*_implementation_source` reference and no `is_pro_server` branch in `projects_endpoints.py`.

**The inversion (most surprising finding in this area).** The multi-project story is *worse* on
Pro in one specific way and *worse* on OSS in another:

| | OSS | Pro |
|---|---|---|
| `_default_project_enabled` (`sql_zen_store.py:14114-14129`) | **True** — a `default` project is auto-created (`:1647`, `:14101-14111`) and used as an implicit filter fallback (`:14513-14522`) | **False** — gated on `ServerConfiguration.get_server_config().is_pro_server` at `sql_zen_store.py:14126`. No implicit default; unscoped filters raise `ValueError("Project scope missing from the filter")` (`:14524`) |
| Dashboard | **Only the default project is viewable.** `warn_if_project_not_visible_on_oss` (`cli/utils.py:2760-2784`) emits: pipelines in a non-default project "will run, but you won't be able to view them in the OSS dashboard" | All projects |

So OSS *can* create and use arbitrary projects through the API/CLI — the SQL scoping works
identically — but the bundled dashboard renders only `default`. That is a **front-end** gate
with no server-side counterpart, and it is the only place in this audit where a capability is
withheld by the dashboard rather than by a `ServerConfiguration` field.

## 3. Gate inventory

| # | Mechanism | Location | Effect |
|---|---|---|---|
| **4** | **Entitlement gate** | `check_entitlement` at `rbac/endpoint_utils.py:102` (create path) / `:188`; `report_usage:107,195`; `report_decrement(ResourceType.PROJECT)` at `projects_endpoints.py:233`. All short-circuit on `feature_gate_enabled` (`server_config.py:704`) at `feature_gate/endpoint_utils.py:30,42,54`. `PROJECT ∈ DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`), set only for Pro (`server_config.py:869`) | Pro: project count metered, over-limit → `SubscriptionUpgradeRequiredError` (`exceptions.py:163`) → **HTTP 402** (`zen_server/exceptions.py:79`). OSS: unlimited |
| **5** | **Deployment-type gate** | `_default_project_enabled` (`sql_zen_store.py:14114`) reads `ENV_ZENML_SERVER` then `ServerConfiguration.get_server_config().is_pro_server` (`server_config.py:820`) at `:14123-14127` | **The only in-tree `is_pro_server` branch outside `server_config.py`/`cli/utils.py`.** Pro suppresses the implicit default project |
| 5 | Deployment-type gate (client side) | `warn_if_project_not_visible_on_oss` (`cli/utils.py:2760`) → `client.zen_store.get_store_info().is_pro_server()` at `:2776`; called from `register_project`, `set_project`, `describe_project` | Warning only; no behaviour change |
| 3 | Pluggable implementation source | `rbac_implementation_source` (`server_config.py:343`) → `rbac_enabled` (`:695`). Concrete Pro class `ZenMLCloudRBAC` (`zen_server/rbac/zenml_cloud_rbac.py:32`) — **ships in the OSS wheel** but is only wired on Pro (`server_config.py:865-867`) | Governs whether the six `verify_permissions_and_*` calls do anything |
| 2 | Env-var boolean | `ENV_ZENML_DEFAULT_PROJECT_NAME` (`constants.py:221`, default `DEFAULT_PROJECT_NAME = "default"` at `:359`) read in `base_zen_store.py:454` | Renames, does not disable, the implicit project |
| 8 | Schema availability | The rename pair `3944116bbd56` → `12eff0206201` plus `41b28cae31ce`/`7834208cc3f6` (artifact project-scoping) and `b53125d4536d` (project metadata); all skippable via `ZENML_DISABLE_DATABASE_MIGRATION` (`sql_zen_store.py:1551-1557`) | A half-migrated DB serves one naming and not the other |
| **9** | **Deprecation** | `workspace_router` (`projects_endpoints.py:61`) with `deprecated=True` on all six mounts (`:78,115,144,177,210,240`), plus the comment "kept for backwards compatibility only; to be removed after the migration" (`:74`) | Legacy `/workspaces` paths still fully functional and correctly marked deprecated. The same `workspace_router` object is imported and re-mounted by four other routers, each also flagging `deprecated=True`: `service_connectors_endpoints.py:91,124,190`, `models_endpoints.py:72`, `model_versions_endpoints.py:89`, `service_endpoints.py:62`. **Deprecation here is consistent — verified, not assumed** |
| 1 | Dependency presence | not applicable | — |
| 6 | Helm value | none project-specific | — |
| 7 | Three-state cascade | none | — |

## 4. Which resources are NOT project-scoped

`ResourceType.is_project_scoped()` (`rbac/models.py:83-102`) returns `False` — i.e. **global** —
for exactly these 10:

`FLAVOR` (`:90`), `SECRET` (`:91`), `SERVICE_CONNECTOR` (`:92`), `STACK` (`:93`),
`STACK_COMPONENT` (`:94`), `RESOURCE_POOL` (`:95`), `RESOURCE_POOL_SUBJECT_POLICY` (`:96`),
`TAG` (`:97`), `SERVICE_ACCOUNT` (`:98`), `PROJECT` itself (`:99`). (`USER` is commented out at
`:101` — "Deactivated for now".)

Everything else is project-scoped: `ARTIFACT`, `ARTIFACT_VERSION`, `CODE_REPOSITORY`, `MODEL`,
`MODEL_VERSION`, `PIPELINE`, `PIPELINE_RUN`, `PIPELINE_SNAPSHOT`, `PIPELINE_BUILD`,
`DEPLOYMENT`, `SCHEDULE`, `RUN_TEMPLATE`, `SERVICE`, `RUN_METADATA`, `TRIGGER`, `WEBHOOK`
(`rbac/models.py:53-81`).

`Resource.model_validate` enforces the invariant both ways: `project_id` **must** be set for a
project-scoped type (`rbac/models.py:173-177`) and **must not** be set for a global one
(`:179-183`).

> Wire-value trap: `ResourceType.PIPELINE_SNAPSHOT = "pipeline_deployment"`
> (`rbac/models.py:61-62`, with the comment "We keep this name for backwards compatibility") —
> the symbol and the serialized value differ, so RBAC policy documents and the enum do not grep
> the same.

The **18 cascade relationships** on `ProjectSchema` (`project_schemas.py:80-149`) — pipelines,
schedules, runs, step_runs, builds, artifact_versions, run_metadata, snapshots,
code_repositories, services, models, model_versions, deployments, visualizations, triggers,
webhooks (and two more) — are the blast surface of `delete_project`.

## 5. OSS degradation

| Capability | OSS behaviour | Substitute |
|---|---|---|
| Project-level authorization | All six endpoint guards fail open (`rbac/utils.py:75,110,243,272,316,381`) | None — `verify_admin_status_if_no_rbac` (`zen_server/utils.py:793`) is **not** called from `projects_endpoints.py`, so even the admin fallback is absent |
| Multi-project dashboard | Only `default` is viewable (`cli/utils.py:2778-2783`) | CLI + SDK + REST work on any project; the warning explicitly points at zenml.io/pro |
| Project quotas | Unmetered (`feature_gate/endpoint_utils.py:30,42,54`) | — (a strict gain for OSS) |
| Cross-project isolation | SQL-level scoping is identical to Pro — `ProjectScopedFilter.apply_filter` (`scoped.py:413`) raises if the scope is missing (`:442`) or non-UUID (`:445-448`) and always appends `table.project_id == self.project` (`:451`) | Isolation of *data* holds; isolation of *users* does not |
| Implicit default project | **Present** in OSS (`sql_zen_store.py:14114`), absent on Pro | Pro clients must always pass an explicit project |

## 6. Self-host path

Tier C — nothing to enable. To get real project-level authorization one must supply an
`RBACInterface` implementation (`zen_server/rbac/rbac_interface.py`) and point
`ZENML_SERVER_RBAC_IMPLEMENTATION_SOURCE` (via the `ZENML_SERVER_` prefix, `constants.py:274`)
at it; `rbac_enabled` (`server_config.py:695`) then flips and all six guards become live. The
methods to implement are those `ZenMLCloudRBAC` provides: `check_permissions:39`,
`list_allowed_resource_ids:81`, `update_resource_membership:124`, `delete_resources:152`
(`zen_server/rbac/zenml_cloud_rbac.py`).

The dashboard's single-project limitation cannot be lifted from the server: no
`ServerConfiguration` field or env var controls it, and the check lives client-side in
`cli/utils.py:2776`.

## 7. Blast radius

| Direction | Target |
|---|---|
| `_set_filter_project_id` | **22 call sites** in `sql_zen_store.py` (`:2853, 2997, 3390, 3661, 4997, 5163, 5507, 5796, 6247, 7434, 7633, 7828, 8363, 8768, 9311, 12515, 12636, 14089, 14154, 14825, 15385` + the definition at `:14481`) — every project-scoped `list_*` method. A change to the resolution order (explicit arg → filter → default, `:14501-14525`) changes **every** list endpoint at once |
| `ProjectScopedFilter.apply_filter` | Inherited by every project-scoped filter model; the `getattr(table, "project_id")` at `scoped.py:451` means any schema missing that column raises `AttributeError` at query time, not import time |
| `delete_project` | 18 cascade relationships (`project_schemas.py:80-149`) + `report_decrement` (`projects_endpoints.py:233`) |
| Dual router | `workspace_router` is defined in `projects_endpoints.py:61` and **imported by at least four other routers** (`service_connectors_endpoints.py:62`, `models_endpoints.py:44`, `model_versions_endpoints.py:60`, `service_endpoints.py:39`) — removing it is a cross-module change, not a local one |
| Store selection | `RBACSqlZenStore` (`zen_server/rbac/rbac_sql_zen_store.py:47`) is chosen whenever `ENV_ZENML_SERVER` is set (`base_zen_store.py:157-162`) — **including OSS servers**. Its overrides call `verify_permission`/`check_entitlement` which then fail open. So the Pro code path executes on OSS; only the leaf checks no-op |
| Client | `Client.active_project` (used at `cli/project.py:66`) resolves through the user's `default_project_id` (`models/v2/core/user.py:214,293,469`) |

## 8. Test coverage

| Path | Covers |
|---|---|
| `tests/unit/models/test_project_models.py` | `ProjectRequest`/`ProjectResponse` validation, incl. the name-derivation validator (`project.py:75-90`) |
| `tests/unit/models/test_filter_models.py` | Filter-model machinery including scoped filters |
| `tests/integration/functional/zen_stores/test_zen_store.py` | Project CRUD and scoping through the store |
| `tests/unit/zen_server/` | No projects-specific test file (`ls` shows `test_auth.py`, `test_rate_limit.py`, `test_tag_resource_rbac.py`, … — no `test_projects*`) |

**Gaps:** `_default_project_enabled` (`sql_zen_store.py:14114`) — the one `is_pro_server` branch
in the store — has no test; there is no test asserting the deprecated `/workspaces` router still
answers; and `tests/unit/zen_server/` has no coverage of the six project endpoints (only
`tests/unit/zen_server/endpoints/test_webhook_endpoints.py` exists).
