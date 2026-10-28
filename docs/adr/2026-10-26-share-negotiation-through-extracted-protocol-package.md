# rubber_ducky's v1.3 models and lifecycle move into a standalone protocol package, and dbunk's engine depends on it and enforces v1.3 by projection

Status: Proposed
Date: 2026-10-26
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes: 2026-10-24-defer-shared-negotiation-library
Superseded by:

## Context

`2026-10-24-defer-shared-negotiation-library` requires this ADR before the reviewer plan. Until it is accepted, dbunk has no executable code that validates or applies v1.3 messages, and dbunk neither imports nor copies rubber_ducky.

The stateful v1.3 rules live in two rubber_ducky files:

- `core/models.py` (288 lines) holds the message models and their single-message validators.
- `core/lifecycle.py` (344 lines) holds the transitions: round order, ID sequencing, answering every open concern, APPROVE gating and escalation completeness.

Both files import only the standard library and pydantic, and lifecycle imports models. The rest of rubber_ducky's core is what pulls in its heavier dependencies: model adapters, the LangGraph workflow, the SQLite checkpointing and the MCP servers. The rubber_ducky package depends on `langchain`, `langchain-core`, `langgraph`, `langgraph-checkpoint-sqlite`, `mcp`, `pydantic` and `python-dotenv`. Its checkpoint serializer allows deserialization only for the types listed in `checkpointing.py`, and records each type by its module path.

Every v1.3 model sets `extra="forbid"`, so dbunk's extension fields are rejected as sent. `PROTOCOL.md` section 8.2 already resolves this by projection. dbunk-conformance shows that projection works against rubber_ducky's core unchanged:

- it subclasses `ReviewRequestBase` and `RebuttalBase` with one payload field, the packet digest;
- it removes the extension fields;
- it replays the result through `replay_review`.

The worked example replays to `approved`, and breaking a lifecycle rule yields `InvalidTransition`.

The verification-layer ADR and `PROTOCOL.md` section 2.5 allow dbunk's pin on rubber_ducky to move only through a dbunk ADR that names the new commit and digest. rubber_ducky has no release tags yet.

How does dbunk's engine share the stateful negotiation rules with rubber_ducky?

## Decision drivers

- One implementation of the stateful v1.3 rules.
- dbunk takes no dependency on review modes, model adapters, LangGraph or SQLite that it does not use.
- The v1.3 models stay strict for every user. dbunk's fields do not loosen them.
- dbunk pins exactly what it depends on, by a named commit.
- The boundary between v1.3 rules and dbunk rules is visible in code.

## Decision

We will move `rubber_ducky/core/models.py` and `rubber_ducky/core/lifecycle.py` into a standalone Python distribution, kept in the rubber_ducky repository.

- **The package.** Its distribution name is `rubber-ducky-protocol`, and its import package is `rubber_ducky_protocol` (both names proposed). It depends only on pydantic. The two files move with their contents unchanged except for import paths.
- **rubber_ducky's own use.** rubber_ducky depends on the package and imports from it at each call site. There are no re-export shims.
- **dbunk's use.** dbunk's engine depends on the package at a pinned commit. The engine validates every dbunk message against `schema/`. It then projects the message as `PROTOCOL.md` section 8.2 defines, and applies the projection through the package's lifecycle, the mechanism dbunk-conformance uses today. dbunk's own rules (claim references, verification records and carry-over, no unaccounted claims, the approval rule) stay in dbunk.
- **No extension point.** No extension point is added to the v1.3 models. They keep `extra="forbid"`.
- **Installing before releases exist.** Until a release channel is decided, dbunk and dbunk-conformance install the package from the rubber_ducky repository by commit, using a git URL with a subdirectory. rubber_ducky declares its own dependency on the package the same way, so rubber_ducky still installs from git. A commit cannot name its own hash, so that published dependency always points at an earlier commit. rubber_ducky's own development and tests therefore resolve the package from the same tree instead, as a workspace path source; the git URL is what outside installs see. Before a commit that changes the package is tagged, rubber_ducky's published dependency is moved to that commit.
- **The pin does not move here.** Moving dbunk's negotiation-layer pin from `0ed4576` to a commit that contains the package needs its own short ADR, as section 2.5 requires. That ADR names the commit and checks two things:
  - `skill/references/review-protocol.md` at that commit still hashes to `1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53`;
  - the package's models and lifecycle differ from `0ed4576:src/rubber_ducky/core/{models,lifecycle}.py` only in import paths.

  The new commit will also carry rubber_ducky's MIT license, added in `e9b21c9` after the current pin. The reviewer plan cannot pin the package until that ADR is accepted.

### Effect on the superseded ADR and the verification-layer ADR

This ADR supersedes `2026-10-24-defer-shared-negotiation-library`. Its interim rules (no dependency, no copy, replay only from an external script) end when this ADR is accepted. When that happens, the defer ADR's Status becomes Superseded. The checks that still matter carry forward, reworded to match exact names:

