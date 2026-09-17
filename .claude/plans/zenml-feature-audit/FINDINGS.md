# FINDINGS — what OSS ZenML actually gives you

Measured against `cc064f100`. Every claim traces to a dossier in `findings/`, and every dossier
claim carries a `file:line`.

## The verdict in one sentence

**ZenML Pro is not an unlock; it is enforcement plus server-side execution.** The open-source build
ships nearly every *capability* and almost none of the *control*. The two things OSS genuinely
cannot do are enforce authorization and execute anything server-side. Everything else either works,
or is one environment variable away.

The naive expectation — that Pro adds features OSS lacks — is wrong in an important direction:
because every gate **fails open** in OSS, the open-source server is *less* restricted than a Pro
one. It has no usage limits, no permission checks, and (for user management) strictly more
endpoints.

## Reading the verdict column

The A/B/C/D tiers in `analysis/0001-gate-taxonomy.md` classify the **six pluggable subsystems**.
For whole feature areas a plainer axis is needed, because most areas are not gated by an
implementation source at all:

| Label | Meaning |
|---|---|
| **Pro-only** | The capability does not function in OSS. |
| **Opt-in** | A complete OSS implementation ships but is off by default. |
| **Open** | Ships and works, but the authorization/metering around it is absent in OSS. |
| **Complete** | Ships and works with no Pro coupling whatsoever. |

Some dossiers use "Tier B" loosely for *any* config-gated area (0008, 0009, 0011). Those are
**Complete** on this axis — no implementation source is involved.

## The table

| # | Area | Verdict | What OSS loses | How to close the gap |
|---|---|---|---|---|
| 1 | [RBAC](findings/0001-rbac.md) | **Pro-only** | All authorization. 9 fail-open returns; degrades to the binary `is_admin` column | Implement `RBACInterface`'s 4 methods; set `rbac_implementation_source` |
| 2 | [Resource Pools](findings/0002-resource-pools.md) | **Pro-only** | The entire feature — all 13 endpoints return HTTP 501 | Implement 15 abstract methods. A disabled 2,020-line test suite specifies the semantics |
| 3 | [User Management](findings/0003-user-management.md) | **Open** | *Nothing* — OSS has **more** endpoints than Pro | n/a. Pro's `EXTERNAL` scheme *removes* local user management |
| 4 | [Pipelines](findings/0004-pipelines.md) | **Split** | Server-side execution only; client-side is ungated | `workload_manager_implementation_source` — but only a dev shim ships |
| 5 | [Event Triggers](findings/0005-event-triggers.md) | **Pro-only (firing)** | Nothing fires. Define/persist/authenticate all work | Write a handler; set `event_handler_sources` (defaults to `[]`) |
| 6 | [Run Templates](findings/0006-run-templates.md) | **Pro-only (execution)** + deprecated | Execution. CRUD is unconditional | Use Pipeline Snapshots instead — do not build on this |
| 7 | [AI workflows](findings/0007-ai-workflows.md) | **Complete**, streaming **Opt-in** | Live event streaming only | `stream_broker_implementation_source` → `RedisStreamsBroker` |
| 8 | [Service Connectors](findings/0008-service-connectors.md) | **Complete** | Nothing. Implicit auth is off by default *as a security choice* | `ZENML_ENABLE_IMPLICIT_AUTH_METHODS` |
| 9 | [Integrations](findings/0009-integrations.md) | **Complete** | Nothing. Gated only by dependency presence | `zenml integration install` / pip extras |
| 10 | [Projects](findings/0010-projects.md) | **Open** | Enforcement + metering. Any user can delete any project | n/a — but note the dashboard, not the server, limits multi-project |
| 11 | [Artifact Management](findings/0011-artifact-management.md) | **Complete** | Nothing — unmetered on *both* editions | Five config knobs only |
| 12 | [Model Management](findings/0012-model-management.md) | **Open** | Enforcement + metering | n/a |
| 13 | [Stacks & Components](findings/0013-stacks-and-components.md) | **Open** | Enforcement — and here there is *no* `is_admin` fallback either | n/a |

## The five things worth acting on

