# zenml-feature-audit/SPEC-01 — Resource Pool Allocation Engine

> Always cite this document as **`zenml-feature-audit/SPEC-01`**, never as a bare "SPEC 01" —
> the numbering is local to the `zenml-feature-audit` plan tree and collides with other trees.

## 1. Provenance & status

| Field | Value |
|---|---|
| Status | **ASPIRATIONAL — not implemented.** No concrete engine exists in this repo. |
| Derived from | `tests/unit/zen_stores/test_resource_request_pool_lifecycle.py` — 2,020 lines, 31 test functions, killed at `:30` by `pytest.skip("Resource pool lifecycle tests are disabled.", allow_module_level=True)` |
| Measured at | commit `cc064f100` |
| Acceptance criteria | The 31 tests themselves. **Un-skipping `:30` and having all 31 pass is the definition of done.** |
| Tense | Requirements are written as MUST/SHOULD for work *not yet done*. |

**What is actually shipped in OSS.** ~3,000 lines of scaffolding around an absent engine:
models (`src/zenml/models/v2/core/resource_pool.py` 256 L, `resource_pool_subject_policy.py` 234 L,
`resource_request.py` 304 L), SQL schemas (`src/zenml/zen_stores/schemas/resource_pool_schemas.py` 362 L,
`resource_pool_policy_schemas.py` 259 L, `resource_request_schemas.py` 270 L), three routers
(`src/zenml/zen_server/routers/resource_pools_endpoints.py` 187 L, `resource_requests_endpoints.py` 109 L,
`resource_pool_subject_policies_endpoints.py`), a CLI (`src/zenml/cli/resource_pool.py` 608 L) and an
alembic migration (`src/zenml/zen_stores/migrations/versions/b0ba2c3800e3_add_resource_pools.py` 348 L).

**The engine is absent.** `src/zenml/zen_stores/resource_pools/` contains only `__init__.py` and
`store_interface.py`; the latter defines two ABCs — `ResourcePoolsStoreInterface` (`store_interface.py:39`)
and `ResourcePoolsSQLStoreInterface` (`:217`) — carrying 15 `@abstractmethod`s and **no concrete subclass
anywhere in the repo**. The implementation is loaded by source string at runtime
(`src/zenml/zen_server/utils.py:368-388`, `initialize_resource_pool_store`, which only logs a warning when
the source cannot be loaded). Without it, `SqlZenStore.resource_pools` raises `NotImplementedError`
(`src/zenml/zen_stores/sql_zen_store.py:1243-1247`), which the server maps to **HTTP 501**
(`src/zenml/zen_server/exceptions.py:95`).

**The tests do not run at HEAD even with `:30` removed.** Section 6 lists four model-surface gaps; the
hard blocker is `ResourcePoolSubjectPolicyRequest.pool_id` (`resource_pool_subject_policy.py:50-52`),
required with no default, which the test helper `_create_pool:118-140` never supplies.

---

## 2. Domain model

### 2.1 Entities

| Entity | Schema | Model | Key fields |
|---|---|---|---|
| `ResourcePool` | `resource_pool_schemas.py:176` | `resource_pool.py:81` (request), `:165` (response) | `name` (globally unique, `:181-185`), `description`, `resources`, `policies` (ordered `desc(priority)`, `:216`) |
| `ResourcePoolResource` | `resource_pool_schemas.py:339` | — | `(pool_id, key)` unique `:344-348`; `key: str`, `total: int`, `occupied: int = 0` (`:360-362`) |
| `ResourceRequest` | `resource_request_schemas.py` | `resource_request.py:60` (request), `:132` (response) | `component_id`, `step_run_id`, `requested_resources`, `preemptible`, `status`, `status_reason` (`:131-133`), `preemption_initiated_by_id` (self-FK, `:104-109`) |
| `ResourcePoolSubjectPolicy` | `resource_pool_policy_schemas.py` | `resource_pool_subject_policy.py:44` | `(pool_id, component_id)`, `priority: int` (higher = preferred, `:53-56`), `reserved: Dict[str,int]`, `limit: Dict[str,int]` |
| `ResourcePoolQueue` (join) | `resource_pool_schemas.py:49` | — | `(pool_id, request_id)` unique `:54-58`; `priority`, `request_created`, `claim_token`, `claim_expires_at` (`:103-104`) |
| `ResourcePoolAllocation` (join) | `resource_pool_schemas.py:107` | `resource_pool.py:53` | `request_id` unique `:112-115`; `allocated_at`, `released_at` (`:163-164`); `priority` resolved through the policy (`:166-173`) |

