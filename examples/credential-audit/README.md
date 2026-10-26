# Worked example: credential audit

A fictional investigation, written to exercise the protocol end to end. A requester plans to rotate a shared API key, called **credential X** here. They ask which services use it and how heavily. A researcher answers with a 15-claim packet. A reviewer checks it over three rounds and approves the third revision.

Every name, number and file in this example is invented, except rubber_ducky, which is cited at a real public commit.

## Files

| Path | What it is |
| --- | --- |
| `bundle-r1/` | The first evidence bundle: `manifest.json` beside a `files/` tree. One file, `docs/vendor-key-policy.txt`, was edited after capture, so it **deliberately fails** its manifest digest. |
| `bundle-r2/` | The same bundle with the policy file recaptured unedited. Every file matches its digest. |
| `packet-r1.json` … `packet-r3.json` | The three packet revisions |
| `messages/01`–`06` | The negotiation, in order: request, response, rebuttal, response, rebuttal, response |

| Digest | Of |
| --- | --- |
| `be4cc8bb…150ec9` | `bundle-r1/manifest.json` (named by packet-r1) |
| `1e98b481…2732a8` | `bundle-r2/manifest.json` (named by packets r2 and r3) |
| `2ed36729…2d7cba` | `packet-r1.json` |
| `367c6341…58403f` | `packet-r2.json` |
| `69912a33…42ab05` | `packet-r3.json` |

## Claims

Claim IDs are UUIDs. The short labels below exist only in this README.

| Label | Claim ID | Class (r1 → r2 → r3) | Claim, as of r3 |
| --- | --- | --- | --- |
| L01 | `071d394c-fba8-5a01-97fa-b2d17ea9bfbb` | scope | What the packet covered, and that it skipped deployed containers and the secrets store |
| L02 | `b011925d-b644-5081-a353-a5ee961b1ff0` | observed | Billing reads credential X from `ACME_API_KEY` |
| L03 | `6d615b76-53b5-5ebd-ae6e-8db2ab39e052` | observed | Reports configures its rubber_ducky reviewer with `api_key_env: ACME_API_KEY` |
| L04 | `8dfadd4c-780c-5917-acfb-a8625884edc0` | derived | 1,100 requests under credential X (r1 said 1,160) |
| L05 | `10405289-3ee1-5b56-a58d-21806c7155ce` | inference | legacy-batch may be the reports service under an older name |
| L06 | `5b333cfe-4839-54a7-aef6-6132a307d4df` | assumption | September is representative |
| L07 | `9c5ffb07-b4b7-5308-9b5d-fbf06c472f09` | requester-context | Rotation is due on 2026-11-15 |
| L08 | `cdb13d83-4246-5b67-90b0-55fa381814ad` | recommendation | Confirm rotation status, then rotate after splitting keys |
| L09 | `88f4ea57-73ed-544f-b349-24ec04f84a2c` | qualification | Correction: the total leaves out 3 rows with an empty service field (qualifies L04) |
| L10 | `cb15ac66-8e40-5e43-86ca-6b04cf87dae4` | qualification | Limit: the config findings cover committed files only (qualifies L02 and L03) |
| L11 | `11456531-fd7f-5cbc-a7bc-3c712cde864d` | observed | "Credential X" means the key labelled acme-prod |
| L12 | `3b2d9ed4-51ac-59eb-b7b8-6a1c5d45da82` | unverified | Limit: the export does not say which host sent each request |
| L13 | `8040934b-03bf-593e-a4a7-209cc4cb4896` | observed | rubber_ducky reads the credential named by `api_key_env`, default `LLM_PROVIDER_KEY` (cited at commit `0ed4576`) |
| L14 | `61a0dfb2-a193-5b65-a32a-a4698208838a` | observed | The vendor's policy says to rotate every 90 days |
| L15 | `c92464ad-fc9d-5d75-b931-265b85de9bab` | observed → inference → inference | Billing probably sends most of its requests with credential X |
| L16 | `452f2259-7d86-5cd5-b3fb-977e0a1cd7f2` | — → — → unverified | Whether credential X was already rotated is not established |

