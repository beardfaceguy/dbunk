# A packet is numbered claims with evidence classes and declared scope, approved only when the reviewer has recorded every claim as checked at its stated class

Status: Approved
Date: 2026-10-24
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

The motivating 221-claim investigation packet (`docs/spec.md`) tagged its claims with 15 labels. Six it called evidence classes: Observed, Derived, Inference, Requester context, Unverified and Recommendation. The other nine labels appeared without definitions: Assumption, Qualification, Limit, Search result, Documented, Observed procedure, Correction, Historical correction and Historical identification. Labelling let a reviewer check each claim at the level it asserted rather than demand proof of everything. But the open-ended list left 43 claims with no defined meaning of "checked".

A rule of "approve when every claim is true" cannot be met by an honest packet. "An extra API key may be legacy configuration" (an inference) and "We did not verify a working deployment container" (an unverified gap) are correct as stated without being proven. Such a rule would reward packets that hide their gaps.

The v1.3 rule alone ("no open blocking concern") does not show that anything was checked: a claim the reviewer never examined raises no concern. One of the review's significant omissions, an uninventoried credential field, contradicted no claim. It most likely became attachable because the packet stated its scope: the environment and fields its scan covered.

What must be true of a packet, and of the reviewer's record of it, before the packet can be approved?

## Decision drivers

- Honest gaps must not block approval; hidden gaps must.
- Approval shows that every claim was examined, not only that none was challenged.
- Each claim can be checked on its own.
- Most omissions have a claim to attach to.
- Structure is machine-checkable; judgment stays with the reviewer.

## Decision

We will require every packet to be a set of uniquely numbered claims. Each claim carries **exactly one class** from a closed set, and **evidence references** into the evidence bundle or pinned revisions, or a stated reason none can exist.

The v1 classes, with what "checked at the stated class" means for each:

| Class | Checked when |
| --- | --- |
| observed | The cited evidence shows the statement exactly. |
| derived | Recomputing from the cited inputs and stated method gives the stated result. |
| inference | The statement follows reasonably from the cited evidence and is worded as a possibility, not a fact. |
| assumption | It is stated as an assumption and the packet uses it consistently. |
| requester-context | It is attributed to the requester and not presented as independently checked. |
| unverified | The cited evidence does not establish it, and it is worded as a gap. |
| recommendation | It is worded as a proposed action, not an observed fact or approved decision. |
| scope | It accurately states what the packet covered: sources, time windows, fields, exclusions. |
| qualification | It accurately limits or interprets the claims it lists as qualified. |

- **Scope claims** carry class `scope`. Every packet has at least one.
- **Qualification claims** carry a structured, non-empty list of the claim IDs they qualify, each present in the same packet revision and none equal to the qualification's own ID. Qualification lists may not form a cycle (A qualifies B, B qualifies A, directly or through other qualifications). Field name and encoding belong in the protocol document.
- **The gap list** is the set of claims with class `unverified`. There is no separate list.
- **Default mapping from the motivating packet's labels:** Limit → scope; Qualification → qualification; Search result, Documented, Observed procedure, Correction and Historical correction → observed; Historical identification → requester-context; Assumption → assumption; the six evidence classes map to their namesakes. This is a default only; each claim is classified by the class table, not by its old label. In the motivating packet the default was wrong for at least four claims:
  - the researcher's own naming convention (observed, not requester-context);
  - a "Correction" that limits earlier counts (qualification);
  - a "Limit" that qualifies two other claims (qualification);
  - a "Limit" that states a gap (unverified).

**Verification record.** Each reviewer response lists the claim IDs the reviewer checked at their stated class against the packet revision under review. A claim's record carries over to a later revision only if all of these are unchanged: its text, its class, its evidence references, and the SHA-256 digests of the files those references point to. A qualification claim's record also requires its list of qualified claim IDs to be unchanged, and every claim on that list to keep its own record.

**No unaccounted claims.** After every reviewer response, each claim in the current revision either has a verification record or is named by an open blocking concern. A claim the reviewer could not check therefore becomes a blocking concern naming it, which keeps a legal v1.3 verdict (REVISE or ESCALATE) available whenever a claim is unchecked.