The **subject** of a policy is a stack component (`component_id`, `resource_pool_policy_schemas.py:82-90`),
not a user. A step run names its subject via `StepRunRequest.resource_requester`
(`src/zenml/models/v2/core/step_run.py:180`).

### 2.2 `ResourceRequestStatus` state machine (`src/zenml/enums.py:669-678`)

```
                   (admission fails feasibility)
create ──► PENDING ─────────────────────────────────► REJECTED   (terminal)
              │                                          ▲
              │ fits now                                 │ pool capacity shrunk below need
              ▼                                          │ (:1101, :1417)
          ALLOCATED ───────────────────────────────────►─┘
              │  │
              │  └─ higher-priority admission ─► PREEMPTING ─► PREEMPTED  (terminal; §6 Q4)
              │
              ├─ owning step run gone ─────────► CANCELLED   (terminal)
              └─ step run reached terminal state ─► RELEASED (terminal; §6 Q4)
```

`PENDING` also reaches `CANCELLED` (orphan sweep, `test_orphan_cleanup_cancels_allocated_and_queued_requests:1237`)
and `ALLOCATED` (later sweep). Requests are born `PENDING`
(`resource_request_schemas.py:187`, `from_request` hard-codes `ResourceRequestStatus.PENDING.value`) — the
statuses the tests assert immediately after `create_resource_request` are therefore produced **synchronously
inside that call**.

`PREEMPTED` (`enums.py:673`) and `RELEASED` (`enums.py:677`) are **never asserted by any of the 31 tests**.
Both are consumed by `src/zenml/orchestrators/step_launcher.py:836-853`, which fails the step on
`REJECTED`/`PREEMPTED`/`CANCELLED` and proceeds on `ALLOCATED`.

---

## 3. The 15 abstract methods — implementation checklist

`ResourcePoolsStoreInterface` (`store_interface.py:39`) — 13 methods, all reachable over HTTP:

| # | Method | Line | What the tests require |
|---|---|---|---|
| 1 | `create_resource_pool(resource_pool: ResourcePoolRequest) -> ResourcePoolResponse` | `:45` | MUST create per-key `ResourcePoolResourceSchema` rows from `capacity`; MUST accept an inline `policies` list (§6 Q1). Used by `_create_pool:134`. |
| 2 | `get_resource_pool(id, hydrate=True) -> ResourcePoolResponse` | `:58` | MUST raise `KeyError` after deletion (`:1618-1619`); hydrated form MUST populate `active_requests` and `queued_requests` (`resource_pool.py:151-158`). |
| 3 | `list_resource_pools(filter_model, hydrate=False) -> Page[...]` | `:73` | Not exercised by these tests. |
| 4 | `update_resource_pool(id, update: ResourcePoolUpdate) -> ResourcePoolResponse` | `:89` | Carries **three** distinct behaviours: capacity rebuild (§4.6), `attach_policies` (§4.7), `detach_policies` (§4.8). |
| 5 | `delete_resource_pool(id) -> None` | `:103` | MUST raise `IllegalOperationError` while any request is queued or allocated (§4.9). |
| 6 | `create_resource_pool_subject_policy(policy)` | `:111` | Used by the CLI attach path (`cli/resource_pool.py:332`); not directly exercised here. |
| 7 | `get_resource_pool_subject_policy(id, hydrate=True)` | `:124` | Not exercised. |
| 8 | `list_resource_pool_subject_policies(filter_model, hydrate=False)` | `:138` | Not exercised (CLI detach path, `cli/resource_pool.py:392`). |
| 9 | `update_resource_pool_subject_policy(id, update)` | `:154` | Not exercised. |
| 10 | `delete_resource_pool_subject_policy(id) -> None` | `:168` | Not exercised (CLI detach path, `cli/resource_pool.py:402`). |
| 11 | `get_resource_request(id, hydrate=True) -> ResourceRequestResponse` | `:178` | The primary assertion vehicle (`_get_request:173`); MUST return current `status` and `status_reason`. |
| 12 | `list_resource_requests(filter_model, hydrate=False)` | `:193` | Not exercised. |
| 13 | `delete_resource_request(id) -> None` | `:209` | Not exercised. |