1. **Server-side pipeline execution is the real Pro boundary.**
   `POST /pipeline_snapshots/{id}/runs`, `/run_templates/{id}/runs` and `/runs/{id}/replay` are
   *not registered at all* without `workload_manager_enabled`
   (`pipeline_snapshot_endpoints.py:351`, `run_templates_endpoints.py:232`, `runs_endpoints.py:703`).
   The only in-tree implementation, `InMemoryWorkloadManager`, is a shim: `build_and_push_image()`
   returns `""`, `get_logs()` returns `""`, and `run()` ignores the `image` argument and subprocesses
   in the API server's own interpreter. **Client-side runs are completely ungated** — if your
   workflow is "developers run pipelines from their machines", OSS costs you nothing here.

2. **Live streaming is genuinely free, and is the one high-value switch.**
   `RedisStreamsBroker` is 286 lines with zero stub returns, and it is the *only* Tier-B subsystem
   ZenML exposes in Helm (`values.yaml:66`, `_environment.tpl:251-252`). Set the source, add the
   `server-streaming` extra and a Redis.

3. **Triggers are a trap.** You can define schedules and webhooks, store them, authenticate
   webhook signatures, and compute the next fire time — and nothing will ever run. `next_occurrence`
   is written and filterable but read by no OSS code path; `utils/native_schedules.py:14` is
   literally docstringed `"""PRO native schedules utility functions."""`. Budget for an external
   scheduler, or a custom handler registered via `event_handler_sources`.

4. **Two loaders fail silently.** `initialize_workload_manager` (`utils.py:215-217`) and
   `initialize_resource_pool_store` (`:384-385`) log a warning and continue on a bad source, so a
   typo leaves you in OSS mode believing the feature is on. Streaming and the dispatcher raise.
   **Always assert the `*_enabled` property, never a clean startup.**

5. **Resource pools are scaffolding around a hole.** ~3,000 lines of models, schemas, routers, CLI
   and a migration ship; the engine does not. `resource_pool_implementation_source` is never
   assigned by anything — not the Pro override block, not Helm — so even a Pro server sets it by
   env var. `initialize_resource_pool_store()` is also called twice (`zen_server_api.py:182`, `:187`).

## One security consequence worth acting on

Stacks are the **worst-governed** surface in OSS. With RBAC off, every `verify_permission` call is a
no-op — and unlike users, secrets and service accounts, **no stack endpoint calls
`verify_admin_status_if_no_rbac`**, so there is not even the coarse `is_admin` fallback.

Concretely: `PATCH /flavors/sync` (`flavors_endpoints.py:190-191`) purges and rewrites the entire
server-wide flavor table, guarded only by `verify_permission(FLAVOR, UPDATE)`. On an OSS server that
is **callable by any authenticated principal, service accounts included**. Scope your API keys
accordingly, or front the server with a proxy that denies this route.

Related: `stack_deployments/` is 1,327 lines with **zero tests**, and it mints a 6-hour API token.

## The missing engine, specified

Resource pools are the one area where the absent implementation can be recovered rather than guessed.
[`specs/0001-resource-pool-engine.md`](specs/0001-resource-pool-engine.md) (`zenml-feature-audit/SPEC-01`)
extracts 46 cited requirements and a 31-test acceptance checklist from the disabled suite —
**3–5 engineer-weeks**, with concurrency (not allocator logic) as the dominant risk, since all 31 tests
are single-threaded against a schema explicitly built for contention.

Two blockers are recorded there rather than glossed: the suite **cannot run at HEAD even un-skipped**
(it references four model fields OSS lacks, three failing silently under `extra="ignore"`), and the
bounded/unbounded resource-key partition that two tests turn on **is defined nowhere in `src/zenml`**.
Method: [`analysis/0003-disabled-tests-as-specs.md`](analysis/0003-disabled-tests-as-specs.md).

## OSS is cut down from a superset, not built up

Three independent artefacts show OSS code referencing a **richer model surface than OSS itself
defines**. Individually each looks like an oddity; together they are a pattern, and they explain why
the scaffolding is so complete around the holes.

