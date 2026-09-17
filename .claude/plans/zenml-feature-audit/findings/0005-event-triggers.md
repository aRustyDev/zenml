# 0005 — Event Triggers (schedules, platform events, webhooks)

Repo `cc064f100`. Every claim carries `file:line`.

---

## 0. History: the plugin system was REMOVED, not evolved

| Migration | Revision | What it did |
|---|---|---|
| `migrations/versions/479103df60b6_add_triggers.py` | `479103df60b6` | original `trigger` / `event_source` / `action` tables |
| `migrations/versions/25155145c545_separate_actions_and_triggers.py` | `25155145c545` | split actions out of triggers |
| `migrations/versions/97109a4a8d26_remove_triggers.py` | `97109a4a8d26`, down_rev `0.93.3` | **`op.drop_table("trigger_execution")` `:26`, `op.drop_table("trigger")` `:27`, `op.drop_table("event_source")` `:28`, `op.drop_table("action")` `:29`** |
| `migrations/versions/3855b5d051cd_add_v2_of_trigger_schemas.py` | `3855b5d051cd`, down_rev `97109a4a8d26` | rebuilt a single polymorphic `trigger` table (`:22-…`) with `type` and `flavor` columns (`:30-31`) |

**There are no "actions" and no "event sources" today.** One polymorphic `Trigger` with `TriggerType` ∈ {`schedule`, `platform_event`, `webhook`} (`enums.py:681-686`) and `TriggerFlavor` ∈ {`native schedule`, `platform event`, `webhook`} (`enums.py:689-694`). Later migrations: `b949c5d1bc0d_add_concurrency_trigger_column`, `7464581d8249_add_trigger_execution_info`, `a3f7c2e9b1d4_add_trigger_dispatch_visibility`, `c033e0f0f0d0_add_pipeline_run_triggered_by_index`.

---

## 1. Surface

| Layer | Path |
|---|---|
| Trigger registry | `src/zenml/triggers/registry.py` — type→model maps at `:28-31` (bodies) and `:33-44` (responses) |
| Models | `src/zenml/models/v2/core/triggers.py` (1271 ln): `ScheduleTrigger` + `next_occurrence` at `:706`, `:757`; `PlatformEventTrigger` `:972` with validator `:978-1000`; webhook trigger models |
| Webhook intake | `src/zenml/webhooks/{intake,events,handler}.py`, `src/zenml/webhooks/providers/{base,registry,custom,github,clickup,slack,types}.py` |
| Event fan-out | `src/zenml/dispatcher/{dispatcher,events,handler}.py` |
| Legacy schedules | `src/zenml/config/schedule.py` (137 ln, `class Schedule` `:31`) |
| Schedule math | `src/zenml/utils/native_schedules.py` (125 ln) |
| Trigger helpers | `src/zenml/utils/trigger_utils.py` (448 ln) |
| REST routers | `routers/trigger_endpoints.py` (466), `routers/webhook_endpoints.py` (423), `routers/schedule_endpoints.py` (210) |
| SQL schemas | `schemas/trigger_schemas.py` (400), `schemas/trigger_assoc.py` (141: `TriggerSnapshotSchema:31`, `TriggerExecutionSchema:96`), `schemas/webhook_schemas.py` (267), `schemas/schedule_schema.py` (271) |
| CLI | `cli/trigger.py` (959: groups `schedule`/`platform_event`/`webhook` at `:53,:58,:63`), `cli/webhook.py` (186) |
| Config | `config/server_config.py:446` `event_handler_sources`, `:447` `webhook_event_handler_sources`, both parsed by the validator at `:655-656` |

---

## 2. Tier verdict

**Tier A for everything that *fires*. Tier B-shaped for the extension point, but the OSS tree ships no handler.**

The area splits cleanly:

| Half | Status | Evidence |
|---|---|---|
| **Define, persist, authenticate, dispatch** | Fully implemented in OSS | Four webhook providers registered eagerly-on-demand (`webhooks/providers/registry.py:83-93`: Custom, GitHub, ClickUp, Slack); full CRUD routers; full CLI; signature verification per provider |
| **Act on a trigger — turn an event into a pipeline run** | **No OSS implementation exists** | see below |

### Evidence that nothing fires in OSS

1. **`EventDispatcher` fans out only to registered handlers** (`dispatcher/dispatcher.py:67-86`). Registration has exactly three sources:
   - `register_event_handlers()` — loads classes named in `server_config().event_handler_sources` (`zen_server/utils.py:1122-1151`). **Default `[]`** (`config/server_config.py:446`).
   - `register_webhook_event_handlers()` — same for `webhook_event_handler_sources` (`zen_server/utils.py:1154-1172`). **Default `[]`** (`config/server_config.py:447`).
   - One hard-wired handler: `StreamEndEventHandler` (`zen_server/utils.py:322-325`), registered only inside `initialize_streaming()`, and it only closes SSE streams (`zen_server/streaming/run_end_handler.py:33`, type check at `:57`).