`ResourcePoolsSQLStoreInterface` (`store_interface.py:217`) — 2 more, session-scoped and server-internal.
Its `__init__` (`:220-228`) self-registers via `store.resource_pools = self`.

| # | Method | Line | What the tests require |
|---|---|---|---|
| 14 | `release_step_run_resources(session, step_run_id) -> None` | `:231` | Called from `sql_zen_store.py:12898-12900` when a step run finishes or retries. No test asserts it directly. |
| 15 | `create_resource_request(session, resource_request) -> ResourceRequestResponse \| None` | `:242` | **The admission entry point.** Returns `None` when the feature is off. Called from `sql_zen_store.py:12448`; a non-`ALLOCATED` result parks the step in `ExecutionStatus.QUEUED` (`:12457-12461`). |

The tests additionally pin **three symbols not present in the ABC**, which the implementation MUST expose
on `SqlZenStore` (§6 Q2):

| Symbol | Pinned by | Signature evidence |
|---|---|---|
| `SqlZenStore._trigger_resource_request_eviction` | `:793`, `:878`, `:1403` | `(self, session, request_id)` — from the monkeypatch stub `_mock_trigger` at `:789` |
| `SqlZenStore._allocate_queued_requests_for_pool` | `:1609-1611` | keyword-only `(session=..., pool_id=...)`, caller commits |
| `SqlZenStore.reconcile_resource_pools` | `:1868`, `:1922`, `:1985` | no arguments, public, opens its own session |

---

## 4. Behavioural requirements

Notation: `cap[k]` = `ResourcePoolResourceSchema.total`; `occ[k]` = `.occupied`; `lim[k]`/`res[k]` = the
subject policy's effective limit/reserved for key `k`; `req[k]` = the request's requested amount.

### 4.1 Admission — feasibility (→ `REJECTED`, terminal)

| # | Requirement | Proof |
|---|---|---|
| R1 | The engine MUST reject a request whose `req[k]` exceeds the pool's **total** `cap[k]`, regardless of current occupancy. | `test_request_rejected_if_exceeds_pool_total_capacity:183` (cap 1, req 2) |
| R2 | The engine MUST reject a request whose `req[k]` exceeds the subject policy's `lim[k]`, even when the pool has spare total capacity. | `test_request_rejected_if_exceeds_policy_limit:222` (cap 2, limit 1, req 2) |
| R3 | The engine MUST admit at the boundary: `req[k] == cap[k]` and `req[k] == lim[k]` are both **allocatable**, not rejections. | `test_request_allocated_when_equal_pool_total_capacity:301`; `test_request_allocated_when_equal_policy_limit:339` |
| R4 | A **non-preemptible** request MUST be rejected when `req[k] > res[k]` — `reserved` is the hard ceiling for non-preemptible work, not merely a guarantee. | `test_non_preemptible_request_rejected_if_exceeds_reserved_share:261` (res 1, lim 2, req 2 → REJECTED) |
| R5 | A non-preemptible request MUST be admitted at `req[k] == res[k]`. | `test_non_preemptible_request_allocated_when_equal_reserved_share:377` |

### 4.2 Effective limits — key inheritance and the unbounded-key rule

| # | Requirement | Proof |
|---|---|---|
| R6 | A policy key absent from `limit`/`reserved` MUST inherit the pool's `cap[k]` as its effective limit — an empty `limit={}` means "the whole pool", never "unbounded". | `test_missing_policy_key_inherits_bounded_pool_capacity:495` (cap 2, limit `{}`: req 2 → ALLOCATED, req 3 → REJECTED) |
| R7 | An **explicit** `limit[k] == 0` MUST override the inherited pool capacity and reject. | `test_explicit_policy_limit_zero_overrides_pool_capacity:551` (cap 2, limit `{"gpu":0}`, req 1 → REJECTED) |
| R8 | A requested key that is **not** in the pool's capacity map MUST reject, **unless** the key belongs to an intrinsically-unbounded set which includes `"cpu"` and excludes `"disk"`. | `test_missing_bounded_pool_key_rejects_request:455` (`disk` → REJECTED) vs `test_missing_unbounded_pool_key_allows_allocation:416` (`cpu: 100` → ALLOCATED); identical pools in both |
| R9 | When a pool **does** declare a capacity for an otherwise-unbounded key, that capacity MUST be enforced. | `test_explicit_bounded_capacity_for_unbounded_key_is_enforced:1623` (cap `{"cpu":1}`, req `{"cpu":2}` → REJECTED) |