| Vestige | Evidence | Referenced elsewhere? |
|---|---|---|
| `attach_resource_pools` / `detach_resource_pools` | Excluded from an update loop at `zen_stores/schemas/component_schemas.py:219-220` | **No** — these two lines are the only occurrences in the entire tree |
| `update_trigger_snapshot_dispatch_state` | `sql_zen_store.py:8988` | **No callers**, and not on `ZenStoreInterface` — it exists for the Pro scheduler to call |
| Four resource-pool model fields | The disabled suite constructs them; OSS lacks all four | Three fail **silently** under `extra="ignore"` (`models/v2/base/base.py:45`) |

Add the contention machinery the resource-pool schema carries but nothing in OSS writes
(`claim_token`, `claim_expires_at`, a unique `resource_pool_allocation.request_id`, a mutable
`occupied` counter with a repair path), and the conclusion is that these features were **removed from
a working implementation**, not left half-built.

**Practical consequence:** grepping OSS for a symbol that "should" exist is not evidence it never did.
When implementing against one of these holes, the surrounding dead references are the best available
description of the missing contract.

## Things that surprised us

- **Service accounts can never be admins.** `is_admin=False` is hardcoded
  (`user_schemas.py:215`, `service_account.py:219`), so on an OSS server CI/CD can never manage
  identities or run secrets backup/restore — only an interactive human can. Enabling RBAC is what
  lifts it, because `zen_server/utils.py:810` then skips the check.
- **A checked-in, disabled specification.** `tests/unit/zen_stores/test_resource_request_pool_lifecycle.py`
  is 2,020 lines and 31 tests, killed at line 30 by `pytest.skip(..., allow_module_level=True)`. It
  names the exact allocator semantics Pro implements: preemption victim selection, reserved
  non-preemptible share, orphan reconciliation, capacity-change requeueing.
- **The "Pro" store class runs on every OSS server.** `RBACSqlZenStore` is selected whenever
  `ENV_ZENML_SERVER` is set (`base_zen_store.py:156-162`) with no edition test. Only its leaf checks
  no-op — which is why *implicit* model creation during a run is permission-checked even though no
  endpoint mentions it.
- **Every server registers all integration flavors.** `register_integration_flavors`
  (`flavor_registry.py:145-152`) has no `check_installation()` gate, unlike `activate_integrations`.
  Flavor *listing* is install-independent; only instantiation fails later.
- **Two branches run the other way.** Pro *loses* the implicit default project
  (`sql_zen_store.py:14114-14129`), and OSS multi-project is limited by the dashboard, not the
  server (`cli/utils.py:2760-2784`) — the only front-end gate in the audit.
- **Dead symbols.** `LATEST_MODEL_VERSION_PLACEHOLDER` (`constants.py:616`) has zero consumers.
  `update_trigger_snapshot_dispatch_state` (`sql_zen_store.py:8988`) has zero callers and is not on
  `ZenStoreInterface` — it exists for the Pro scheduler to call.
  `DEFAULT_ZENML_SERVER_FILE_DOWNLOAD_SIZE_LIMIT` (`constants.py:406`) is 2 GiB behind a `# 20 GB` comment.

## Scope note

**Resource Pools (#2) was not in the original brief.** It is included because it is the clearest
Pro-only case in the repo — entitlement-gated, RBAC-typed, fully scaffolded, engine absent. Drop it
if it is out of scope for you.

**Five routers still fall outside the thirteen areas** (the four stack routers now have dossier 0013).
They were checked so the gap is not silent — **all five are "Open"**: each uses the standard RBAC
wrappers, none carries an entitlement check.

| Router | RBAC calls | Entitlement calls |
|---|---|---|
| `logs_endpoints.py` | 13 | 0 |
| `code_repositories_endpoints.py` | 10 | 0 |
| `hook_invocations_endpoints.py` | 7 | 0 |
| `run_metadata_endpoints.py` | 4 | 0 |
| `tags_endpoints.py` | 4 | 0 |

Each is a sub-surface of an area already covered (pipelines, artifacts) rather than a feature in its
own right.

**Also not covered:** the dashboard (closed-source, not in this repo), ZenML Pro's own control
plane, and runtime behaviour — every finding here is static. See `README.md` for the verification
steps that would confirm the Opt-in claims against a running server.