2. **The only concrete `EventHandler` subclasses in `src/`** are `StreamEndEventHandler` (`zen_server/streaming/run_end_handler.py:33`) and the abstract adapter `WebhookEventHandler` (`webhooks/handler.py:22`). The remaining subclasses are test doubles (`tests/unit/webhooks/test_handlers.py:22,:33`; `tests/unit/zen_server/endpoints/test_webhook_endpoints.py:222`). **No handler in OSS starts a pipeline run.**
3. **`run_snapshot(..., trigger_id=...)`** — the parameter exists (`zen_server/pipeline_execution/utils.py:295`) and drives `zen_store().create_trigger_execution(...)` (`:465-469`), but **no OSS caller passes it**. All three callers (`routers/runs_endpoints.py:759`, `routers/pipeline_snapshot_endpoints.py:407`, `routers/run_templates_endpoints.py:317`) omit it.
4. **Native schedules never fire.** `next_occurrence` is *computed* (`models/v2/core/triggers.py:706`, `:757` via `utils/native_schedules.py:74`) and *stored* and *filtered on* (`models/v2/core/triggers.py:413,:476`), but no OSS code reads it to dispatch. The module's own docstring is `"""PRO native schedules utility functions."""` (`utils/native_schedules.py:14`).
5. **The dispatch-state write path has no OSS caller.** `SqlZenStore.update_trigger_snapshot_dispatch_state` (`zen_stores/sql_zen_store.py:8988`) is referenced **nowhere else in `src/` or `tests/`**, and is not on `ZenStoreInterface` — only the *read/clear* side is (`zen_stores/zen_store_interface.py:2051`, `rest_zen_store.py:2875`). It is an in-process hook for a Pro component holding the `SqlZenStore` directly.
6. Even if a handler existed, the run it would start requires the workload manager — see 0004 §2. In OSS that route does not exist.

### Platform events specifically

`PipelineRunStatusUpdate` is the only platform event actually emitted, from `sql_zen_store.py:13218-13226` — and the emit is itself guarded:

```python
dispatcher = EventDispatcher()
if dispatcher.has_handlers():            # :13219
    dispatcher.dispatch_event(PipelineRunStatusUpdate(...))   # :13221
```

With zero handlers registered the model is not even serialized. `PLATFORM_EVENT_REGISTRY` (`enums.py:776-780`) maps `PIPELINE`, `PIPELINE_RUN`, `PIPELINE_SNAPSHOT` to their event enums, and `RUN_COMPLETED`/`RUN_FAILED` are describable (`enums.py:770-773`) — so the UI can *offer* these triggers on an OSS server that will never act on them.

---

## 3. Gate inventory

| # | Mechanism | Where | Effect |
|---|---|---|---|
| 3 | Pluggable implementation source (list form) | `config/server_config.py:446` `event_handler_sources: list[str] = []`; loader `zen_server/utils.py:1126-1151`; called from lifespan `zen_server/zen_server_api.py:201` | Empty ⇒ platform-event triggers inert |
| 3 | Pluggable implementation source (list form) | `config/server_config.py:447` `webhook_event_handler_sources: list[str] = []`; loader `zen_server/utils.py:1157-1172`; lifespan `zen_server/zen_server_api.py:202` | Empty ⇒ authenticated webhook deliveries are accepted and **discarded** |
| 4 | Entitlement gate | `routers/trigger_endpoints.py:141` (create), `:256` (update), `:337` (attach-to-snapshot) — all `check_entitlement(feature=SCHEDULE_FEATURE)`, constant `constants.py:702` = `"schedule"` | **Tier C, fails open**: `feature_gate/endpoint_utils.py:30-31`. OSS may create unlimited triggers of any type |
| 4 | Entitlement gate — **absent where you'd expect it** | `routers/schedule_endpoints.py` has **no** `check_entitlement`; `routers/webhook_endpoints.py` has none either | Legacy schedules and webhook objects are never metered even on Pro |
| 3 | Pluggable implementation source | `config/server_config.py:346` `workload_manager_implementation_source` | Downstream: even a Pro handler needs this to actually run anything |
| 5 | Deployment-type gate | `config/server_config.py:820` `is_pro_server`; sources force-set in `get_server_config()` `:851-885` | Pro servers get the handler sources for free |
| 6 | Helm value | **none** — `event_handler_sources` and `webhook_event_handler_sources` do not appear in `helm/values.yaml` or `helm/templates/` | Env var only |
| 1 | Dependency presence | `croniter` imported unconditionally at `utils/native_schedules.py:19`; it is a hard dependency, not optional | No gate |
| 8 | Schema availability | tables created by `3855b5d051cd`; suppressible via `ENV_ZENML_DISABLE_DATABASE_MIGRATION` (`zen_stores/sql_zen_store.py:1551-1557`) | Same both tiers |
| 9 | Deprecation | `routers/schedule_endpoints.py:62` and `:100` mark the **workspace-scoped** `create_schedule`/`list_schedules` aliases `deprecated=True` | The legacy `Schedule` (`config/schedule.py:31`) remains the *orchestrator-native* schedule and is not deprecated as a concept |