> R8 is the single most under-specified rule in the suite. The membership of the unbounded set is asserted
> by example only and defined nowhere in `src/` — see §6 Q3.

### 4.3 Allocation vs. queueing (→ `ALLOCATED` or `PENDING`)

| # | Requirement | Proof |
|---|---|---|
| R10 | A feasible request that does not fit **right now** MUST go `PENDING`, not `REJECTED`. | `test_allocation_respects_non_preemptible_reserved_share:649` (`req_2`) |
| R11 | Non-preemptible allocations MUST draw **only** from the subject's `reserved` share; once that share is exhausted, further non-preemptible requests queue even while the pool has free capacity within `limit`. | `:649` — cap 2, res 1, lim 2: `req_1` NP → ALLOCATED, `req_2` NP → PENDING |
| R12 | Preemptible allocations MAY draw up to `lim[k]`, including capacity beyond `reserved`. | `:649` — `req_3` preemptible → ALLOCATED, consuming the second unit |
| R13 | There MUST be **no head-of-line blocking**: a later-created eligible request is allocated while an earlier ineligible one still waits. | `:649` — `req_3` (created after `req_2`) is ALLOCATED while `req_2` stays PENDING |
| R14 | The hydrated pool MUST report allocated requests under `active_requests` and queued ones under `queued_requests`. | `:723-728` |

### 4.4 Preemption

| # | Requirement | Proof |
|---|---|---|
| R15 | An incoming request from a **higher-priority** policy MUST preempt a lower-priority *preemptible* allocation: the victim moves to `PREEMPTING` and `_trigger_resource_request_eviction(session, victim_id)` MUST be called exactly once with the victim's id. | `test_preemption_triggers_for_preemptible_victims:816` (`evictions == [victim.id]`, victim → `PREEMPTING`, `:889-895`) |
| R16 | The engine MUST NOT preempt a **non-preemptible** allocation. The higher-priority request goes `PENDING` instead, the victim stays `ALLOCATED`, `evictions == []`, and the pool's queue is non-empty. | `test_preemption_skips_non_preemptible_victims:731` (`:803-815`) |
| R17 | Victim candidates MUST be filtered to allocations holding at least one of the resource **keys** the incoming request needs. | `test_preemption_candidates_filtered_by_requested_resource_keys:1320` — pool holds a `gpu` victim and a `cpu` victim; a `gpu` request evicts only `gpu_victim` (`:1418`) |
| R18 | Preemption MUST be a two-phase handoff: the engine sets `PREEMPTING` and delegates the actual teardown to the eviction hook. It MUST NOT transition the victim straight to `PREEMPTED` inside `create_resource_request`. | `:889-895` asserts `PREEMPTING` with the hook stubbed out |

R16's fixture makes the victim **both** non-preemptible **and** reserved-covered (`reserved={"gpu":1}`,
`:731-760`), so the two protections are confounded there — see §6 Q5.

### 4.5 Orphan reconciliation

