# 0006 — Run Templates (DEPRECATED)

Repo `cc064f100`. Every claim carries `file:line`.

> **Verdict up front: do not use Run Templates in new work.** They are deprecated at every
> layer — SDK, CLI, REST — and superseded by **Pipeline Snapshots**, which carry the same
> parameterization machinery via the *same* helper module. Use `pipeline.create_snapshot(...)`
> / `zenml pipeline snapshot create` / `POST /pipeline_snapshots`.

---

## 1. Surface

| Layer | Path | Note |
|---|---|---|
| Model | `src/zenml/models/v2/core/run_template.py` (543 ln) | `RunTemplateRequest:70` (`source_snapshot_id: UUID` `:82`), `RunTemplateUpdate:98`, `RunTemplateResponseBody:126` (`runnable:129`), `…Metadata:138`, `…Resources:156` (`source_snapshot:159`), `RunTemplateResponse:187`, `RunTemplateFilter:335` |
| SQL schema | `src/zenml/zen_stores/schemas/run_template_schemas.py` (376 ln) | `RunTemplateSchema:50`, `hidden:62`, FK `source_snapshot_id:92` |
| Template helpers | `src/zenml/zen_stores/template_utils.py` (416 ln) | `validate_snapshot_is_templatable:45`, `generate_config_template:95`, `generate_config_schema:146` |
| REST router | `src/zenml/zen_server/routers/run_templates_endpoints.py` (324 ln) | router itself `deprecated=True` at `:72` |
| Client SDK | `src/zenml/client.py` | `create_run_template:4003`, `get_run_template:4032`, `list_run_templates:4060`, `update_run_template:4131`, `delete_run_template:4179` |
| Pipeline SDK | `src/zenml/pipelines/pipeline_definition.py:1728` | `create_run_template` — docstring literally begins `"DEPRECATED:"` (`:1731`) |
| CLI | `src/zenml/cli/pipeline.py:547` | `create_run_template` — `"""DEPRECATED: Create a run template for a pipeline."""` (`:553`) |
| Migrations | `migrations/versions/7d1919bb1ef0_add_run_templates.py` (rev `7d1919bb1ef0`, 2024-07-22), `6611d4bcc95b_add_hidden_option_for_templates.py` (rev `6611d4bcc95b`, down_rev `0.80.1`), `76a7b9451ccd_add_build_template_deployment_id.py` | tables still present |

---

## 2. Tier verdict

**Tier A in effect on OSS — but for the 0004 reason, not for a template-specific one.**

- CRUD (`create`/`list`/`get`/`update`/`delete`) is **unconditional OSS** — `routers/run_templates_endpoints.py:76-231` sits outside any guard.
- **Execution** is Tier A: `POST /run_templates/{id}/runs` is registered only under
  `if server_config().workload_manager_enabled:` (`routers/run_templates_endpoints.py:232`),
  and `workload_manager_implementation_source` defaults to `None`
  (`config/server_config.py:346`, property `:713-719`). The only in-tree implementation is the
  stub `InMemoryWorkloadManager` (`zen_server/pipeline_execution/in_memory_workload_manager.py:30`),
  referenced nowhere else in the repo. See 0004 §2 for the full trace.
- `SqlZenStore.run_template` / `run_snapshot` are `NoReturn` stubs (`zen_stores/sql_zen_store.py:5640,5653-5656`).

So on default OSS a run template is a **catalogued, parameterizable, unexecutable** object — the same status as a snapshot.

---

## 3. Gate inventory

| # | Mechanism | Where | Effect |
|---|---|---|---|
| 9 | **Deprecation** (the defining gate here) | Router-level `deprecated=True` `routers/run_templates_endpoints.py:72`; endpoint-level `:85` (create alias), `:122` (list alias). SDK warning `pipelines/pipeline_definition.py:1742-1746`. Client warning `client.py:3355-3357` (`"Triggering a run template is deprecated. Use Client().trigger_pipeline(snapshot_id=...) instead."`). CLI warning `cli/pipeline.py:562-566` (`"...Please use zenml pipeline snapshot create instead."`) | OpenAPI marks the whole tag deprecated |
| 3 | Pluggable implementation source | `routers/run_templates_endpoints.py:232` gate on `workload_manager_enabled`; field `config/server_config.py:346` | Removes `POST /{id}/runs` entirely on OSS |
| 4 | Entitlement gate | `routers/run_templates_endpoints.py:290` `check_entitlement(feature=RUN_TEMPLATE_TRIGGERS_FEATURE_NAME)`; constant `constants.py:466` = `"template_run"` | **Tier C, fails open** (`feature_gate/endpoint_utils.py:30-31`). Unreachable on OSS anyway — it sits inside the `workload_manager_enabled` block |
| 8 | Schema availability | `7d1919bb1ef0`, `6611d4bcc95b`; skippable via `ENV_ZENML_DISABLE_DATABASE_MIGRATION` (`zen_stores/sql_zen_store.py:1551-1557`) | Same both tiers |
| — | Templatability precondition | `zen_stores/sql_zen_store.py:6180` calls `template_utils.validate_snapshot_is_templatable(snapshot)` (`template_utils.py:45`) at creation | Not a tier gate; a data precondition |

