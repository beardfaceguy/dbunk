# rubber_ducky's v1.3 models and lifecycle move into a standalone protocol package, and dbunk's engine depends on it and enforces v1.3 by projection

Status: Proposed
Date: 2026-10-26
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

`2026-10-24-defer-shared-negotiation-library` requires this ADR before the reviewer plan. Until it is accepted, dbunk has no executable code that validates or applies v1.3 messages, and dbunk neither imports nor copies rubber_ducky.

The stateful v1.3 rules live in two rubber_ducky files:

- `core/models.py` (288 lines) holds the message models and their single-message validators.
- `core/lifecycle.py` (344 lines) holds the transitions: round order, ID sequencing, answering every open concern, APPROVE gating and escalation completeness.

Both files import only the standard library and pydantic. The rest of rubber_ducky's core is what pulls in its heavier dependencies: model adapters, the LangGraph workflow, the SQLite checkpointing and the MCP servers. rubber_ducky as a package depends on `langchain`, `langgraph`, `langgraph-checkpoint-sqlite`, `mcp` and `python-dotenv`.

Every v1.3 model sets `extra="forbid"`, so dbunk's extension fields are rejected as sent. `PROTOCOL.md` section 8.2 already resolves this by projection. dbunk-conformance shows that the projection runs against rubber_ducky's core unchanged:

- it subclasses `ReviewRequestBase` and `RebuttalBase` with one payload field, the packet digest;
- it removes extension fields from responses;
- it replays the result through `replay_review`.

The worked example replays to `approved`. Breaking a lifecycle rule yields `InvalidTransition`.

How does dbunk's engine share the stateful negotiation rules with rubber_ducky?

## Decision drivers

- One implementation of the stateful v1.3 rules.
- dbunk takes no dependency on review modes, model adapters, LangGraph or SQLite that it does not use.
- The v1.3 models stay strict for every user. dbunk's fields do not loosen them.
- dbunk pins exactly what it depends on, and the pin is checkable.
- The boundary between v1.3 rules and dbunk rules is visible in code.

## Decision

We will move `rubber_ducky/core/models.py` and `rubber_ducky/core/lifecycle.py` into a standalone Python distribution, kept in the rubber_ducky repository.

- **The package.** Its distribution name is `rubber-ducky-protocol` and its import package is `rubber_ducky_protocol` (both proposed). It depends only on pydantic. The two files move with their contents unchanged except for import paths. rubber_ducky itself depends on the package and imports from it at each call site. There are no re-export shims.
- **dbunk's use of it.** dbunk's engine depends on the package at a pinned release. The engine validates every dbunk message against `schema/`, projects it as `PROTOCOL.md` section 8.2 defines, and applies the projection through the package's lifecycle. That is the same mechanism dbunk-conformance uses today. dbunk's own rules (claim references, verification records and carry-over, no unaccounted claims, the approval rule) stay in dbunk.
- **No extension point.** No extension point is added to the v1.3 models. They keep `extra="forbid"`.
- **The pin.** This ADR authorizes moving dbunk's negotiation-layer pin from `0ed4576` to the first tagged rubber_ducky release that contains the package. The condition is that `skill/references/review-protocol.md` at that release still hashes to `1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53`. Any other change of protocol text still needs its own ADR, as the verification-layer ADR says.
- **dbunk-conformance** switches from the full `rubber-ducky` package to the protocol package.

The release channel (git tags only, or also a package index) is not decided here.

## Alternatives

### A. Depend on the full `rubber-ducky` package — rejected

This is the obvious shortcut, and dbunk-conformance does it today. It is rejected for the reasons the defer ADR gave, which still hold: it brings in unused review modes, rubber_ducky-named paths and config, model adapters, and LangGraph with its SQLite checkpointing.

### B. Copy the two files into dbunk — rejected

Rejected because two copies of the rules drift.

### C. Add an extension point to the v1.3 models upstream — rejected

The extension point could be `extra="allow"` or an `extensions` field. dbunk messages would then validate directly. Rejected because it loosens the models for every rubber_ducky user to suit one consumer. Projection already gives dbunk what it needs without that.

### D. Extract the LangGraph workflow and persistence too — rejected

Rejected because dbunk's log is its record (`PROTOCOL.md` 2.3), and its loop needs no graph. Whether dbunk's loop uses LangGraph is decided by `2026-10-26-reviewer-loop-on-official-sdks`.

### E. Extract models and lifecycle only, and enforce v1.3 by projection (chosen)

One implementation of the rules, a pydantic-only dependency, and no change to v1.3 strictness.

## Consequences

- The rubber_ducky repository holds two packages, and a protocol change is released from there first.
- dbunk's engine is written in Python, because the shared rules are Python.
- Cost: the extraction is work in rubber_ducky before dbunk's reviewer can start.
- Cost: dbunk keeps a projection layer. That layer is a second place where a mistake in dbunk's mapping could hide a v1.3 violation. dbunk-conformance's tests guard it.
- Not decided: the release channel and the version scheme of the protocol package.

## Confirmation

A later reviewer or agent checks compliance by:

- The protocol package's dependency list is pydantic only.
- `diff` of the package's models and lifecycle against `0ed4576:src/rubber_ducky/core/{models,lifecycle}.py` shows only import-path changes.
- rubber_ducky's test suite passes after the extraction. No module in rubber_ducky re-exports the moved symbols.
- dbunk's dependency manifest names the protocol package at a pinned release, and does not name `rubber-ducky`, `langchain`, `langgraph` or `langgraph-checkpoint-sqlite`.
- `review-protocol.md` at the newly pinned release hashes to `1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53`.
- dbunk-conformance passes on the worked example using the protocol package alone.
- Every message dbunk's engine accepts is applied through the package's lifecycle.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::ProtocolModel`: `extra="forbid"`, and imports limited to the standard library and pydantic.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/lifecycle.py::replay_review`, `::ReviewState`: the stateful rules, and the `request_model`/`rebuttal_model` seam a domain binds.
- `beardfaceguy/rubber_ducky@0ed4576:pyproject.toml`: the package's dependencies.
- `beardfaceguy/dbunk-conformance@4bb8eb7:src/dbunk_conformance/check.py::PacketReviewRequest`, `::PacketRebuttal`, `::project`: projection working against the unchanged core.
- `PROTOCOL.md` section 8.2: the projection.

## Revisit when

rubber_ducky publishes a protocol version with an extension point, or a second consumer needs the lifecycle with different semantics.

## Open questions

None.