| # | Requirement | Proof |
|---|---|---|
| R19 | Before admitting a new request, the engine MUST cancel every request — `ALLOCATED` **and** `PENDING` — whose owning step run no longer exists, and free the capacity they held so the new request can be allocated. | `test_orphaned_requests_are_cancelled_before_allocation:896`; `test_orphan_cleanup_cancels_allocated_and_queued_requests:1237` |
| R20 | A cancelled orphan's `status_reason` MUST be exactly `"Cancelled because owning step run no longer exists."` | `:951-955` |
| R21 | `reconcile_resource_pools()` MUST cancel orphaned allocations with **no** new request as a trigger. | `test_reconciliation_cancels_orphaned_allocations_without_new_requests:1829` |
| R22 | `reconcile_resource_pools()` MUST repair drifted `ResourcePoolResourceSchema.occupied` counters by recomputing them from live allocations. | `test_reconciliation_repairs_occupied_resource_counters:1877` (counter forced to 0 out-of-band, `:1914-1921`; must return to 1) |
| R23 | `reconcile_resource_pools()` MUST delete `ResourcePoolQueueSchema` rows whose request is no longer `PENDING`. | `test_reconciliation_cleans_stale_queue_entries:1937` (1 stale row → 0) |
| R24 | `_allocate_queued_requests_for_pool(session=..., pool_id=...)` MUST perform the same orphan sweep, and MUST NOT commit — the caller commits. | `test_detach_and_delete_pool_succeed_after_requests_are_drained:1607-1612` |

### 4.6 Capacity change (shrink) — queue rebuild

| # | Requirement | Proof |
|---|---|---|
| R25 | Shrinking capacity MUST **not** disturb existing allocations, even ones now over-committed against the new total. | `test_decreasing_pool_capacity_keeps_allocated_and_rebuilds_queue:1101` (allocated `gpu:2` survives `cap 2→1`) |
| R26 | Shrinking capacity MUST re-evaluate every `PENDING` request and reject those that have become infeasible, leaving the still-feasible ones `PENDING`. | `:1101` (`pending_too_large` → REJECTED, `pending_small` → PENDING) |
| R27 | Re-evaluation MUST be per-key and inclusive across **all** keys of a multi-key request. | `test_capacity_rebuild_keeps_boundary_multikey_requests_eligible:1417` — `cap {gpu:1,cpu:4}→{gpu:1,cpu:3}`: `{gpu:1,cpu:3}` stays PENDING, `{gpu:1,cpu:4}` → REJECTED |
| R28 | Re-evaluation MUST NOT reject a pending **non-preemptible** request that still fits its `reserved` share. | `test_capacity_rebuild_keeps_pending_non_preemptible_if_still_eligible:1497` |
| R29 | Growing capacity MUST make newly-declared keys usable by policies that never named them. | `test_policy_without_key_can_use_new_pool_key_after_capacity_update:590` (`disk` rejected before, allocated after `capacity={"gpu":1,"disk":2}`) |

### 4.7 Attaching a policy

| # | Requirement | Proof |
|---|---|---|
| R30 | `ResourcePoolUpdate(attach_policies=[...])` MUST trigger an allocation sweep on the target pool, picking up requests left `PENDING` under a **different** pool's policy. | `test_attaching_policy_allocates_pending_requests_from_other_pool:963` |
| R31 | Once allocated, the request's pool pointer MUST name the pool that actually admitted it, and it MUST be removed from every other pool's queue. | `:1028-1033` (`running_in_pool.id == pool_b.id`; `pool_a.queued_requests == []`) |

R31 implies a pending request is enqueued in **every** pool whose policies admit its component, and that
allocation in one pool is a global dequeue. No test asserts multi-pool enqueue directly — see §6 Q6.

### 4.8 Detaching a policy

| # | Requirement | Proof |
|---|---|---|
| R32 | Detaching a policy MUST raise `IllegalOperationError` while that component has allocated requests in the pool, leaving all request states untouched. | `test_detaching_policy_blocked_if_component_has_active_requests:1035` |
| R33 | The block MUST also fire when the component has **only queued** requests. | `test_detach_policy_blocked_when_component_has_only_queued_requests:1715` |
| R34 | The block MUST be scoped **per component** — another component's activity in the same pool MUST NOT prevent the detach. | `test_detach_policy_not_blocked_by_other_component_activity:1662` |
| R35 | The error message MUST contain the literal substrings `queued=<n>` and `allocated=<n>`. | `:1782-1783` (`"queued=1"`, `"allocated=0"`) |
| R36 | Detach MUST succeed once the component's requests are drained. | `test_detach_and_delete_pool_succeed_after_requests_are_drained:1614-1616` |

### 4.9 Deleting a pool