**Approval.** A packet may receive the v1.3 verdict APPROVE only when every claim in the current revision has a verification record and no blocking concern is open.

**Reclassification** (for example, observed to inference) is a valid way to resolve a concern. The revised packet records the claim's previous class, so a human can see every downgrade.

The claim-ID format is a product choice in `docs/spec.md`. Schema encoding is open and belongs in the protocol document.

## Alternatives

### A. Approve when every claim is true — rejected

The obvious rule. Rejected because honest inference and unverified claims could never pass, so it rewards packets that omit their gaps.

### B. "No open blocking concern" alone, as in v1.3 — rejected

The default if dbunk adds nothing. Rejected because an unexamined claim passes, so approval would not show that anything was checked.

### C. One packet-level verdict only — rejected

rubber_ducky's existing shape. Rejected because it cannot show which of 221 claims failed, and a researcher cannot fix what isn't named.

### D. An open, extensible label vocabulary, as the motivating packet used — rejected

Rejected because the reviewer's obligation is defined per class. An open vocabulary left 43 claims with no definition of "checked".

### E. Sample a subset of claims, or accept one reviewer attestation that all were checked — rejected

Cheaper for packets of 200 or more claims. Rejected because neither shows that every claim was examined: sampling leaves most claims unchecked by design, and a single attestation records no per-claim result.

### F. A numeric confidence score per claim — rejected

Rejected because a score invites false precision and gives the reviewer nothing concrete to verify.

### G. Closed classes, scope claims, per-claim verification records (chosen)

Checkable per claim, honest about gaps, shows every claim was examined, and gives omissions an anchor.

## Consequences

- Researchers must classify every claim and declare scope; packets get longer.
- Reviewers judge each claim against its class definition, not against "truth", and must record what they checked.
- Cost: verification records lengthen reviewer responses and extend the v1.3 Review Response (conformance by projection, `2026-10-24-verification-layer-on-rubber-ducky-negotiation`).
- Cost: downgrading contested claims is a cheap route to APPROVE. Recorded previous classes make it visible but do not prevent it.
- Cost: class definitions need worked examples, or reviewers will apply them inconsistently.
- Omission concerns usually attach to a scope claim (`2026-10-24-concerns-are-free-text-with-claim-references`).
- Not decided: schema encoding, how many claims a packet may hold.

## Confirmation

A later reviewer or agent checks compliance by:

- The packet schema rejects a claim without exactly one class from the closed set.
- The packet schema rejects a claim with neither an evidence reference nor a no-evidence reason.
- The packet schema rejects a packet with no `scope` claim, and a `qualification` claim with an empty list of qualified claim IDs.
- Rules that span claims are not JSON Schema checks. For every worked-example packet, a reviewer or agent confirms: claim IDs are unique; each qualification's listed IDs exist in the revision and exclude its own ID; and qualification lists form no cycle. Whether dbunk later ships an executable packet validator is not decided here.
- No conforming implementation records APPROVE while any claim in the current revision lacks a verification record or any blocking concern is open.
- A revised claim whose text, class, evidence references or referenced file digests changed has no carried-over verification record.
- A qualification claim whose list of qualified claim IDs changed, or one of whose listed claims lost its record, has no carried-over verification record.
- No reviewer response in any worked example leaves a claim with neither a verification record nor an open blocking concern naming it.
- A sanitized worked example includes at least one claim of each class, one recorded reclassification, and the four kinds of claim where the default mapping does not apply.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/lifecycle.py::apply_review_response` — APPROVE is refused while blocking concerns are open, and REVISE and ESCALATE need one open, hence the no-unaccounted-claims rule.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/models.py::ReviewResponse` — verdicts are packet-level; `verdict_matches_concerns` forbids APPROVE with blocking concerns, which this decision keeps.
- `2026-10-24-verification-layer-on-rubber-ducky-negotiation` — reserves claim-ID prefixes `B`, `S` and `R`.
- `docs/spec.md` — the motivating packet and review. The packet is private; its label and claim counts were verified by the ADR's reviewer against it.

## Revisit when

Reviewers repeatedly disagree on what a class means in practice, or packets regularly need a class the set does not have.

## Open questions

None.