| Earlier check | Replaced by |
| --- | --- |
| Defer ADR: no dbunk manifest names `rubber-ducky` or `rubber_ducky` | No dbunk manifest or lockfile has a dependency named exactly `rubber-ducky`. The only rubber_ducky distribution dbunk depends on is `rubber-ducky-protocol`. |
| Defer ADR: `grep -rnE '^\s*(import\|from) rubber_ducky'` returns nothing | `grep -rnE '^\s*(import\|from) rubber_ducky(\.\|\s\|$)' --include=*.py .` in dbunk returns nothing. Imports of `rubber_ducky_protocol` are allowed. |
| Defer ADR: no executable source that validates or applies v1.3 messages; replay only from an external script with rubber_ducky installed separately | dbunk's engine applies v1.3 rules only through the protocol package. dbunk-conformance may depend on the package instead of the full `rubber-ducky`. |
| Verification-layer ADR, Confirmation and Anchors: `rubber_ducky.core.lifecycle::replay_review` | Once the pin moves, `rubber_ducky_protocol.lifecycle::replay_review` at the newly pinned commit. Until then, the existing reference at `0ed4576` stands. |
| Verification-layer ADR, Confirmation: the replay runs from a script outside the dbunk repository with rubber_ducky installed separately | The replay may run in dbunk's engine, through the protocol package, or in dbunk-conformance with the protocol package installed. |

The follow-up ADR that moves the pin lists the verification-layer ADR in its own `Supersedes` line, for those Confirmation and Anchors items only, the way the capability-probe ADR names a partial supersession. It also updates the Commit and SHA-256 rows in `PROTOCOL.md` section 2.

## Alternatives

### A. Depend on the full `rubber-ducky` package — rejected

This is the obvious shortcut, and dbunk-conformance does it today. It is rejected because it brings in unused review modes, rubber_ducky-named paths and config, model adapters, and LangGraph with its SQLite checkpointing.

### B. Copy the two files into dbunk — rejected

Rejected because two copies of the rules drift.

### C. Add an extension point to the v1.3 models upstream — rejected

The extension point could be `extra="allow"` or an `extensions` field. dbunk messages would then validate directly. Rejected because it loosens the models for every rubber_ducky user to suit one consumer. Projection already gives dbunk what it needs.

### D. Extract the LangGraph workflow and persistence too — rejected

Rejected because dbunk's log is its record (`PROTOCOL.md` 2.3), and its loop needs no graph (`2026-10-26-reviewer-loop-on-official-sdks`).

### E. Extract models and lifecycle only, and enforce v1.3 by projection (chosen)

One implementation of the rules, a dependency on pydantic alone, and no change to v1.3 strictness.

## Consequences

- The rubber_ducky repository holds two packages. A protocol change is released from there first.
- dbunk's engine is written in Python, because the shared rules are Python.
- Cost: the extraction is work in rubber_ducky before dbunk's reviewer can start, and a second short ADR is needed to move the pin.
- **Cost: existing rubber_ducky checkpoints.** The checkpoint serializer records each type by module path, and allows only the types it lists. After the move, its allowlist names the new paths, so reviews checkpointed under the old paths would load as raw data.
  - rubber_ducky's journal is canonical, so the migration is to drop those checkpoints and rebuild them from the journal.
  - rubber_ducky needs a test that loads a review checkpointed before the extraction.
- Cost: dbunk keeps a projection layer. That is a second place where a mistake in dbunk's mapping could hide a v1.3 violation. dbunk-conformance's tests guard it.
- **Spec change on acceptance:** the spec's deferred item "A shared negotiation library extracted from rubber_ducky" becomes decided scope.
- Not decided: the release channel and the version scheme of the protocol package.

## Confirmation

A later reviewer or agent checks compliance by:

- **The package:** its dependency list is pydantic only.
- **Diff:** comparing the package's models and lifecycle with `0ed4576:src/rubber_ducky/core/{models,lifecycle}.py` shows only import-path changes.
- **rubber_ducky after the extraction:**
  - its test suite passes, run against the package in the same tree (the workspace path source), not a pinned copy;
  - no module in rubber_ducky re-exports the moved symbols;
  - a review checkpointed before the extraction loads, through the documented rebuild from the journal.
- **dbunk's dependencies:** dbunk's manifest and lockfile name `rubber-ducky-protocol` at a pinned commit. They have no dependency named exactly `rubber-ducky`, `langchain`, `langgraph` or `langgraph-checkpoint-sqlite`.
- **dbunk's imports:** `grep -rnE '^\s*(import|from) rubber_ducky(\.|\s|$)' --include=*.py .` in dbunk returns nothing.
- **The engine:**
  - a test feeds it a message that breaks a v1.3 lifecycle rule, and the engine rejects it with `InvalidTransition` from the package;
  - review state changes only through the package's `apply_review_response`, `apply_rebuttal`, `finalize_escalation` or `replay_review`.
- **dbunk-conformance** passes on the worked example using the protocol package alone.
- **The pin:** dbunk's pin moves only through an accepted ADR naming the commit.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::ProtocolModel`: `extra="forbid"`, and imports limited to the standard library and pydantic.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/lifecycle.py::replay_review`, `::ReviewState`: the stateful rules, and the `request_model`/`rebuttal_model` seam a domain binds.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/checkpointing.py::_REVIEW_CHECKPOINT_TYPES`: the type allowlist for checkpoint deserialization.
- `beardfaceguy/rubber_ducky@0ed4576:pyproject.toml`: the package's dependencies.
- `beardfaceguy/dbunk-conformance@4bb8eb7:src/dbunk_conformance/check.py::PacketReviewRequest`, `::PacketRebuttal`, `::project`: projection against the unchanged core.
- `PROTOCOL.md` sections 2.5 and 8.2: version changes and projection.

## Revisit when

rubber_ducky publishes a protocol version with an extension point, or a second consumer needs the lifecycle with different semantics.

## Open questions

None.