| # | Requirement | Proof |
|---|---|---|
| R37 | Deleting a pool MUST raise `IllegalOperationError` while any request is queued or allocated, and the pool MUST survive. | `test_deleting_pool_blocked_if_pool_has_active_requests:1185` |
| R38 | The block MUST fire when only **allocated** requests exist, with the same `queued=<n>` / `allocated=<n>` message shape. | `test_delete_pool_blocked_when_only_allocated_requests_exist:1785` (`"queued=0"`, `"allocated=1"`) |
| R39 | Deletion MUST succeed after draining, after which `get_resource_pool` raises `KeyError`. | `:1617-1619` |

### 4.10 Step-run lifecycle coupling

| # | Requirement | Proof |
|---|---|---|
| R40 | Every resource request MUST be owned by a real step run row; the tests build one per request via `_create_step_run_in_db:35-96` (pipeline → snapshot → run `RUNNING` → step run `RUNNING`). | helper `:35` |
| R41 | Deleting the owning **pipeline run** (`delete_run`) MUST make the request an orphan, i.e. the step-run FK MUST cascade. | `:939`, `:1421-1422`, `:1867` |
| R42 | Admission MUST be synchronous inside `create_resource_request` — every test reads the terminal-for-now status immediately after creation, with no polling. | every test, e.g. `:210-219` |
| R43 | A non-`ALLOCATED` result MUST park the step run in `ExecutionStatus.QUEUED`. | `sql_zen_store.py:12456-12461` (production caller, not asserted by these tests) |

### 4.11 Concurrency and locking

The 31 tests are **entirely single-threaded** and express no concurrency requirement. The schema, however,
scaffolds a lease protocol that the engine is expected to use:

| # | Requirement | Evidence |
|---|---|---|
| R44 (SHOULD) | Queue entries SHOULD be claimed under a lease: `ResourcePoolQueueSchema.claim_token` / `claim_expires_at` (`resource_pool_schemas.py:103-104`) with a supporting index on `(pool_id, claim_expires_at, priority, request_created, request_id)` (`:72-81`). | schema only |
| R45 (SHOULD) | Queue draining SHOULD order by `(priority DESC, request_created ASC)` — the shape of the index at `resource_pool_schemas.py:63-71` and of the policy ordering at `:216`. | schema only |
| R46 (SHOULD) | Release SHOULD be soft: `ResourcePoolAllocationSchema.released_at` (`:164`) rather than row deletion, with `request_id` uniquely constrained (`:112-115`) so a request has at most one allocation ever. | schema only |

**Passing all 31 tests therefore does not demonstrate concurrency correctness.** See §7.

---

## 5. Test inventory — acceptance checklist

| # | Test | Line | Pins |
|---|---|---|---|
| 1 | `test_request_rejected_if_exceeds_pool_total_capacity` | 183 | R1 |
| 2 | `test_request_rejected_if_exceeds_policy_limit` | 222 | R2 |
| 3 | `test_non_preemptible_request_rejected_if_exceeds_reserved_share` | 261 | R4 |
| 4 | `test_request_allocated_when_equal_pool_total_capacity` | 301 | R3 (capacity boundary) |
| 5 | `test_request_allocated_when_equal_policy_limit` | 339 | R3 (limit boundary) |
| 6 | `test_non_preemptible_request_allocated_when_equal_reserved_share` | 377 | R5 |
| 7 | `test_missing_unbounded_pool_key_allows_allocation` | 416 | R8 (positive half) |
| 8 | `test_missing_bounded_pool_key_rejects_request` | 455 | R8 (negative half) |
| 9 | `test_missing_policy_key_inherits_bounded_pool_capacity` | 495 | R6 |
| 10 | `test_explicit_policy_limit_zero_overrides_pool_capacity` | 551 | R7 |
| 11 | `test_policy_without_key_can_use_new_pool_key_after_capacity_update` | 590 | R29 |
| 12 | `test_allocation_respects_non_preemptible_reserved_share` | 649 | R10–R14 |
| 13 | `test_preemption_skips_non_preemptible_victims` | 731 | R16 |
| 14 | `test_preemption_triggers_for_preemptible_victims` | 816 | R15, R18 |
| 15 | `test_orphaned_requests_are_cancelled_before_allocation` | 896 | R19, R20 |
| 16 | `test_attaching_policy_allocates_pending_requests_from_other_pool` | 963 | R30, R31 |
| 17 | `test_detaching_policy_blocked_if_component_has_active_requests` | 1035 | R32 |
| 18 | `test_decreasing_pool_capacity_keeps_allocated_and_rebuilds_queue` | 1101 | R25, R26 |
| 19 | `test_deleting_pool_blocked_if_pool_has_active_requests` | 1185 | R37 |
| 20 | `test_orphan_cleanup_cancels_allocated_and_queued_requests` | 1237 | R19 (both states) |
| 21 | `test_preemption_candidates_filtered_by_requested_resource_keys` | 1320 | R17 |
| 22 | `test_capacity_rebuild_keeps_boundary_multikey_requests_eligible` | 1417 | R27 |
| 23 | `test_capacity_rebuild_keeps_pending_non_preemptible_if_still_eligible` | 1497 | R28 |
| 24 | `test_detach_and_delete_pool_succeed_after_requests_are_drained` | 1572 | R24, R36, R39 |
| 25 | `test_explicit_bounded_capacity_for_unbounded_key_is_enforced` | 1623 | R9 |
| 26 | `test_detach_policy_not_blocked_by_other_component_activity` | 1662 | R34 |
| 27 | `test_detach_policy_blocked_when_component_has_only_queued_requests` | 1715 | R33, R35 |
| 28 | `test_delete_pool_blocked_when_only_allocated_requests_exist` | 1785 | R38 |
| 29 | `test_reconciliation_cancels_orphaned_allocations_without_new_requests` | 1829 | R21 |
| 30 | `test_reconciliation_repairs_occupied_resource_counters` | 1877 | R22 |
| 31 | `test_reconciliation_cleans_stale_queue_entries` | 1937 | R23 |

