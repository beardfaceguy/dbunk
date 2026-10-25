# Reviewers verify packets against an immutable, SHA-256-hashed evidence bundle from a separate read-only context

Status: Proposed
Date: 2026-10-24
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

In the manual review of the motivating packet (`docs/spec.md`), many claims could not be checked:

- **Evidence never handed over.** The packet's local evidence folder and its raw usage export never reached the reviewer; the packet itself said the export needed a separate approved transfer.
- **Sources unreachable.** A model hub was blocked by the reviewer's network policy, and cloud observations could not be re-queried.
- **Evidence that moved.** A cited tracker document was edited after the comments that cited it, and the packet's own SDK copies came from moving upstream branches.

The claims pinned to commits were the ones that could be checked exactly.

The reviewer had not seen the investigation transcript and so did not inherit its framing.

What does a dbunk reviewer receive, and what may it do?

## Decision drivers

- The reviewer can reopen every piece of evidence itself.
- Both sides can prove they examined the same bytes.
- The reviewer does not inherit the author's private reasoning.
- The reviewer cannot alter what it is judging, but can recompute derived claims.
- No credentials travel with the evidence or into the reviewer.

## Decision

We will hand every reviewer an immutable evidence bundle and run it in a separate read-only context.

- **Bundle:** the evidence files plus a manifest listing each file's path, SHA-256, source and capture time. Live sources (cloud APIs, logs, dashboards) enter only as captured snapshots.
- **Bundle digest:** the SHA-256 of the manifest's exact stored bytes. Because the manifest lists every file's digest, one bundle digest covers every file. The manifest's encoding belongs in the protocol document.
- **Immutable:** a changed bundle is a new version with a new digest; a reviewed bundle is never edited.
- **Bound to the packet:** the packet names the bundle digest and the full commit SHA of every repository it cites.
- **Reviewer inputs:** the packet, the bundle, and the pinned revisions as local checkouts or archives provided by the caller. No author transcript and no credentials, including no repository tokens.
- **Reviewer actions:**
  - The reviewer may read its inputs.
  - It may run non-mutating, network-isolated computation with an ephemeral scratch area outside the bundle and checkouts, so it can recompute derived claims. How computation is exposed is the reviewer runtime's choice; for dbunk's own reviewer, see `2026-10-24-reviewer-loop-with-daimonos-tools`.
  - It cannot write to the bundle, the pinned checkouts or the review log, and cannot submit to the review log. The caller submits its response.
- **Fail closed:** evidence that is missing, unopenable or digest-mismatched is a blocking concern on the claims that cite it.

## Alternatives

### A. Cite links only — rejected

The obvious shortcut. Rejected because linked targets move and access differs between author and reviewer, which is exactly what left many claims in the motivating packet unverifiable.

### B. Pinned or content-addressed links only, no bundle — rejected

Pinned commits were the claims that checked exactly. Rejected because live API and log observations, and hosts the reviewer cannot reach, have no pinnable form.

### C. Give the reviewer the author's full conversation — rejected

Rejected because it couples the reviewer to private reasoning, weakens independence, and still lacks a stable evidence identity.

### D. Let the reviewer re-query live systems — rejected

Even with scoped, read-only, short-lived credentials, results are not reproducible, and the two sides cannot prove they saw the same bytes.

### E. Immutable hashed bundle plus a separate read-only reviewer (chosen)

Gives the reviewer both a bounded contract and the means to verify it independently.

## Consequences

- Authors bear the cost of capturing and bundling evidence.
- Bundles can be large (the motivating packet's usage export had about 115,000 records); size handling belongs in the plan.
- A digest proves identity, not truth; semantic review is still required.
- Cost: data that governance will not let into a bundle leaves every claim that cites it blocked until the data can be bundled or the claim is reclassified.
- Sensitive data and credentials must be kept out of bundles, which needs a check, not just a rule.
- Not decided: where bundles are stored and how they are transported; the sandbox that provides network isolation and scratch space (`2026-10-24-reviewer-loop-with-daimonos-tools`).

## Confirmation

A later reviewer or agent checks compliance by:

- Recomputing the bundle digest from the stored manifest bytes and every file digest from stored file bytes, without reserializing, and comparing to the packet and manifest.
- Every manifest entry has a source and a capture time.
- Every evidence reference resolves to a manifest entry or a path at a pinned full commit SHA.
- A write attempted from the reviewer context into the bundle, a pinned checkout or the review log is denied; a write to the scratch area succeeds and is discarded after the review.
- A network request attempted from the reviewer context fails.
- The reviewer's inputs contain no author transcript, and a credential-pattern scan of the bundle and reviewer environment finds nothing.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/audit.py::ArtifactEvidence` — existing path-plus-SHA-256 artifact identity.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/adapters.py::_participant_messages` — rubber_ducky's automatic reviewer sees only review state and has no tools, so it cannot reopen evidence.
- `docs/spec.md` — the motivating packet and review.

## Revisit when

A trusted service can attest evidence and verification results directly, or data-governance rules require a different retention boundary for bundles.

## Open questions

None.
