# TASKS — dbunk v1: protocol, schemas and worked examples

Depends on: docs/plans/2026-10-25-v1-protocol-schemas-and-examples.md, approved ADRs 2026-10-24-verification-layer-on-rubber-ducky-negotiation, 2026-10-24-claims-carry-evidence-classes, 2026-10-24-concerns-are-free-text-with-claim-references, 2026-10-24-immutable-evidence-bundle-and-read-only-reviewer, 2026-10-24-defer-shared-negotiation-library, spec docs/spec.md

Each task: `- [x] T0xx [P?] {behavior} — {files} — verify: {command or check}`

`[P]` means the task touches different files and does not depend on an unfinished task in the same phase.

Shell variables used in verify steps:

- `V='uvx check-jsonschema'`, the schema validator. It is not a dbunk dependency.
- `RD=<a rubber_ducky checkout>`.

## Phase 0 — Foundation

Blocks every later phase.

- [x] T001 `PROTOCOL.md` with section headings 1–9 from the plan, and section 2 written in full. Section 2 covers the pinned reference (commit, path, SHA-256), the normative sections, the sections not adopted, and what "log" means. It does not restate any v1.3 rule. — `PROTOCOL.md` (new) — verify: `git -C $RD show 0ed4576123380169594b48233bb573a573fb094c:skill/references/review-protocol.md | sha256sum` prints `1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53`, and `grep -c 1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53 PROTOCOL.md` ≥ 1 (plan: In scope #1; ADR verification-layer: Confirmation 1–2)
- [x] T002 [P] Shared definitions: `claim_id`, `sha256`, `commit_sha`, `evidence_class`, `relative_path`. — `schema/defs.schema.json` (new) — verify: `$V --check-metaschema schema/defs.schema.json` (plan: Interfaces / Shared definitions)
- [x] T003 Do not restate v1.3 negotiation rules in `PROTOCOL.md`. — `PROTOCOL.md` — verify: a reviewer confirms no section defines round limits, dispositions, verdicts or ID allocation (ADR verification-layer: Confirmation 1)

Checkpoint: the pinned protocol digest matches, and the shared definitions pass the meta-schema check. — `$V --check-metaschema schema/*.schema.json`

## Phase 1 — Packet and bundle

Goal: a researcher can write a valid packet and manifest, and invalid ones are rejected.

Independent check: the valid and invalid fixtures below pass and fail as labelled, with no message schemas present.

- [x] T010 [P] Manifest schema. — `schema/manifest.schema.json` (new) — implements: Interfaces / Evidence bundle — verify: `$V --schemafile schema/manifest.schema.json schema/fixtures/valid/manifest.json`
- [x] T011 [P] Packet schema, including the `contains` scope rule, `qualifies` required exactly for `qualification`, and evidence or `no_evidence_reason`. — `schema/packet.schema.json` (new) — implements: Interfaces / Claim packet — verify: `$V --schemafile schema/packet.schema.json schema/fixtures/valid/packet.json`
- [x] T012 Invalid fixtures. Each one must be rejected:
  - a claim ID starting with `B`, `S` or `R`, and an uppercase UUID
  - a claim with no class, and one with an unknown class
  - a claim with neither evidence nor a reason
  - a packet with no scope claim
  - a qualification with empty `qualifies`
  - a manifest with an absolute path, and one with a `..` path

  — `schema/fixtures/invalid/*.json` (new) — verify: `for f in schema/fixtures/invalid/packet-*.json; do ! $V --schemafile schema/packet.schema.json "$f" || echo "ACCEPTED $f"; done` prints nothing, and the same loop over `manifest-*.json` (ADR claims: Confirmation 1–3; ADR verification-layer: Confirmation 3; ADR bundle: Confirmation 4)
- [x] T013 `PROTOCOL.md` sections 3–6 cover the following.
  - Packet fields and claim-ID format.
  - Evidence references. Each one resolves to a manifest entry or to a path at a pinned full SHA.
  - Packet digest.
  - The class table, word for word from the claims ADR.
  - Scope and qualification rules, including no self-reference and no cycles. The gap list. Reclassification, with `previous_class`.
  - Bundle layout. Manifest completeness: no unlisted files and no duplicate paths. No symlink entries. The bundle digest, and immutability.
  - Reviewer inputs and actions. The model-call channel. The fail-closed rule.

  — `PROTOCOL.md` — verify: every Decision bullet of the claims and bundle ADRs maps to a sentence in sections 3–6 (reviewer check) (plan: In scope #1)
- [x] T014 Do not name a provider, transport, store or agent framework. — `PROTOCOL.md`, `schema/` — verify: `grep -riE 'openrouter|anthropic|openai|langgraph|langchain|sqlite|postgres|mcp' PROTOCOL.md schema/` returns nothing (ADR provider: Confirmation 1; ADR verification-layer: Confirmation 6)

Checkpoint: all valid fixtures pass and all invalid fixtures fail. — the T010–T012 commands

## Phase 2 — Messages

Goal: the four message types validate, with dbunk extension fields, and `PROTOCOL.md` defines the projection.

Independent check: hand-written valid and invalid message fixtures pass and fail as labelled.

- [x] T020 [P] Review-request schema. — `schema/review-request.schema.json` (new) — implements: Interfaces / Messages — verify: `$V --schemafile schema/review-request.schema.json schema/fixtures/valid/review-request.json`
- [x] T021 [P] Review-response schema. Concerns carry `claim_ids` with `uniqueItems`. The response carries `verification_records`. The single-message v1.3 rules are encoded. Concerns have no category field. — `schema/review-response.schema.json` (new) — verify: `$V --schemafile schema/review-response.schema.json schema/fixtures/valid/review-response-*.json`. One fixture has a concern with empty `claim_ids` (ADR concerns: Confirmation 1, 3)
- [x] T022 [P] Rebuttal and escalation-summary schemas. — `schema/rebuttal.schema.json`, `schema/escalation-summary.schema.json` (new) — verify: `$V` against `schema/fixtures/valid/rebuttal.json` and `schema/fixtures/valid/escalation-summary.json`
- [x] T023 Invalid message fixtures. Each one must be rejected:
  - APPROVE with a blocking concern
  - round-1 response with `resolved_concern_ids`
  - round-1 REVISE with no blocking concern
  - a concern with a duplicate claim ID
  - a concern with a `category` field
  - round-3 RE-REVIEW
  - FINAL-POSITION in round 1

  — `schema/fixtures/invalid/{review-response,rebuttal}-*.json` (new) — verify: the same "prints nothing" loop as T012, per schema (ADR concerns: Confirmation 1–2)
- [x] T024 `PROTOCOL.md` section 7 covers the following.
  - The extension fields of each message.
  - `claim_ids` rules: fixed when raised; the widening and narrowing patterns; removed claims; suggestions do not affect status.
  - Verification records and when they carry over, including the extra condition for qualification claims.
  - The no-unaccounted-claims rule and the approval rule.

  Section 8 covers the projection and the single-message versus cross-message boundary. Section 9 is the schema index.

  — `PROTOCOL.md` — verify: every Decision bullet of the concerns ADR and the verification-record rules of the claims ADR map to a sentence (reviewer check) (plan: In scope #1–2)
- [x] T025 Do not add executable source or dependencies. — repository — verify:
  - `ls pyproject.toml requirements*.txt uv.lock package.json yarn.lock 2>/dev/null` prints nothing
  - `grep -rnE '^\s*(import|from) rubber_ducky' --include=*.py .` prints nothing
  - `git ls-files | grep -vE '^(docs/|schema/|examples/|PROTOCOL.md|LICENSE|\.gitignore)'` prints nothing

  (ADR defer: Confirmation 1–3)

Checkpoint: all message fixtures pass or fail as labelled. — the T020–T023 commands

## Phase 3 — Worked example

Goal: one sanitized packet and a full three-round negotiation that ends in APPROVE and replays through rubber_ducky.

Independent check: every file in the example validates against its schema, and the bundle digests recompute.

- [x] T030 Synthetic evidence bundle for a fictional "which services use credential X" investigation:
  - a usage export as CSV
  - two config files
  - a captured documentation snippet
  - a repository reference to the public `beardfaceguy/rubber_ducky` at `0ed4576123380169594b48233bb573a573fb094c`

  — `examples/credential-audit/bundle/manifest.json`, `examples/credential-audit/bundle/files/**` (new) — verify:
  - `cd examples/credential-audit/bundle/files && sha256sum -c <(jq -r '.files[] | "\(.sha256)  \(.path)"' ../manifest.json)`
  - `find . -type l` prints nothing
  - `diff <(find . -type f | sed 's|^\./||' | sort) <(jq -r '.files[].path' ../manifest.json | sort)` is empty

  (ADR bundle: Confirmation 1, 2, 4, 10)
- [x] T031 Packet revision 1, with about 15–20 claims. It contains:
  - at least one claim of each of the nine classes
  - a qualification naming two claims
  - the four kinds of claim where the default label mapping does not apply: the researcher's own naming convention (observed), a correction that limits earlier counts (qualification), a limit that qualifies two other claims (qualification), and a limit that states a gap (unverified)
  - one claim citing a bundle file that will be made missing or digest-mismatched

  — `examples/credential-audit/packet-r1.json` (new) — verify: `$V --schemafile schema/packet.schema.json examples/credential-audit/packet-r1.json` (ADR claims: Confirmation 9, first half)
- [x] T032 Round 1. The request carries the packet-r1 digest and the bundle digest. The response is REVISE with:
  - **B1**, naming three claims (multi-claim)
  - **B2**, with empty `claim_ids`
  - **B3**, naming every claim that cites the broken bundle file (fail-closed)
  - **S1**, with `claim_ids`
  - verification records for the claims B1–B3 do not name

  — `examples/credential-audit/messages/01-review-request.json`, `02-review-response.json` (new) — verify: `$V` against the request and response schemas (ADR concerns: Confirmation 6; ADR bundle: Confirmation 6)
- [x] T033 Round 1 rebuttal and round 2.
  - **Rebuttal 1:** ACCEPTs B1 by reclassifying observed→inference, with `previous_class` recorded in packet-r2. DISPUTEs B2 with R1. ACCEPTs B3 by restoring the file into a new bundle version. Carries the packet-r2 digest.
  - **Response 2:** resolves B1 and B3. Answers R1. Raises B4, which names only one added claim and cites B2 in its text (coverage widening). Gives REVISE.

  — `examples/credential-audit/packet-r2.json`, `bundle-r2/**`, `messages/03-rebuttal.json`, `04-review-response.json` (new) — verify: `$V` against each schema; T030's bundle checks pass for `bundle-r2` (ADR claims: Confirmation 9, reclassification; ADR concerns: Confirmation 5)
- [x] T034 Round 2 rebuttal and round 3. Rebuttal 2 answers B2 and B4, ACCEPTing B4, and carries the packet-r3 digest. Response 3 resolves both and APPROVEs. Every claim in packet-r3 has a verification record, either in response 3 or carried over unchanged from an earlier response. — `examples/credential-audit/packet-r3.json`, `messages/05-rebuttal.json`, `06-review-response.json` (new) — verify: `$V` against each schema (ADR claims: Confirmation 5, 8)
- [x] T035 Conformance script in `beardfaceguy/dbunk-conformance`. It:
  - projects every message as `PROTOCOL.md` section 8 defines;
  - validates the projections against rubber_ducky's v1.3 models at `0ed4576`;
  - replays them with `rubber_ducky.core.lifecycle::replay_review`;
  - checks that each ACCEPTing rebuttal changes the packet digest;
  - checks the cross-document rules: claim IDs unique; `qualifies` IDs present, not self-referencing and acyclic; concern `claim_ids` present in the reviewed revision; no unaccounted claims after each response; APPROVE only with full records; no record carried over for a claim that changed.

  — `beardfaceguy/dbunk-conformance` (new repository) — verify: the script exits 0 on `examples/credential-audit` and reports final status `approved` (ADR verification-layer: Confirmation 4–5; ADR claims: Confirmation 4–7; ADR concerns: Confirmation 2)
- [x] T036 Example README with a per-claim status table after each response. Each claim ID shows a short label. The table shows that S1 does not change any claim's status. — `examples/credential-audit/README.md` (new) — verify: a reviewer checks every row against the messages (ADR concerns: Confirmation 7)
- [x] T037 Do not include private or company detail. — `examples/**`, `PROTOCOL.md`, `schema/**` — verify: `git grep -nEi '<the internal-terms list kept outside the repository>'` returns nothing (plan: Design rules)

Checkpoint: every example file validates, both bundles recompute, and the conformance script exits 0.

## Phase 4 — Close-out

- [x] T040 Run each ADR Confirmation item that v1 can check, and list the reviewer-runtime items that wait for the reviewer plan in the PR body. The waiting items are: the bundle ADR's Confirmation 5 (checkout HEAD) and 7–9, which need a reviewer runtime; and the provider and reviewer-loop ADRs' Confirmation items other than the name scan. — PR description — verify: the checklist is present in the PR

## Done when

- [x] Every in-scope behavior has a checked task
- [x] No out-of-scope item has code
- [x] Every confirmation check in the ADRs that v1 can run passes; the rest are listed as waiting for the reviewer plan
- [x] `$V --check-metaschema schema/*.schema.json`, all fixture loops, the example validations, the bundle checks and the conformance script pass
