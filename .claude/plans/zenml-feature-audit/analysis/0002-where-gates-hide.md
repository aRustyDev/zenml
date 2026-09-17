# 0002 — Where the gates hide: two enforcement layers, not one

Measured against `cc064f100`.

A router-level audit of authorization is **incomplete by construction** in this codebase. Permission
enforcement happens in two places, and the second is invisible to anyone reading
`zen_server/routers/`.

## Layer 1 — the router

Most endpoints enforce via the generic wrappers in `zen_server/rbac/endpoint_utils.py`
(`verify_permissions_and_create_entity:58`, `..._get_entity:200`, `..._list_entities:220`,
`..._update_entity:267`, `..._delete_entity:295`, `..._prune_entities:320`), or by calling
`verify_permission_for_model` directly.

**Four routers carry no router-level permission call at all.** Verified by scanning every file in
`src/zenml/zen_server/routers/` for `verify_permission|check_permissions|verify_permissions_and|batch_verify_permissions`:

| Router | Why it has none |
|---|---|
| `server_endpoints.py` | Uses the admin fallback instead — `if not server_config().rbac_enabled:` at `:207`, and `auth_scheme != AuthScheme.EXTERNAL` at `:224` |
| `tag_resource_endpoints.py` | **Enforced one layer down, in `RBACSqlZenStore`** — see below |
| `devices_endpoints.py` | OAuth devices are self-scoped to the authenticated user |
| `resource_requests_endpoints.py` | Genuinely ungated — calls `zen_store()` directly behind only `Security(authorize)` (`:64, :90, :109`), unlike the sibling pools/policies routers |

## Layer 2 — the store subclass

`zen_server/rbac/rbac_sql_zen_store.py` defines `RBACSqlZenStore(SqlZenStore)`, selected in
`zen_stores/base_zen_store.py:157-162` **whenever `ENV_ZENML_SERVER` is set** — i.e. always,
server-side. It injects permission checks into store operations that cannot be gated at the router
because the resource does not exist yet, or is created implicitly:

| Method | Check | Line |
|---|---|---|
| `batch_create_tag_resource` (`:50`) | `batch_verify_permissions_for_schemas` | `:65` |
| `batch_delete_tag_resource` (`:75`) | `batch_verify_permissions_for_schemas` | `:87` |
| `_get_or_create_model` (`:96`) | `verify_permission` (CREATE) | `:115` |
| `_get_or_create_model` | `verify_permission_for_model` (READ) | `:148` |
| `_get_model_version` (`:152`) | `verify_permission_for_model` (READ) | `:175` |
| `_get_or_create_model_version` (`:178`) | `verify_permission` (CREATE) | `:201` |

This is why **implicit ZenML Model creation during a pipeline run is permission-checked** even
though no endpoint mentions it.

## Consequences for this audit

1. A dossier's gate inventory must check **both** layers. "No check in the router" is not evidence
   of an ungated resource.
2. Both layers collapse to no-ops together — every one of these calls routes through `rbac/utils.py`,
   which early-returns when `rbac_enabled` is false (`:75, 110, 243, 272, 316, 381, 812, 839, 860`).
   So in OSS the second layer is as absent as the first.
3. `ResourceType` → SQLModel schema resolution happens in one dict,
   `_get_resource_type_schema_mapping()` (`rbac/utils.py:692`). Adding a resource type without
   touching that dict yields a resource that cannot be permission-checked.

## Entitlement call density, for cross-reference

Routers containing `check_entitlement` / `report_usage` / `report_decrement`, by count:

| Router | Calls |
|---|---|
| `trigger_endpoints.py` | 4 |
| `resource_pools_endpoints.py` | 3 |
| `resource_pool_subject_policies_endpoints.py` | 3 |
| `models_endpoints.py` | 2 |
| `pipelines_endpoints.py` | 2 |
| `projects_endpoints.py` | 2 |
| `pipeline_snapshot_endpoints.py` | 2 |
| `runs_endpoints.py` | 2 |
| `run_templates_endpoints.py` | 2 |

Nine routers out of 40. Everything else is unmetered even on Pro.