The claims include every evidence class. Four of them show labels that a naive conversion would misclassify:

- **L11**, the researcher's own naming convention: `observed`, not `requester-context`.
- **L09**, a correction that limits an earlier count: `qualification`.
- **L10**, a limit that qualifies two other claims: `qualification`.
- **L12**, a limit that states a gap: `unverified`.

## The negotiation

**Round 1.** The researcher submits packet-r1 with bundle-r1. The reviewer answers REVISE:

- **B1** names three claims (L04, L11, L15). The total of 1,160 includes the 60 requests under a second key label, which contradicts the naming claim L11. L15 states as fact something the export contradicts.
- **B2** has an empty `claim_ids` list. The packet never says whether credential X was already rotated.
- **B3** fails closed. The policy file does not match its digest, so it names every claim citing that file (L08 and L14).
- **S1** is a suggestion naming L05. L05 is still verified, because suggestions do not affect status.

**Rebuttal 1.** The researcher answers each blocking concern:

- ACCEPTs B1: L04 is corrected, and L15 is reclassified from observed to inference. Packet-r2 records `previous_class`.
- DISPUTEs B2, adding point R1.
- ACCEPTs B3: the policy file is recaptured in bundle-r2.

The packet digest changes.

**Round 2.** The reviewer resolves B1 and B3, answers R1, and keeps B2 open. It raises **B4**, which names only L08 and cites B2 in its text. This is the coverage-widening pattern: a new concern for the added claim, instead of changing B2's `claim_ids`. The reviewer re-records L09 because the claim it qualifies, L04, changed. Verdict: REVISE.

**Rebuttal 2.** The researcher ACCEPTs B2 and B4. Packet-r3 adds L16 as a gap and rewords L08. L15 drops `previous_class`, because it was not reclassified in this revision.

**Round 3.** The reviewer resolves B2 and B4 and records L08 and L16. Every other record carries over unchanged. Verdict: **APPROVE**.

## Per-claim status

Status is computed after each response. A claim is **blocked** while an open blocking concern names it, and **verified** once it has a verification record (new or carried over).

| Label | After round 1 | After round 2 | After round 3 |
| --- | --- | --- | --- |
| L01 | verified | verified | verified |
| L02 | verified | verified | verified |
| L03 | verified | verified | verified |
| L04 | blocked (B1) | verified | verified |
| L05 | verified (S1 does not block) | verified | verified |
| L06 | verified | verified | verified |
| L07 | verified | verified | verified |
| L08 | blocked (B3) | blocked (B4) | verified |
| L09 | verified | verified (re-recorded) | verified |
| L10 | verified | verified | verified |
| L11 | blocked (B1) | verified | verified |
| L12 | verified | verified | verified |
| L13 | verified | verified | verified |
| L14 | blocked (B3) | verified | verified |
| L15 | blocked (B1) | verified | verified |
| L16 | — | — | verified |

B2 blocks no claim, because it names none. It still has to be resolved before APPROVE.

## Checking it

From the dbunk repository root:

```bash
V='uvx check-jsonschema'
E=examples/credential-audit
for r in 1 2 3; do $V --schemafile schema/packet.schema.json $E/packet-r$r.json; done
$V --schemafile schema/manifest.schema.json $E/bundle-r*/manifest.json
$V --schemafile schema/review-request.schema.json $E/messages/01-review-request.json
$V --schemafile schema/review-response.schema.json $E/messages/0[246]-review-response.json
$V --schemafile schema/rebuttal.schema.json $E/messages/0[35]-rebuttal.json

# Bundle digests: bundle-r1 reports one FAILED file by design; bundle-r2 is all OK.
for b in bundle-r1 bundle-r2; do
  (cd $E/$b/files && sha256sum -c <(jq -r '.files[] | "\(.sha256)  \(.path)"' ../manifest.json))
done
```

Projection, replay through rubber_ducky v1.3, and the rules that span documents are checked by [dbunk-conformance](https://github.com/beardfaceguy/dbunk-conformance).
