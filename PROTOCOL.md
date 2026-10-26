# dbunk protocol

## 1. Purpose and status

dbunk is a protocol for one agent to fact-check another agent's research. A **researcher** submits a **claim packet**: numbered claims, each with an evidence class and evidence references. With it comes an **evidence bundle**, a fixed, hashed set of the files that evidence points to. A **reviewer** checks every claim at the level its class asserts, raises concerns that name the claims they challenge, and approves the packet only when every claim has been checked and no blocking concern is open. Disagreements are negotiated in at most three rounds and then escalated to a human.

This document is dbunk protocol version 1. It is framework-neutral: it names no transport, store, agent framework or model provider. The schemas in `schema/` are part of it.

"Must" and "must not" mark requirements. A message, packet or bundle that breaks one is invalid.

## 2. Negotiation layer

dbunk does not define its own negotiation. The exchange between a researcher and a reviewer follows the rubber_ducky Agent Review Protocol, adopted by reference and not restated here:

| | |
| --- | --- |
| Repository | `https://github.com/beardfaceguy/rubber_ducky` |
| Commit | `0ed4576123380169594b48233bb573a573fb094c` |
| Path | `skill/references/review-protocol.md` |
| Version | 1.3 |
| SHA-256 of the file bytes | `1841924176bcc56e2d0d17818f1566869d789de745381d7272240f4e061f8d53` |

In that document, the researcher is the **worker** and the reviewer is the **reviewer**. The reviewed artifact is the claim packet together with its evidence bundle.

### 2.1 Normative sections

These parts of the pinned document are normative for dbunk:

- **Role Determination.**
- **Overview**, including how rounds are counted and the round-three final position.
- **Message Formats:** the content of the Review Request, Review Response and Rebuttal messages.
- **Rules:** Concern tracking, Verdicts, Consensus, Honest disagreement, and Missing information.
- **Escalation report:** its fields.

### 2.2 Sections not adopted

- **Log File.** dbunk names no log location or file format.
- **Markdown rendering of messages.** dbunk messages are JSON, defined by the schemas in `schema/`.
- **The domain-skill payload rule.** dbunk defines its own payload, the claim packet.
- **"Trivial changes may skip review."** A submitted packet is always reviewed.

### 2.3 The log

In dbunk, the **log** is the ordered record of one review's messages, kept in whatever store an implementation chooses. The worker owns it, as Role Determination says.

### 2.4 Extensions

dbunk adds fields to the v1.3 messages but never changes a v1.3 rule. Section 7 lists the added fields. Section 8 defines how a dbunk exchange is checked against v1.3.

### 2.5 Version changes

dbunk moves to another version of the rubber_ducky protocol only through a new dbunk decision record that names the new commit and digest.

## 3. Claim packet

A claim packet is one JSON document that validates against `schema/packet.schema.json`. Its fields:

| Field | Meaning |
| --- | --- |
| `schema_version` | `"1"` |
| `bundle_digest` | The digest of the evidence bundle this packet cites (section 5) |
| `repositories` | Every repository the packet cites, each with its `url` and full 40-character `commit` SHA. May be empty |
| `claims` | The claims, in the order the researcher presents them |

### 3.1 Packet digest and revisions

The **packet digest** is the SHA-256 of the packet file's bytes exactly as stored, written as 64 lowercase hex characters. Nothing is re-serialized before hashing. A packet revision is identified by its digest, so any change to the bytes is a new revision.

### 3.2 Claims

Each claim has:

- `id`: the claim ID (3.3).
- `class`: exactly one evidence class (section 4).
- `text`: the statement, as the researcher asserts it.
- `evidence`: evidence references (3.4), or `no_evidence_reason`, a stated reason no evidence can exist. A claim must have at least one evidence reference or a reason.
- `qualifies`: only on a `qualification` claim (4.3).
- `previous_class`: only on a claim reclassified in this revision (4.5).

Claim IDs must be unique within a packet. Schemas cannot check this; a reviewer can.

### 3.3 Claim IDs

A claim ID is a UUID in the lowercase canonical form of RFC 9562, for example `3f2b8c1e-9a4d-4e6f-8b7a-1c2d3e4f5a6b`. Lowercase hex never begins with `B`, `S` or `R`, the prefixes the negotiation layer uses for concerns, suggestions and rebuttal points, so a claim ID cannot be mistaken for one of those.

A claim keeps its ID across revisions. A claim may be revised in place (reworded, reclassified or given new evidence), but its ID is never reassigned to an unrelated assertion, even after the claim is removed.

### 3.4 Evidence references

An evidence reference points either into the bundle or into a cited repository:

