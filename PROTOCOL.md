# dbunk protocol

## 1. Purpose and status

To be written in Phase 1.

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

To be written in Phase 1.

## 4. Evidence classes

To be written in Phase 1.

## 5. Evidence bundle

To be written in Phase 1.

## 6. Reviewer contract

To be written in Phase 1.

## 7. Messages

To be written in Phase 2.

## 8. Conformance

To be written in Phase 2.

## 9. Schemas

To be written in Phase 2.
