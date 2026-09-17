# ZenML Feature Audit — OSS vs. Pro parity

**What this is.** A feature-by-feature map of the ZenML fork: where each capability lives, how far
a change to it reaches, which flag or gate controls it, and — the question that drove the work —
**what the open-source build actually gives us versus what is stubbed out and requires ZenML Pro.**

**Measured against** `cc064f100` (branch `main`), on `docs/zenml-feature-audit`. No source code was
changed by this audit.

## Read in this order

1. **[`FINDINGS.md`](FINDINGS.md)** — the verdict. One table: area × tier × what OSS loses × how to
   self-host. Start here.
2. **[`analysis/0001-gate-taxonomy.md`](analysis/0001-gate-taxonomy.md)** — how gating works in this
   codebase. The four tiers, the nine mechanisms, and the vocabulary traps. Read before any dossier.
3. **`findings/`** — one dossier per feature area, each answering the same seven questions.

## Dossiers

| # | Area | File |
|---|---|---|
| 1 | RBAC | [`findings/0001-rbac.md`](findings/0001-rbac.md) |
| 2 | Resource Pools | [`findings/0002-resource-pools.md`](findings/0002-resource-pools.md) |
| 3 | User Management | [`findings/0003-user-management.md`](findings/0003-user-management.md) |
| 4 | Pipelines | [`findings/0004-pipelines.md`](findings/0004-pipelines.md) |
| 5 | Event Triggers | [`findings/0005-event-triggers.md`](findings/0005-event-triggers.md) |
| 6 | Run Templates | [`findings/0006-run-templates.md`](findings/0006-run-templates.md) |
| 7 | AI workflows | [`findings/0007-ai-workflows.md`](findings/0007-ai-workflows.md) |
| 8 | Service Connectors | [`findings/0008-service-connectors.md`](findings/0008-service-connectors.md) |
| 9 | Integrations | [`findings/0009-integrations.md`](findings/0009-integrations.md) |
| 10 | Projects | [`findings/0010-projects.md`](findings/0010-projects.md) |
| 11 | Artifact Management | [`findings/0011-artifact-management.md`](findings/0011-artifact-management.md) |
| 12 | Model Management | [`findings/0012-model-management.md`](findings/0012-model-management.md) |

Resource Pools was not in the original brief. It is included because it is the clearest Tier-A
example in the repo — entitlement-gated, RBAC-typed, fully scaffolded, and missing its engine.

## The dossier template

Every dossier answers these seven, in order:

1. **Surface** — client SDK / CLI / REST router / pydantic model / SQL schema / migration, as paths.
2. **Tier verdict** — A, B, C or D, with the concrete evidence.
3. **Gate inventory** — every mechanism touching the area, labelled 1–9, each with `file:line`.
4. **OSS degradation** — what a user loses, and what substitutes for it.
5. **Self-host path** — Tier B: the env var and class. Tier A: what would have to be written.
6. **Blast radius** — direct callers/callees plus the cross-module hops that matter.
7. **Test coverage** — the mirroring directory under `tests/unit/`.

## Conventions

- Every factual claim carries a `file:line`. A claim without one is a bug in the dossier.
- A Tier A/B verdict names the concrete class, or states explicitly that none exists and how that
  was checked.
- Paths are relative to the repo root unless stated otherwise.

## Provenance

Findings were produced with `gitnexus` (call-graph and route mapping) and `serena` (LSP-accurate
symbol reads) against a fresh index, cross-checked by direct `grep`. Two tooling notes for anyone
extending this work: `gitnexus impact()` returns ~99k characters on hub symbols and must be run
inside a subagent; and only the base `serena` MCP server has the zenml project activated.
