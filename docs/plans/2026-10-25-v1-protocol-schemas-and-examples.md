# PLAN — dbunk v1: protocol, schemas and worked examples

Status: Draft
Date: 2026-10-25
Spec: docs/spec.md
ADRs: 2026-10-24-verification-layer-on-rubber-ducky-negotiation, 2026-10-24-claims-carry-evidence-classes, 2026-10-24-concerns-are-free-text-with-claim-references, 2026-10-24-immutable-evidence-bundle-and-read-only-reviewer, 2026-10-24-defer-shared-negotiation-library, 2026-10-24-provider-neutral-reviewer-models, 2026-10-24-reviewer-loop-with-daimonos-tools (all approved)
Branch:
Tasks: docs/plans/2026-10-25-v1-protocol-schemas-and-examples/TASKS.md

## Goal

A researcher agent can write a claim packet and evidence bundle that a reviewer agent can check claim by claim. Both sides can exchange review messages that validate against published JSON Schemas and replay through rubber_ducky's v1.3 lifecycle. Anyone can learn the protocol from one document and one worked example.

Success measure (from the spec): a real investigation packet of at least 200 claims, rewritten in dbunk format and reviewed under the protocol, surfaces at least the significant findings an expert manual review found.

## In scope

1. A framework-neutral protocol document (`PROTOCOL.md`) for inter-agent fact-checking, layered on rubber_ducky's review protocol v1.3.
2. JSON Schemas (`schema/`) for claim packets, evidence-bundle manifests, reviewer responses with claim references and verification records, and researcher rebuttals.
3. Worked examples (`examples/`), including a sanitized multi-class packet and a full negotiation that replays through rubber_ducky's lifecycle.

## Out of scope

- An MCP server implementing the protocol.
- dbunk's built-in reviewer agent (model loop, tool and path gates, sandboxed computation).
- A shared negotiation library extracted from rubber_ducky.
- Storage and transport of evidence bundles.

An implementer does not build an out-of-scope item from this plan.

The reviewer-loop and provider ADRs bind the deferred reviewer. They apply here only through the rules below: neither `PROTOCOL.md` nor `schema/` may name a provider, transport, store or agent framework.

## Design rules

- The negotiation rules live only in the pinned rubber_ducky v1.3 text: `beardfaceguy/rubber_ducky@0ed4576123380169594b48233bb573a573fb094c:skill/references/review-protocol.md`, SHA-256 `1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53`. `PROTOCOL.md` cites them and does not restate round limits, dispositions, verdicts or ID allocation.
- `PROTOCOL.md` lists which v1.3 sections are normative and which are not adopted, exactly as `2026-10-24-verification-layer-on-rubber-ducky-negotiation` does.
- dbunk messages are a superset of v1.3 messages. Every dbunk field added to a v1.3 message is listed as an extension field, so the projection knows what to remove.
- `schema/` encodes single-message constraints only: types, enums, ID patterns, and consistency within one message. Rules that span messages, or a message and a packet, are stated in `PROTOCOL.md` and checked by people or the conformance script. They are not checked by JSON Schema.
- dbunk contains no executable source. It has no dependency manifest, no `import rubber_ducky`, and nothing that validates or applies v1.3 messages. Projection, replay and the cross-document checks live in the separate public repository `beardfaceguy/dbunk-conformance`, which installs rubber_ducky at the pinned commit and reads dbunk's examples from a pinned dbunk commit.
- `PROTOCOL.md` and `schema/` name no transport, storage engine, agent framework or model provider. `grep -riE 'openrouter|anthropic|openai'` over both returns nothing.
- Digests are SHA-256 of exact stored bytes, written as 64 lowercase hex characters. Nothing is re-serialized before hashing.
- Everything in `examples/` is synthetic or public. It contains no company names, internal paths, people, credentials or real usage data.
- Company repository rules on auth, pagination, seeds, `__generated__` and jscpd do not apply: this repository has no service, database or source code.

## Interfaces

All schemas are JSON Schema draft 2020-12 with `additionalProperties: false`. Names marked (proposed) are open until the plan's PR merges.

