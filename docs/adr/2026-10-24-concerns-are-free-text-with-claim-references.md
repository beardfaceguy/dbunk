# A reviewer concern is free text that references zero or more claim IDs

Status: Proposed
Date: 2026-10-24
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

A v1.3 concern is an ID, a kind (blocking or suggestion) and free text, and the v1.3 message models reject unknown fields. dbunk needs each concern tied to the claims it challenges. Per-claim status then combines open blocking concerns with the reviewer's verification records (`2026-10-24-claims-carry-evidence-classes`).

Findings from the manual review of the motivating packet (`docs/spec.md`) had different shapes. A wrong citation challenged one claim. A finding about an undisclosed credential consumer bore on an inference, a limit and a recommendation together. An uninventoried configuration field challenged only the packet's declared scope. A fixed list of concern categories would have forced each into the nearest bucket.

How is a concern structured?

## Decision drivers

- Reviewers describe problems in their own words.
- Concerns that name claims can be validated and counted per claim.
- A finding that spans several claims stays one finding.
- Something the packet never mentions can still be raised.

## Decision

We will add a `claim_ids` list to the v1.3 concern. The concern's text stays free text, with no category field and no prebuilt list of concern types. The blocking or suggestion kind is kept from v1.3. dbunk concerns are therefore a superset of the v1.3 concern, checked by projection (`2026-10-24-verification-layer-on-rubber-ducky-negotiation`).

Rules for `claim_ids`:

- Each ID must exist in the packet revision the response reviews, and no ID may appear twice in one list.
- A concern's `claim_ids` are fixed when it is raised. v1.3 forbids reissuing a finding under a new ID, so the set changes only this way:
  - **To cover more claims,** the reviewer raises a new concern about only the added claims, citing the original concern's ID in its text rather than restating it.
  - **To cover fewer claims,** the reviewer resolves the original concern only once it no longer applies to any claim it names. Until then the original stays open and continues to block every claim it names. Any distinct problem that remains after resolution is raised as a new concern with different substance.
- A claim ID may be revised in place (reworded, reclassified, re-evidenced) but is never reassigned to an unrelated assertion, including after its claim is removed.
- If a revision removes a claim that an open concern names, the concern stays open until the reviewer resolves it; removal is the researcher's answer, not an automatic resolution.
- Only blocking concerns affect per-claim status. Suggestions may carry `claim_ids` for reference.

An empty `claim_ids` list means the concern covers something the packet never mentions. Attaching findings to a scope claim where one applies is a reviewer obligation stated in the protocol and judged by people; no validator can enforce it.

## Alternatives

### A. Keep the v1.3 concern unchanged and cite claim IDs in the text — rejected

The obvious shortcut: no message change at all. Rejected because per-claim status could not be computed or validated mechanically, and a claim ID mentioned in passing cannot be told apart from one being challenged.

### B. A category enum (wrong evidence, misclassified, arithmetic error, …) — rejected

It would give built-in statistics by category. Rejected because it pushes reviewers into the nearest bucket, loses nuance, and churns the schema as new kinds of problems appear.

### C. A separate "omission" concern type — rejected

Rejected because required scope claims give most omissions a claim to attach to, and an empty list covers the rest without a second type.

### D. Exactly one claim per concern — rejected

Rejected because one finding often spans claims (the credential-consumer finding bore on three); splitting it duplicates text and fragments the rebuttal.

### E. Free text plus zero or more claim IDs (chosen)

Reviewer judgment stays unconstrained while every attachable concern is attached.

## Consequences

- The concern half of per-claim status is computable from open blocking concerns; verification records supply the rest.
- Cost: dbunk concerns are not valid rubber_ducky v1.3 concerns as sent (v1.3 forbids extra fields). Any shared negotiation library must add an extension point upstream, wrap the model, or carry claim references outside the concern (`2026-10-24-defer-shared-negotiation-library`).
- Cost: no built-in statistics by concern category; any categorization is later analysis of the text.
- Validators must check `claim_ids` against the reviewed packet revision.
- Not decided: whether rebuttal points (`R` IDs) also carry claim references.

## Confirmation

A later reviewer or agent checks compliance by:

- The concern schema in `schema/` has no category or type field beyond the v1.3 kind.
- Validation rejects a concern naming a claim ID absent from the reviewed packet revision, or naming one ID twice.
- A concern with an empty `claim_ids` list validates.
- A later message cannot change an existing concern's `claim_ids`, and a concern resolved while any claim it names still has the problem is a protocol violation the reviewer flags under v1.3 Concern tracking.
- A worked example shows the coverage-widening pattern: a new concern naming only added claims and citing the original concern's ID. The reviewer checks under v1.3 Concern tracking that no new concern restates an open one.
- A sanitized worked example includes one concern naming several claims and one naming none.
- Per-claim status in the worked example ignores suggestions.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::Concern` — the v1.3 concern shape this supersets.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::ProtocolModel` — `extra="forbid"`, so the superset is not directly valid v1.3.
- `beardfaceguy/rubber_ducky@0ed4576:skill/references/review-protocol.md` — Concern tracking: concerns keep stable IDs across rounds, and a finding is never reissued under a new ID.
- `docs/spec.md` — the motivating packet and review.

## Revisit when

Analysis of real reviews shows a small set of recurring concern kinds that reviewers would rather select than write.

## Open questions

None.