---

## 6. Open questions

**Q1 — `ResourcePoolRequest` has no `policies` field.** `_create_pool:134-140` passes
`ResourcePoolRequest(name=..., capacity=..., policies=[...])`, but the model (`resource_pool.py:81-103`)
declares only `name`, `description`, `capacity`. `BaseZenModel` sets `extra="ignore"`
(`src/zenml/models/v2/base/base.py:45`), so the policies would be **silently dropped**. The spec requires
adding the field; the tests do not say whether inline policies are equivalent to N separate
`create_resource_pool_subject_policy` calls or are transactional with pool creation.

**Q2 — `ResourcePoolUpdate` has no `attach_policies` / `detach_policies`.** The model
(`resource_pool.py:109-122`) carries only `description` and `capacity`; the tests use both new fields
(`:1017`, `:1088`, `:1615`, `:1711`). The shipped CLI takes a different route entirely — separate
`create_resource_pool_subject_policy` / `delete_resource_pool_subject_policy` calls
(`cli/resource_pool.py:332`, `:402`). Whether the CLI must migrate onto the update-model surface, or both
surfaces coexist, is undefined. Relatedly, `component_schemas.py:219-220` excludes
`attach_resource_pools`/`detach_resource_pools` from `ComponentUpdate` — fields that do not exist in the
OSS `ComponentUpdate` at all, implying a symmetric component-side attach surface that is also absent.

**Q3 — the "unbounded key" set is asserted by example and defined nowhere.** R8 rests on `"cpu"` being
allocatable against a pool that does not declare it (`:416`) while `"disk"` is not (`:455`). No list,
constant, or predicate in `src/` names such a set. `ResourceSettings.merged_requested_resources`
(`src/zenml/config/resource_settings.py:245-267`) emits `gpu`, `mcpu`, `memory_mb` and arbitrary
`pool_resources` keys — note **`mcpu`, not `cpu`** — and `sql_zen_store.py:12446` injects a synthetic
`step_run: 1`. So the tests' own `"cpu"` is not even a key the production path produces. **Do not guess
this set; it must be recovered from the Pro implementation or decided explicitly.** It is the single
largest correctness risk in the admission path.

**Q4 — `PREEMPTED` and `RELEASED` are never asserted.** `test_preemption_triggers_for_preemptible_victims:816`
stubs `_trigger_resource_request_eviction`, so nothing pins who writes `PREEMPTED`, when, or what
`status_reason` it carries — even though `step_launcher.py:846-851` reads that reason. Likewise no test
covers `release_step_run_resources` (`store_interface.py:231`) or the `RELEASED` status, so the release
path — does it free `occupied`, set `released_at`, trigger a queue sweep? — is entirely unspecified.