### Shared definitions — `schema/defs.schema.json` (proposed)

| Name | Shape |
| --- | --- |
| `claim_id` | `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$` (lowercase RFC 9562 form; never starts with `B`, `S` or `R`) |
| `sha256` | `^[0-9a-f]{64}$` |
| `commit_sha` | `^[0-9a-f]{40}$` |
| `evidence_class` | enum `observed`, `derived`, `inference`, `assumption`, `requester-context`, `unverified`, `recommendation`, `scope`, `qualification` |
| `relative_path` | non-empty; no leading `/`; no `..` segment; no `\` |

### Evidence bundle

A bundle is a manifest file plus a file tree. Manifest paths are relative to the root of that tree. The manifest is not itself an entry in the tree. The **bundle digest** is the SHA-256 of the manifest's stored bytes.

`schema/manifest.schema.json`:

- `schema_version`: `"1"`
- `files`: array of `{ path: relative_path, sha256, source: string, captured_at: RFC 3339 date-time }`, with at least one entry

Path uniqueness and the rule that the tree has exactly the manifest's files are cross-document rules, stated in `PROTOCOL.md`.

### Claim packet — `schema/packet.schema.json`

The **packet digest** is the SHA-256 of the packet file's stored bytes. A packet revision is identified by its digest.

- `schema_version`: `"1"`
- `bundle_digest`: `sha256`
- `repositories`: array of `{ url: string, commit: commit_sha }`, possibly empty
- `claims`: array of claim objects, with at least one entry and at least one claim of class `scope` (`contains`)

Each claim object has these fields:

- `id`: `claim_id`
- `class`: `evidence_class`
- `text`: non-empty string
- `evidence`: array of evidence references
- `no_evidence_reason` (proposed): non-empty string
- `qualifies` (proposed): array of `claim_id`, required and non-empty when `class` is `qualification`, forbidden otherwise
- `previous_class` (proposed): `evidence_class`, present only on a claim reclassified in this revision

A claim needs either a non-empty `evidence` list or a `no_evidence_reason`.

An evidence reference is one of:

- `{ bundle_path: relative_path, locator?: string }`
- `{ repository: url, path: relative_path, locator?: string }`, where `repository` matches an entry in `repositories`

`locator` (proposed) is free text such as a symbol, line range or column.

### Messages

Each message schema is the v1.3 model's fields plus the extension fields listed below. Field names and constraints are copied from `rubber_ducky/core/models.py` at `0ed4576`.

| Schema | v1.3 fields (copied) | Extension fields (removed by projection) |
| --- | --- | --- |
| `schema/review-request.schema.json` | `protocol_version: "1.3"`, `round: 1`, `task_id`, `title`, `proposed_solution`, `known_concerns`, `questions` | `packet_digest` (becomes the projected payload), `bundle_digest` |
| `schema/review-response.schema.json` | `round` 1–3, `position`, `blocking_concerns`, `suggestions`, `resolved_concern_ids`, `prior_point_responses`, `verdict` | `packet_digest` (the revision reviewed), `verification_records` (array of `claim_id`, unique), and `claim_ids` (array of `claim_id`, unique) on each concern |
| `schema/rebuttal.schema.json` | `round`, `position`, `blocking_responses` (≥1), `new_points`, `requesting` | `packet_digest` (the revised revision, or the unchanged one; becomes the projected payload) |
| `schema/escalation-summary.schema.json` | `unresolved_blocking_concerns` (≥1), `worker_position`, `reviewer_position`, `decision_needed` | none |

The JSON Schema enforces only these single-message v1.3 rules:

- ID patterns: `^B[1-9][0-9]*$`, `^S[1-9][0-9]*$`, `^R[1-9][0-9]*$`.
- APPROVE carries no blocking concerns.
- A round-1 response has no `resolved_concern_ids` and no `prior_point_responses`.
- A round-1 REVISE has at least one blocking concern.
- A round-3 rebuttal is not RE-REVIEW, and FINAL-POSITION appears only in round 3.

Rules on increasing order and cross-list uniqueness cannot be expressed in JSON Schema. They are left to the conformance replay. Rebuttal points carry no claim references in v1, which the concerns ADR leaves undecided.

### Projection (defined in `PROTOCOL.md`, executed outside the repository)

- **Request:** the v1.3 request fields, plus one payload field set to `packet_digest`.
- **Response:** remove `packet_digest`, `verification_records` and every concern's `claim_ids`.
- **Rebuttal:** the v1.3 rebuttal fields, plus one payload field set to `packet_digest`. A rebuttal that ACCEPTs any concern must carry a `packet_digest` different from the one before it.
- **Escalation summary:** unchanged.

### `PROTOCOL.md` sections (proposed order)

1. Purpose and status (dbunk protocol version 1)
2. Negotiation layer: pinned reference, normative sections, sections not adopted, meaning of "log"
3. Claim packet: fields, claim IDs, evidence references, packet digest
4. Evidence classes: the class table with "checked when", scope and qualification rules, gap list, reclassification
5. Evidence bundle: manifest, completeness, bundle digest, immutability, binding to the packet
6. Reviewer contract: inputs, actions, model-call channel, fail-closed rule
7. Messages: extension fields per message, `claim_ids` rules, verification records and carry-over, the no-unaccounted-claims rule, the approval rule
8. Conformance: the projection, replay, and the single-message versus cross-message boundary
9. Schemas: index of `schema/` files

## Error and edge behavior

| Condition | Result |
| --- | --- |
| Claim ID uppercase or not a UUID | Packet schema rejects it |
| Claim with no class, two classes, or a class outside the set | Packet schema rejects it |
| Claim with neither evidence nor `no_evidence_reason` | Packet schema rejects it |
| Packet with no `scope` claim | Packet schema rejects it |
| `qualification` claim with empty or missing `qualifies` | Packet schema rejects it |
| Duplicate claim ID, a `qualifies` ID that is absent or self-referencing, or a qualification cycle | Protocol violation stated in `PROTOCOL.md`; checked in the worked example by the conformance script |
| Manifest path absolute, containing `..`, or a symlink | Manifest schema rejects absolute and `..` paths; the protocol says a symlink entry is invalid, and the example checks find none |
| Bundle file missing, unopenable or digest-mismatched | Reviewer raises a blocking concern whose `claim_ids` names every claim citing that file (shown in the worked example) |
| Tree has files not in the manifest, or the manifest has duplicate paths | Bundle invalid; stated in `PROTOCOL.md`; checked for the example bundle |
| Concern `claim_ids` names an ID absent from the reviewed revision, or repeats one | Response schema rejects repeats (`uniqueItems`); absent IDs are a protocol violation checked by the conformance script |
| Claim the reviewer could not check | Reviewer raises a blocking concern naming it, so no claim is left unaccounted for |
| APPROVE while a claim lacks a verification record | Protocol violation; the conformance script flags it in the example |
| Rebuttal ACCEPTs a concern with an unchanged `packet_digest` | Projection fails; replay reports which rebuttal |
| Removed claim still named by an open concern | Concern stays open until the reviewer resolves it |
| Projected message rejected by the v1.3 models or lifecycle | `InvalidTransition` or a validation error from rubber_ducky, naming the event index |
| Example text contains a company or private detail | Blocking in review; `git grep` scan in Done when |

## Gaps

- **Recording `compute` calls and reviewer audit fields** (`provider`, `model`, `wire`, probe outcome) in the response schema. The reviewer-loop and provider ADRs require them for dbunk's reviewer, which is deferred. They are not in v1 schemas and are added with the reviewer plan as a schema version change.

## Risks

- **The worked example could leak details from the private motivating packet.** It is written from scratch, with synthetic data, and the internal-terms scan runs before every push.
- **Schemas could drift from rubber_ducky's v1.3 models.** The replay validates every projected message against the pinned models. If they disagree, fix the dbunk schema, not the projection.
- **One worked example may not exercise every rule.** The tasks map each ADR Confirmation item to a specific message in the example. Any item still uncovered gets a second example, not a looser rule.
- **The UUID claim IDs make examples hard to read.** Example READMEs show a short label beside each ID in their status tables. The IDs themselves stay UUIDs.