---

## 4. OSS degradation

Nothing is lost *relative to snapshots* — both are equally unexecutable server-side on default OSS (0004 §4). What you lose by choosing templates over snapshots is forward compatibility.

The successor has **full parity on the only thing templates uniquely offered** — a generated config template and JSON schema for the run form:

| Function | Used by RunTemplate | Used by PipelineSnapshot |
|---|---|---|
| `template_utils.generate_config_template` (`template_utils.py:95`) | `schemas/run_template_schemas.py:314` | `schemas/pipeline_snapshot_schemas.py:578` |
| `template_utils.generate_config_schema` (`template_utils.py:146`) | `schemas/run_template_schemas.py:319` | `schemas/pipeline_snapshot_schemas.py:583` |

Same module, same call, guarded by the same `include_config_schema and build and build.stack_id` condition. A `RunTemplate` is now a thin named pointer at a snapshot (`source_snapshot_id`, `models/v2/core/run_template.py:82`) plus a `hidden` flag (`schemas/run_template_schemas.py:62`).

`pipeline.create_run_template(...)` itself now creates a snapshot first and wraps it
(`pipelines/pipeline_definition.py:1748-1753`) — the template is strictly derived.

---

## 5. Self-host path

Not applicable as a distinct capability. To make run templates *executable* on a self-hosted server you need exactly what 0004 §5 describes — a `WorkloadManagerInterface` implementation set on `ZENML_SERVER_WORKLOAD_MANAGER_IMPLEMENTATION_SOURCE`. There is no template-specific switch.

**Recommendation for new work:** target `PipelineSnapshotResponse` and `POST /pipeline_snapshots/{id}/runs` (`routers/pipeline_snapshot_endpoints.py:351-411`). Use `Client().trigger_pipeline(snapshot_id=...)`, never `template_id=` — the latter warns and routes to the deprecated store method (`client.py:3353-3366`).

---

## 6. Blast radius

```
Client.create_run_template :4003 ──► zen_store.create_run_template
   └─ SqlZenStore :6180 ── template_utils.validate_snapshot_is_templatable (template_utils.py:45)

RunTemplateSchema.to_model (run_template_schemas.py:~280)
   ├─ template_utils.generate_config_template :314
   └─ template_utils.generate_config_schema   :319      ◄── shared with snapshots

POST /run_templates/{id}/runs  (:234, gated :232)
   → check_entitlement :290 → run_snapshot(snapshot=template.source_snapshot, template_id=…) :317
        └─ pipeline_execution/utils.py:287 … → workload_manager()   [absent on OSS]
```

- **`template_utils` is shared, not template-owned.** Changing `generate_config_schema` changes the snapshot run-form too. `_replace_step_parameter_definitions` (`template_utils.py:340`) is the fiddly part.
- `RunTemplateFilter` (`models/v2/core/run_template.py:335`) joins through `RunTemplateSchema.source_snapshot_id` at six places (`:437,458,477,496,515,533`) to filter by pipeline/stack/build — a template query is really a snapshot query.
- `76a7b9451ccd_add_build_template_deployment_id.py` ties builds to templates; `PipelineBuildResponse` carries the link.
- Deleting a template does **not** delete its source snapshot (`routers/run_templates_endpoints.py:216-230` delegates to `verify_permissions_and_delete_entity` on the template only).

---

## 7. Test coverage

| Path | Covers |
|---|---|
| `tests/unit/zen_stores/test_template_utils.py` | `generate_config_template` / `generate_config_schema` / step-def renaming — i.e. the shared machinery |

There is **no** `tests/unit/.../test_run_template*.py`. The model, schema, and router have no dedicated unit tests; coverage rides entirely on `template_utils`, which snapshots also depend on. Consistent with a deprecated surface in maintenance mode.
