# 0003 — Disabled tests as specifications

Measured against `cc064f100`.

## The method

When a vendor ships a feature's *scaffolding* but withholds its *engine*, the behavioural contract
usually survives somewhere. In this repo it survived in a test suite that was disabled rather than
deleted.

**So: treat an unconditionally-disabled test suite as a specification for work not yet done.** It is
better than prose documentation in two ways — it is precise about edge cases, and it is written
against real types rather than narrative.

**It is not, however, automatically executable — and assuming otherwise is the trap.** See §
"The suite does not run at HEAD" below: this one was written against a *richer* model surface than
OSS ships, so un-skipping it is a necessary but far from sufficient definition of done.

This is now a step in the audit process, not a one-off observation. It runs as:

1. Sweep for module-level skips: `grep -rn "allow_module_level=True" tests/ --include="*.py"`.
2. **Discard the conditional ones.** A skip guarded by `if platform.system() == ...` or
   `if sys.version_info >= ...` is an environment guard, not a parked spec.
3. For each unconditional survivor, read the whole file and extract the requirements into
   `specs/NNNN-<topic>.md`, citing `test_name:line` for every requirement.
4. The test inventory becomes the acceptance-criteria checklist.

## The sweep result

Five module-level skips exist in the test tree. **Four are environment guards; exactly one is a
parked specification.**

| File | Skip condition | Verdict |
|---|---|---|
| `tests/unit/zen_stores/test_resource_request_pool_lifecycle.py:30` | **none — unconditional** | **Parked specification** |
| `tests/integration/.../pytorch/materializers/test_pytorch_module_materializer.py:19` | `platform.system() == "Windows"` | Environment guard |
| `tests/integration/.../pytorch/materializers/test_pytorch_dataloader_materializer.py:22` | platform | Environment guard |
| `tests/integration/.../neural_prophet/materializers/test_neural_prophet_materializer.py:22` | platform | Environment guard |
| `tests/integration/.../huggingface/steps/test_accelerate_runner.py:23` | `sys.version_info >= (3, 14)` | Environment guard |

Two facts make the survivor categorically different from the rest:

- It is the **only unconditional** module-level skip in the repository. Its reason string is simply
  `"Resource pool lifecycle tests are disabled."` — no environment, no dependency, no platform.
- It is the **only one in `tests/unit/`**. The other four are integration tests guarding optional
  third-party imports, which is ordinary practice.

It also imports the real `SqlZenStore` and the concrete schemas rather than mocks, so it is written
against the production contract, not a test double.

## Output

The extraction lives at [`../specs/0001-resource-pool-engine.md`](../specs/0001-resource-pool-engine.md)
(cite as `zenml-feature-audit/SPEC-01` — spec numbers are per-product and collide across products, so
always qualify them).

Per the plan-tree convention, `specs/` here is **aspirational**: it states what should become true, in
the future tense. It is not a description of the current system. The matching statement of current
reality is dossier [`../findings/0002-resource-pools.md`](../findings/0002-resource-pools.md), which
records that every resource-pool endpoint returns HTTP 501 today.

## The suite does not run at HEAD — verified

Removing the `pytest.skip` is not enough. The suite was authored against model fields that the OSS
package does not contain, so it encodes a **model API** as well as an engine:

| What the tests use | Reality at `cc064f100` | Failure mode |
|---|---|---|
| `ResourcePoolSubjectPolicyRequest(...)` without `pool_id` | `pool_id: UUID` is **required, no default** (`models/v2/core/resource_pool_subject_policy.py:50-52`) | Loud — `ValidationError` |
| `ResourcePoolRequest.policies` | does not exist | **Silent** — dropped |
| `ResourcePoolUpdate.attach_policies` / `detach_policies` | do not exist | **Silent** — dropped |
| `ResourceRequestResponse.running_in_pool` | does not exist | **Silent** — dropped |

Three of the four fail *silently* because `BaseModel` sets `extra="ignore"`
(`models/v2/base/base.py:45`). A naive "remove the skip and see what breaks" would therefore produce
tests that run, pass the constructor, and assert against a model that quietly discarded the input.

**Consequence for the method:** the extracted spec must state the model surface the tests presume,
not only the behaviour. `SPEC-01` records this as a precondition. It also means the OSS resource-pool
scaffolding is incomplete *at the model layer*, not merely missing its engine — a sharper finding than
dossier 0002 originally recorded.

**Generalised rule:** before treating a disabled suite as executable acceptance criteria, diff the
symbols it references against the symbols that currently exist. Where the package uses
`extra="ignore"`, absent fields will not announce themselves.

## Caveat

A disabled test encodes the behaviour its author chose to test, which is not necessarily the whole
contract. Anything the suite is silent about is genuinely unknown — `SPEC-01 §6 Open questions` is
where that silence is recorded, and it must not be filled in by inference.