- `{ "bundle_path": "<path>" }`: a path that must equal the `path` of a manifest entry in the bundle the packet names.
- `{ "repository": "<url>", "path": "<path>" }`: a path inside a repository. `repository` must equal the `url` of an entry in `repositories`, and the path is read at that entry's pinned commit.

Paths are relative, use forward slashes, and have no empty, `.` or `..` segments. Either form may add `locator`, free text such as a symbol, a line range or a column that narrows the reference.

Every evidence reference must resolve to a manifest entry or to a path at a pinned full commit SHA. A packet must not cite a branch, tag, short SHA or live URL as evidence; live sources enter only as captured files in the bundle.

## 4. Evidence classes

Every claim carries exactly one class. The class says what the claim asserts, and so what "checked" means for it. A reviewer checks each claim against its class definition, not against whether it is ultimately true. An honest inference or an admitted gap passes; a claim that overstates its evidence does not.

### 4.1 The classes

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

The set is closed. A packet must not use any other class.

### 4.2 Scope claims

Every packet must have at least one `scope` claim. Scope claims give a reviewer something to attach an omission to: a finding that the packet missed a source, field or time window usually challenges a scope claim.

### 4.3 Qualification claims

A `qualification` claim must list, in `qualifies`, the IDs of the claims it qualifies. The list must:

- be non-empty, with no repeated ID;
- name only claims present in the same packet revision;
- not name the qualification's own ID.

Qualification lists must not form a cycle: A qualifies B and B qualifies A, directly or through other qualifications. Claims of other classes must not carry `qualifies`.

### 4.4 The gap list

The packet's gaps are exactly its claims of class `unverified`. There is no separate gap list.

### 4.5 Reclassification

Changing a claim's class, for example from `observed` to `inference`, is a valid way to answer a concern. A revised packet that reclassifies a claim must record the claim's class from the revision before in `previous_class`, so a human can see every downgrade. A claim not reclassified in a revision must not carry `previous_class`.

## 5. Evidence bundle

An evidence bundle is a **manifest** plus a **file tree**: the evidence files the packet cites. The manifest is a JSON document that validates against `schema/manifest.schema.json`. It is stored beside the file tree, not inside it.

### 5.1 Manifest

The manifest lists every file in the tree:

| Field | Meaning |
| --- | --- |
| `path` | The file's path relative to the root of the file tree, in the form section 3.4 describes |
| `sha256` | The SHA-256 of the file's stored bytes |
| `source` | Where the file came from: a URL, a command, a system or a person |
| `captured_at` | When it was captured, as an RFC 3339 date-time |

The manifest must be complete. The set of files in the tree must equal the set of manifest paths, with no unlisted files and no path listed twice. No entry may be a symbolic link.

Live sources, such as cloud APIs, logs and dashboards, enter a bundle only as captured snapshots.

### 5.2 Bundle digest

The **bundle digest** is the SHA-256 of the manifest's bytes exactly as stored. The manifest lists every file's digest, so one bundle digest covers every file. Verifiers recompute digests from stored bytes and never re-serialize the manifest before hashing.

### 5.3 Immutability and binding

A bundle is never edited once reviewed. A changed bundle is a new bundle with a new digest. A packet names the digest of the bundle it cites, and the full commit SHA of every repository it cites, so both sides can prove they examined the same bytes.

Sensitive data and credentials must not enter a bundle.

## 6. Reviewer contract

This section states what any reviewer receives and may do, whoever implements it.

### 6.1 Reviewer context and model-call channel

The **reviewer context** is the environment where the reviewer's tool calls and computation run. The **model-call channel** is separate. It holds only the model credential, reaches only the configured model endpoint, and never runs tools.

Anything the reviewer reads may be sent to the model endpoint, which may be a third party. Rules about what may enter a bundle must also cover what may be sent there.

### 6.2 Inputs

The reviewer receives:

- the packet;
- the bundle;
- each cited repository at its pinned commit, as a checkout or archive the caller provides. A checkout's HEAD must equal the pinned full SHA with a clean working tree, and an archive's tree must match that commit.

The reviewer does not receive the researcher's transcript or private reasoning. The reviewer context holds no credentials, including repository tokens.

### 6.3 Actions

The reviewer:

- may read its inputs;
- may run computation that does not modify its inputs and has no network access, with an ephemeral scratch area outside the bundle and checkouts, to recompute `derived` claims;
- must not write to the bundle, the pinned checkouts or the log;
- does not submit to the log. The researcher records the reviewer's response, as the negotiation layer's worker.

### 6.4 Failing closed

Evidence that is missing, cannot be opened, or does not match its manifest digest is never treated as checked. The reviewer must raise a blocking concern that names every claim citing it.

## 7. Messages

To be written in Phase 2.

## 8. Conformance

To be written in Phase 2.

## 9. Schemas

To be written in Phase 2.