### `event_handler_sources` — what "empty by default" actually disables

| Consequence | Line |
|---|---|
| `PipelineRunStatusUpdate` is never even constructed (short-circuited by `has_handlers()`) | `zen_stores/sql_zen_store.py:13219` |
| `PLATFORM_EVENT` triggers (`RUN_COMPLETED`, `RUN_FAILED`) never chain a downstream snapshot | no caller of `run_snapshot(trigger_id=…)` |
| Webhook deliveries: intake authenticates, records stats (`routers/webhook_endpoints.py:399-401`) and builds a `WebhookEvent` (`:404-412`), then hands it to `BackgroundTask(EventDispatcher().dispatch_event, event)` (`:413-415`) which fans out to **zero** handlers | `dispatcher/dispatcher.py:73-86` |
| Loader failures are logged, not fatal — a bad source string leaves the server silently handler-less | `zen_server/utils.py:1138-1141`, `:1148-1151`, `:1168-1172` |

**The webhook HTTP contract is deliberately decoupled from this.** `webhooks/AGENTS.md:29-35`: *"A successful 2XX intake response means only that ZenML accepted the trusted delivery… It does not confirm trigger matching, queuing, or snapshot execution."* On OSS, **every** webhook returns 2XX and nothing ever happens — and that is by design, not a bug.

---

## 4. OSS degradation

| Lost | Substitute |
|---|---|
| Cron/interval schedules managed by ZenML (`TriggerFlavor.NATIVE_SCHEDULE`) | **Orchestrator-native schedules**: the legacy `Schedule` model (`config/schedule.py:31`), passed at `pipeline.with_options(schedule=...)`. Requires `stack.orchestrator.config.is_schedulable` (`pipelines/pipeline_definition.py:984`), which the built-in Local/LocalDocker orchestrators do **not** set — so you need Airflow/Kubernetes/Vertex/SageMaker/etc. (12 flavors override it `True`). The scheduling then lives in *that* system, not in ZenML. |
| Run-completed / run-failed chaining (pipeline B after pipeline A) | Call the next step in-process, or use the orchestrator's own DAG |
| GitHub push → pipeline, Slack command → pipeline, ClickUp → pipeline | External CI: have GitHub Actions run the pipeline client-side |
| Trigger run-concurrency policy (`TriggerRunConcurrency` `enums.py:697-701`) | n/a — nothing runs |
| Trigger execution history (`TriggerExecutionSchema` `schemas/trigger_assoc.py:96`) | rows are never written |
| **Not lost:** webhook object CRUD, secret rotation (`routers/webhook_endpoints.py:260`), signature verification, raw-event inspection (`:188`), the whole trigger CRUD/CLI surface, and entitlement limits (fail-open) | — |

The degradation is unusually deceptive: the dashboard and CLI let you build the entire trigger graph, and webhooks return HTTP 200. Only the *effect* is missing.

---

## 5. Self-host path

This is Tier A in substance. Both hooks exist and are documented, but **no class in this repo satisfies either**.

```bash
# Platform events (PipelineRunStatusUpdate → your code)
ZENML_SERVER_EVENT_HANDLER_SOURCES="my_pkg.handlers.MyTriggerHandler"

# Webhook deliveries (authenticated WebhookEvent → your code)
ZENML_SERVER_WEBHOOK_EVENT_HANDLER_SOURCES="my_pkg.handlers.MyWebhookHandler"
```

Comma-separated; whitespace and empty entries are stripped by the validator (`config/server_config.py:655-656`, behaviour asserted in `tests/unit/config/test_server_configuration.py:61-70`).

Contract to implement:

| Base | Required | Line |
|---|---|---|
| `EventHandler` | `handle_event(self, event: Event) -> None` (abstract) | `dispatcher/handler.py:24-30` |
| `EventHandler` | `async classmethod create() -> EventHandler` — **must** be overridden; the base raises `NotImplementedError` (`:44-46`) and the docstring says so explicitly (`:36-38`) | `dispatcher/handler.py:32-46` |
| `WebhookEventHandler` | `handle_webhook_event` | `webhooks/handler.py:22` |

Your handler must then match the event against stored triggers, resolve the attached snapshot (`TriggerSnapshotSchema` `schemas/trigger_assoc.py:31`, with `run_configuration` and `dispatch_state` `:69`), call `run_snapshot(..., trigger_id=…, trigger_execution_info=…)` (`zen_server/pipeline_execution/utils.py:287,295-296`), and record dispatch state via `SqlZenStore.update_trigger_snapshot_dispatch_state` (`zen_stores/sql_zen_store.py:8988`). **And the server must also have a workload manager configured** (0004 §2) or `run_snapshot` cannot execute anything.

For native schedules you additionally need a ticker: nothing in OSS polls `next_occurrence`. You would write a scheduler that queries `TriggerFilter(next_occurrence=…)` (`models/v2/core/triggers.py:413`), fires, and rolls the field forward using `utils/native_schedules.py:74 calculate_first_occurrence` / `:53 next_occurrence_for_cron` / `:22 next_occurrence_for_interval` — those three functions are the only part of native scheduling that ships.

---

## 6. Blast radius

```
provider webhook POST
  → routers/webhook_endpoints.py  (route → _receive_webhook_event :328)
      pre_validate (:308) → body (:318) → run_in_threadpool (:319)
      → provider.authenticate / parse_delivery      (webhooks/providers/base.py)
      → zen_store().record_webhook_event            (:399)
      → BackgroundTask(EventDispatcher().dispatch_event, WebhookEvent)  (:413)
           └─ dispatcher/dispatcher.py:73-86 ── for each registered handler
                └─ [OSS: none]  /  [Pro: handler → run_snapshot(trigger_id=…)]

run status change
  sql_zen_store.py:13218  if dispatcher.has_handlers():
      → PipelineRunStatusUpdate → same fan-out
           └─ StreamEndEventHandler (streaming only, closes SSE)
```

**Hops that matter:**
- `zen_stores/sql_zen_store.py:18` imports `EventDispatcher` — the **store layer** is the platform-event emitter, so any change to run-status transitions is a trigger-semantics change.
- `routers/trigger_endpoints.py:135-139` and `:249-254` call `get_webhook_provider(...).validate_configuration(...)` at write time, so a provider's target-event schema is enforced on trigger creation, not only on delivery.
- `webhooks/providers/registry.py:83-93` registers built-ins lazily under a lock (`:84-86` idempotence guard); adding a provider is a 6-step cross-layer change (`webhooks/AGENTS.md:37-54`).
- `EventDispatcher` is a `SingletonMetaClass` (`dispatcher/dispatcher.py:27`) — process-wide, so handler registration is not per-request or per-tenant.
- Handler failures are swallowed per handler (`dispatcher/dispatcher.py:81-86`), so a broken Pro handler degrades silently exactly like an absent one.
- `TriggerSnapshotSchema.dispatch_state` (`schemas/trigger_assoc.py:69`, parsed at `:78-90`) is the only user-visible signal that dispatch is failing; `clear_trigger_dispatch_error` is on the interface (`zen_stores/zen_store_interface.py:2051`) and surfaced in the CLI (`cli/trigger.py:536-597`).

---

## 7. Test coverage

| Area | Tests |
|---|---|
| Webhook providers + handlers | `tests/unit/webhooks/test_providers.py`, `tests/unit/webhooks/test_handlers.py` |
| Webhook HTTP intake | `tests/unit/zen_server/endpoints/test_webhook_endpoints.py` |
| Schedule math | `tests/unit/utils/test_trigger_utilities.py` (covers all three `native_schedules` functions) |
| Legacy `Schedule` model | `tests/unit/pipelines/test_schedule.py` |
| Dispatch state policy | `tests/unit/zen_stores/test_trigger_dispatch_state_policy.py` |
| Webhook trigger CLI | `tests/unit/cli/test_webhook_trigger.py` |
| Handler-source parsing | `tests/unit/config/test_server_configuration.py:61-78` |

**Gaps:** there is no `tests/unit/triggers/` and no `tests/unit/dispatcher/`. `EventDispatcher` itself is exercised only indirectly (via `tests/unit/zen_server/streaming/` and the webhook handler tests). Nothing tests the end-to-end "event → pipeline run" path, because that path does not exist in OSS.
