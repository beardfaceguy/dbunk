# dbunk

dbunk is a protocol for one AI agent to fact-check another agent's research.

An agent that researches a question for you produces a confident answer. You usually can't tell which parts it saw, which it calculated, which it guessed and which it left out. A second agent asked to "review this" has the same problem: it reads a polished document, with nothing to check it against.

dbunk fixes the handover between them. The researcher must:

- break its answer into numbered **claims**;
- say for each claim what kind of statement it is;
- hand over the exact evidence it relied on, frozen and hashed.

The reviewer then checks every claim against that evidence, and may approve only when nothing is left unchecked.

This repository holds the protocol, its JSON Schemas and a worked example. It is version 1, and framework-neutral: it names no agent framework, model provider, transport or store.

## How it works

```
 researcher                                   reviewer
     |                                           |
     |  Review Request: claim packet + bundle    |
     |------------------------------------------>|  checks every claim
     |                                           |  against the bundle
     |  Review Response: concerns + verdict      |
     |<------------------------------------------|
     |                                           |
     |  Rebuttal + revised packet                |
     |------------------------------------------>|
     |                    ...                    |
     |      at most three rounds, then APPROVE   |
     |      or an escalation summary for a human |
```

### 1. The claim packet

The researcher turns its answer into a **claim packet**: a JSON document listing every factual statement, number, conclusion and recommendation as a separate claim. Each claim has an ID, its text, exactly one **evidence class**, and references to the evidence behind it.

### 2. Evidence classes

The class says what the claim asserts, and so what "checked" means for it:

| Class | The claim passes review when |
| --- | --- |
| observed | the cited evidence shows it exactly |
| derived | recomputing from the cited inputs and stated method gives the stated result |
| inference | it follows reasonably from the evidence, and is worded as a possibility |
| assumption | it is stated as an assumption and used consistently |
| requester-context | it is attributed to the person who asked, not presented as checked |
| unverified | it is worded as a gap the evidence does not close |
| recommendation | it is worded as a proposal, not a fact or a decision |
| scope | it accurately states what was covered and what was left out |
| qualification | it accurately limits the claims it lists |

The reviewer judges each claim against its class, not against whether it is ultimately true. An honest inference or an admitted gap passes. A claim that overstates its evidence fails, even if it happens to be right.

Every packet needs at least one `scope` claim. That gives the reviewer somewhere to attach an omission, such as a source that was never searched.

### 3. The evidence bundle

Every file the claims cite goes into an **evidence bundle**: a file tree with a `manifest.json` beside it. The manifest lists each file's path, SHA-256 digest, source and capture time.

- Live sources, such as APIs, logs and dashboards, go in as captured snapshots. The reviewer never re-queries a live system, so both sides examine the same bytes.
- The SHA-256 of the manifest is the **bundle digest**. It covers every file, and the packet names it.
- A bundle is never edited once reviewed. A changed bundle is a new bundle with a new digest.
- Repositories are cited by full 40-character commit hash, never by branch or tag.

### 4. The review

The reviewer reads the packet and the bundle. It is read-only. It may recompute `derived` claims in a scratch area, offline. It never sees the researcher's transcript or private reasoning.

Its response contains:

- **Blocking concerns** (`B1`, `B2`, ...): free text, each naming the claim IDs it challenges. An empty list means something the packet never mentions.
- **Suggestions** (`S1`, `S2`, ...), which never block approval.
- **Verification records:** the claims it checked and passed in this revision.
- **A verdict:** APPROVE, REVISE or ESCALATE.

Evidence that is missing, unreadable or fails its digest is never treated as checked. The reviewer raises a blocking concern naming every claim that cites it.

### 5. The rules that make approval mean something

- **No unaccounted claims.** After every response, each claim must either have a verification record or be named by an open blocking concern.
- **APPROVE needs every claim verified** and no blocking concern open.
- **Changed claims are re-checked.** A verification record carries over to a new revision only if the claim's text, class, evidence references and evidence files are all unchanged.
- **Downgrades are visible.** A claim reclassified to answer a concern, for example from `observed` to `inference`, records its previous class.

### 6. Negotiation and escalation

The back-and-forth follows the [rubber_ducky](https://github.com/beardfaceguy/rubber_ducky) Agent Review Protocol v1.3, pinned by commit and digest. dbunk adds fields to its messages but never changes its rules:

- the researcher answers each blocking concern with ACCEPT (with a revised packet), DISPUTE or CLARIFY;
- there are at most three rounds;
- if round 3 does not end in APPROVE, the researcher writes an escalation summary, and a human decides.

## Repository layout

| Path | What it holds |
| --- | --- |
| [`PROTOCOL.md`](PROTOCOL.md) | The normative specification |
| [`schema/`](schema/) | JSON Schemas (draft 2020-12) for the packet, manifest and every message, with valid and invalid fixtures |
| [`examples/credential-audit/`](examples/credential-audit/) | A fictional end-to-end review: three packet revisions, two bundles (one with a deliberately tampered file) and all six messages |
| [`docs/adr/`](docs/adr/) | Architecture decision records |
| [`docs/plans/`](docs/plans/) | Implementation plans |
| [`docs/spec.md`](docs/spec.md) | Product choices |

## Checking documents

The schemas check one document at a time. To validate a packet:

```sh
uvx check-jsonschema --schemafile schema/packet.schema.json my-packet.json
```

Rules that span documents, such as unique claim IDs, digests, carry-over of verification records and conformance to rubber_ducky v1.3, are checked by the separate [dbunk-conformance](https://github.com/beardfaceguy/dbunk-conformance) repository.

## Status

Version 1 of the protocol, schemas and worked example is complete. An automated reviewer that runs the review loop against a model is being designed; see the decision records in `docs/adr/`.

## License

[MIT](LICENSE)
