# dbunk does not depend on or copy rubber_ducky code until a shared negotiation library is decided

Status: Proposed
Date: 2026-10-24
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

dbunk adopts rubber_ducky's v1.3 negotiation by reference (`2026-10-24-verification-layer-on-rubber-ducky-negotiation`). rubber_ducky's core already separates the negotiation from the reviewed artifact. Its code-review binding (`code/models.py` and `code/service.py`) is 99 lines bound through class-level extension points; with its CLI and MCP wrappers the code mode is 335 lines.

A few rubber_ducky-specific couplings remain in the core:

- the `rubber_ducky/` workspace directory;
- the `rubber_ducky` config path;
- the graph name `rubber-ducky-code`;
- worker-adapter prompt and error wording about a "revised diff";
- a LangGraph checkpoint that mirrors and is reconciled against its own SQLite journal;
- v1.3 message models that reject unknown fields, while dbunk adds fields to them.

dbunk v1 is a specification only and has no engine yet. Its reviewer loop (`2026-10-24-reviewer-loop-with-daimonos-tools`) will validate review responses, and some v1.3 rules can be checked from one message alone while others need conversation state.

When dbunk gets an engine, how does it share the negotiation implementation, and what may dbunk encode before then?

## Decision drivers

- One implementation of the stateful negotiation rules.
- dbunk's engine needs (per-claim status) are known before shared code is shaped.
- dbunk does not pull in the code and plan review modes it does not use.
- The boundary is checkable by a later agent.

## Decision

We will not import rubber_ducky as a dependency and will not copy its lifecycle code into dbunk until a follow-up ADR decides how the negotiation layer is shared.

**What dbunk may encode now.** Single-message constraints, as JSON Schema under `schema/`: field types, enums, ID patterns, and consistency within one message (for example, APPROVE carries no blocking concern).

**What waits for the follow-up ADR.** Cross-message rules that need conversation state:

- round order and expected message type;
- ID sequencing across responses;
- tracking open concerns, and requiring each rebuttal to answer every open concern;
- APPROVE gating on open concerns;
- escalation completeness.

**Trigger.** The follow-up ADR is required before the first plan that implements any cross-message rule, or before the reviewer plan, whichever comes first.

**Conformance replay.** Until then, conformance replay against rubber_ducky's lifecycle runs from a script outside the dbunk repository, with rubber_ducky installed separately, never as a dbunk dependency.

## Alternatives

### A. Copy rubber_ducky's core into dbunk — rejected

The obvious shortcut. Rejected because two copies of the rules drift.

### B. Depend on the `rubber-ducky` package directly — rejected

Rejected because it brings unused review modes, rubber_ducky-named paths and config, and its LangGraph mirror into dbunk, and its models would reject dbunk's extension fields.

### C. Extract a shared library now — rejected

Rejected as premature: dbunk's per-claim needs are unknown until its schemas exist, and the extraction would reshape rubber_ducky's core for a design not yet settled.

### D. Defer, encode only single-message constraints, no dependency and no copy (chosen)

Keeps one stateful implementation and decides its shape once dbunk's needs are known.

## Consequences

- dbunk has no executable negotiation engine for now.
- Cost: the specification may assume behavior the eventual engine cannot provide. Worked examples replayed against rubber_ducky's lifecycle reduce this.
- The follow-up ADR must account for dbunk's extension fields, since v1.3 models forbid extra fields (`2026-10-24-concerns-are-free-text-with-claim-references`).
- Not decided: whether the shared library keeps LangGraph, and where it lives. This must stay consistent with the agent-loop library ADR named in `2026-10-24-reviewer-loop-with-daimonos-tools`.

## Confirmation

A later reviewer or agent checks compliance by:

- No dependency manifest in dbunk (`pyproject.toml`, `requirements*.txt`, `uv.lock`, `package.json`, `yarn.lock`) names `rubber-ducky` or `rubber_ducky`.
- `grep -rnE '^\s*(import|from) rubber_ducky' --include=*.py .` in dbunk returns nothing.
- Until the follow-up ADR is Accepted, dbunk contains no executable source outside `schema/` that validates or applies v1.3 messages, and no function named `apply_review_response`, `apply_rebuttal`, `finalize_escalation` or `replay_review`.
- `PROTOCOL.md` at the dbunk repository root cites the pinned v1.3.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/code/service.py::CodeReviewService`, `code/models.py::CodeReviewState` — a review mode is a thin binding over the core.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/service.py::ReviewService` — hardcodes the `rubber_ducky/` workspace directory.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/service.py::_reconcile_graph` — the LangGraph checkpoint reconciled against the journal.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/workflow.py::build_review_graph` — two-node LangGraph named `rubber-ducky-code`.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/adapters.py::WorkerAdapter` — worker prompt and error wording tied to diffs.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/reviewer_config.py::reviewer_config_path` — the `rubber_ducky` config path.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::ReviewResponse`, `::ProtocolModel` — single-message validators, and `extra="forbid"`.

## Revisit when

The trigger above is reached, or rubber_ducky changes its protocol version or core structure.

## Open questions

None.