**Q5 — is `preemptible=False` or `reserved` coverage the protection?** `:731`'s victim is non-preemptible
*and* sits inside `reserved={"gpu":1}`. The two guards are confounded; the test name credits the
preemptible flag, but the fixture cannot discriminate. An implementation that protected only
reserved-covered allocations would also pass.

**Q6 — multi-pool enqueue and pool selection.** R31 implies a pending request sits in several pools'
queues at once, but no test asserts it directly, and nothing specifies **which** pool wins when two admit
the same request simultaneously (highest policy priority? most free capacity? lowest pool id?).

**Q7 — victim selection order among equals.** R15/R17 establish *eligibility*. Nothing specifies the
ordering when several eligible victims exist (lowest priority first? newest allocation? largest holding?),
nor whether the engine may evict **multiple** victims to satisfy one request — every preemption test has
exactly one candidate.

**Q8 — `_trigger_resource_request_eviction` contract.** Only its call signature is pinned, by a stub
(`:789-791`). Its return value, failure semantics, idempotency, and whether it may be called more than
once for a victim are all undefined.

**Q9 — the claim/lease protocol has zero coverage.** `claim_token` / `claim_expires_at`
(`resource_pool_schemas.py:103-104`) and their index (`:72-81`) exist in the migration
(`b0ba2c3800e3_add_resource_pools.py`) but no test writes or reads them. Lease duration, renewal,
expiry handling, and the isolation level the allocator needs are all unspecified.

**Q10 — `ResourceRequestResponse.running_in_pool` does not exist.** The tests read it at `:1030-1031`;
the model exposes the same relationship as `.pool` (`resource_request.py:214-221`, backed by
`resource_request_schemas.py:118-130`). Either the tests are stale against a rename, or an alias is
required. Cosmetic, but it is a hard test failure.

---

## 7. Effort estimate

**3–5 engineer-weeks** for one engineer already fluent in `SqlZenStore`, to the point where all 31 tests pass.

| Component | Size | Basis |
|---|---|---|
| 10 CRUD methods (`store_interface.py:45–168`, minus the 3 with engine behaviour) | ~3–5 days | Schemas, response models, filters, routers and CLI all exist; this is wiring against established `SqlZenStore` patterns. |
| Model-surface extensions (Q1, Q2, Q10) + `pool_id` optionality | ~1 day | Small, but touches public request/update models, so it needs the backward-compat review CLAUDE.md mandates. |
| Admission evaluator (R1–R14) | ~4–6 days | Four interacting ceilings (pool total, policy limit, reserved share, current occupancy) × per-key × preemptible/non-preemptible, and R8 is undefined (Q3). |
| Preemption (R15–R18) + eviction handoff | ~3–4 days | Victim filtering, priority comparison, two-phase state handoff; Q7/Q8 need decisions first. |
| Queue management, multi-pool enqueue/dequeue, attach/detach sweeps (R30–R36) | ~4–5 days | Cross-pool invariants are the hardest to get right and the least covered. |
| Capacity rebuild (R25–R29) | ~2–3 days | Re-evaluation over all pending requests, with grandfathering. |
| `reconcile_resource_pools` (R21–R23) | ~2–3 days | Three independent repairs: orphans, counters, stale queue rows. |
| Getting the last few tests green | ~3–5 days | Historically the tail on engine work of this shape. |

**Dominant risk: concurrency, not logic.** All 31 tests run single-threaded against SQLite via
`clean_client`, so they validate the *decision function* and validate nothing about the *allocator under
contention*. Yet the schema is explicitly built for contention — a claim-token lease, a claim-expiry
index, a unique constraint on `resource_pool_allocation.request_id`, and a mutable `occupied` counter that
`test_reconciliation_repairs_occupied_resource_counters:1877` exists precisely because it is expected to
drift. A green suite would therefore be consistent with an implementation that double-allocates capacity
under two concurrent `create_resource_request` calls, or that leaks `occupied` on a rollback. **Budget
explicitly for a concurrency test layer that is not in the 2,020 lines**, and settle Q3 (the unbounded-key
set) before writing the evaluator — it is cheap to decide now and expensive to discover wrong later.
