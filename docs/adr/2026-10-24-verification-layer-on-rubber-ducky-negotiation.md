# dbunk adds a claim-verification layer on top of rubber_ducky's three-round negotiation protocol

Status: Approved
Date: 2026-10-24
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

Agent research arrives with confident claims, and confident mistakes look the same. A manual review of one 221-claim investigation packet found the arithmetic and most source claims correct, but also an understated credential consumer, a likely mis-attribution of usage, an uninventoried configuration field and a wrong citation (`docs/spec.md`). A researcher must be able to contest a reviewer's finding, and a deadlock must reach a human rather than loop.

rubber_ducky already defines that negotiation, domain-agnostically. Its review protocol v1.3:

- allows at most three reviewer responses, with one worker rebuttal after each (ACCEPT, DISPUTE or CLARIFY per blocking concern);
- uses reviewer verdicts APPROVE, REVISE or ESCALATE;
- ends with a round-three FINAL-POSITION and a worker-written escalation summary for the human operator.

Its core lifecycle enforces those rules. The same document also fixes a log location (`rubber_ducky/` at the workspace root) and Markdown message rendering, and every v1.3 message model rejects unknown fields. rubber_ducky's author is dbunk's decider, and its original repository is public. It knows nothing about claims, evidence classes or evidence bundles.

Does dbunk define its own back-and-forth, or reuse rubber_ducky's and add only what is specific to fact-checking?

## Decision drivers

- The researcher can contest findings; the exchange is bounded and ends with a human.
- Agents that use both rubber_ducky and dbunk learn one set of negotiation rules.
- No second copy of rules that can drift.
- Anyone can read every rule dbunk depends on.
- The protocol stays framework-neutral: implementations conform to it, and it names no transport, store or agent framework.

## Decision

We will define dbunk as two layers.

**Negotiation layer.** The rubber_ducky review protocol, pinned to `beardfaceguy/rubber_ducky@0ed4576123380169594b48233bb573a573fb094c:skill/references/review-protocol.md` (Version 1.3, SHA-256 `1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53`), in a public repository. It is adopted by reference and not restated.

- **Normative for dbunk:** Role Determination; Overview (round counting and the round-three final position); the content of the Review Request, Review Response and Rebuttal messages; Rules (Concern tracking, Verdicts, Consensus, Honest disagreement, Missing information); the fields of the Escalation report.
- **Not adopted:** the Log File section's location and format (dbunk names no store), Markdown rendering of messages (dbunk messages are JSON), the domain-skill payload rule (dbunk defines its own payload, the packet), and "trivial changes may skip review" (a submitted packet is always reviewed).
- **"Log" in dbunk** means the ordered record of a review's messages, in whatever store an implementation chooses. The worker still owns it, as Role Determination says.

**Verification layer (new).** The claim packet, evidence classes, scope claims, the evidence bundle, reviewer obligations and the approval rule. In v1.3 terms, the reviewed artifact is the packet plus its evidence bundle.

**Conformance by projection.** The verification layer may add fields to v1.3 messages (for example, claim references on concerns) but may not change v1.3 rules. A dbunk exchange conforms when its projection replays cleanly. The projection:

- removes every dbunk extension field;
- binds each request and rebuttal to a minimal v1.3 request or rebuttal type whose only payload field is the packet digest;
- for each rebuttal that ACCEPTs a concern, carries a packet digest different from the previous one, standing in for v1.3's rule that an ACCEPT includes a revised artifact.

Each projected message must validate against the pinned v1.3 message models, and the sequence must replay through the v1.3 lifecycle without an invalid transition. dbunk messages are therefore a superset of v1.3, not directly valid v1.3 messages.

**Claim IDs** must not begin with `B`, `S` or `R`, the prefixes v1.3 uses for concerns, suggestions and rebuttal points. A packet that prefixes one section's claims with `S` produces `S10`, which reads as suggestion 10; `S01` and `R07` look like concern IDs to a person or model. The rest of the claim-ID format belongs in the protocol document.

**Version changes.** dbunk moves to a different rubber_ducky protocol version only through a new dbunk ADR that names the new commit and digest. A change to negotiation semantics is made in rubber_ducky as a new protocol version.

## Alternatives

### A. Write dbunk-specific negotiation rules — rejected

The obvious shortcut: copy and adapt v1.3 into dbunk. Rejected because two rule texts drift, and agents using both tools would learn two slightly different protocols.

### B. Add a "packet" review mode to rubber_ducky — rejected

This would satisfy "one rule set" most directly. Rejected because dbunk owns a reviewer agent with tools (`2026-10-24-reviewer-loop-with-daimonos-tools`), while rubber_ducky's automatic reviewer has none by design. Putting per-claim state and evidence-bundle handling into rubber_ducky's shared core would widen rubber_ducky beyond artifact review.

### C. Reference pinned v1.3 in rubber_ducky's public repository and add a verification layer (chosen)

One negotiation rule set, readable by anyone, with fact-checking semantics kept in dbunk.

## Consequences

- dbunk inherits round limits, ID rules, dispositions and the escalation format without writing them.
- Cost: negotiation changes are made in rubber_ducky first, then adopted by a dbunk ADR.
- Cost: for large packets, every blocking concern must settle within three reviewer responses, every rebuttal must answer every open concern, and there is no partial approval.
- Cost: dbunk messages are not valid v1.3 messages as sent; conformance is checked by projection.
- Cost: rubber_ducky's public repository has no license file. Reading the reference needs none, but reuse by others does, so its author should add one.
- The verification layer must map per-claim results onto v1.3's packet-level verdict (`2026-10-24-claims-carry-evidence-classes`).
- Not decided: where the negotiation layer's code lives (`2026-10-24-defer-shared-negotiation-library`).

## Confirmation

A later reviewer or agent checks compliance by:

- `PROTOCOL.md` at the dbunk repository root contains no section defining round limits, dispositions, verdicts or ID allocation, and cites the pinned commit, path and SHA-256 above.
- The SHA-256 of the pinned `review-protocol.md` bytes equals the digest above.
- The packet schema rejects claim IDs beginning with `B`, `S` or `R`.
- Every worked example, projected as above, validates against the v1.3 message models and replays through `rubber_ducky.core.lifecycle::replay_review` without `InvalidTransition`. The replay runs from a script outside the dbunk repository with rubber_ducky installed separately (`2026-10-24-defer-shared-negotiation-library`).
- In the projected replay, every rebuttal that ACCEPTs a concern carries a new packet digest.
- `PROTOCOL.md` and `schema/` name no transport, storage engine or agent framework.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576123380169594b48233bb573a573fb094c:skill/references/review-protocol.md` — the v1.3 text adopted by reference, including the Log File and Message Formats sections not adopted.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/lifecycle.py::apply_review_response`, `::apply_rebuttal`, `::finalize_escalation`, `::replay_review` — the enforced transitions and the replay used for conformance.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::ProtocolModel` — `extra="forbid"` on every v1.3 message, hence conformance by projection.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::Concern`, `::RebuttalPoint` — the reserved `B`, `S` and `R` ID patterns.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/code/models.py::ReviewRequest`, `::Rebuttal` — how a domain binds one payload field, and the ACCEPT-requires-revision check the projection stands in for.

## Revisit when

rubber_ducky publishes a protocol version that changes round semantics or adds an extension point, or dbunk needs a negotiation behavior v1.3 cannot express (for example, approving part of a packet mid-review).

## Open questions

None.
