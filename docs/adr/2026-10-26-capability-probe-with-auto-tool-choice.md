# dbunk's capability probe and reviewer calls use automatic tool choice with strict tool schemas, never a forced tool call

Status: Proposed
Date: 2026-10-26
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes: 2026-10-24-provider-neutral-reviewer-models (the Capability probe bullet, and the Confirmation item on the probe only)
Superseded by:

## Context

`2026-10-24-provider-neutral-reviewer-models` requires a capability probe before any review event is written. That probe "forces a call to one tool and validates the call's arguments against that tool's schema". A model that fails it is refused with `ReviewerModelUnsupported`.

Anthropic's current model reference lists four models that reject forced tool use: Claude Fable 5.1, Claude Mythos 5.1, Claude Opus 5.5 and Claude Sonnet 5.5. Each one answers `tool_choice` `{"type": "any"}` or `{"type": "tool", "name": ...}` with a 400, which also applies on token counting and batches.

The reference recommends instead:

- `{"type": "auto"}` plus a prompt instruction naming the tool;
- `strict: true` on the tool definition, so arguments are schema-valid;
- or structured outputs.

Earlier models, including Claude Opus 5 and Claude Sonnet 5, accept forced tool use. pydantic-ai's model profiles record the same split from live tests.

Chat Completions endpoints accept a function's `strict: true` where they implement it, and some OpenAI-compatible servers ignore it. Either way, the reviewer loop already validates every response against the review-response schema, with one corrective retry (`2026-10-24-reviewer-loop-with-daimonos-tools`).

As written, the approved probe would refuse the current flagship Claude models outright.

How does dbunk test, and then use, a model's ability to return schema-valid tool calls without forcing one?

## Decision drivers

- Current models on every supported wire can be used. The probe does not refuse a capable model.
- The probe still refuses a model or endpoint that cannot return a schema-valid tool call.
- The probe uses the same request shape the review uses, so passing the probe predicts the review.
- No provider-specific request fields beyond what each wire defines.

## Decision

We will replace the provider ADR's Capability probe bullet with this one. Every other part of that ADR stands.

- **Tool definitions.** Every tool dbunk defines carries `strict: true`, in each wire's own form. That covers the review-response tool and every evidence tool. Every request that offers tools sends `tool_choice` as automatic, explicitly: `"auto"` on Chat Completions and `{"type": "auto"}` on Messages. dbunk never sends a forced tool choice.
- **The probe.** Before writing any review event, dbunk sends one probe request. It offers a single strict probe tool, uses automatic tool choice, and carries a system instruction to call that tool. The probe passes only if the response contains exactly one call to the probe tool, and its arguments validate against the tool's schema. Anything else fails: no call, a text-only answer, a different tool, or arguments that do not validate. A failure raises `ReviewerModelUnsupported`, as before.
- **Unchanged rules.** No sampling parameters, no metadata lookup, no override and no retry of the probe. The probe outcome is recorded in the audit.
- **The final response.** A reviewer turn ends when the model calls the review-response tool. A turn that ends without that call, or with arguments that fail the review-response schema, gets the loop's one corrective retry. A second failure fails the turn.

## Alternatives

### A. Keep the forced-call probe, and declare models that reject it unsupported — rejected

This is the obvious shortcut: no change to an approved ADR. It is rejected because it excludes the current flagship models of one of the two wires.

### B. Force the call where the model accepts it, and fall back to auto elsewhere — rejected

This would keep the stronger guarantee on older models. It is rejected for two reasons. dbunk would need a per-model capability table, which the provider ADR forbids ("no metadata lookup"). And two request shapes mean the probe no longer predicts the review.

### C. Structured outputs for the final response, with the probe testing structured output — rejected

This gives a strong guarantee on endpoints that implement it. It is rejected because dbunk would support two features on each wire (tool calling for evidence, structured output for the verdict). The probe would then test a different mechanism from the one the evidence tools use, and some OpenAI-compatible endpoints implement neither consistently.

### D. Automatic tool choice with strict tools and an explicit instruction (chosen)

One mechanism on both wires, matching the model vendor's guidance. The loop's schema validation backs it up.

## Consequences

- dbunk works with the current and older models on both wires.
- Cost: on Messages, a model may answer in text instead of calling the tool. The probe catches a model that does this habitually. An occasional slip costs one corrective retry.
- Cost: on Chat Completions endpoints that ignore `strict`, schema validity rests only on dbunk's validation and retry.
- Cost: an approved ADR is partly superseded. When this ADR is accepted, the provider ADR's Status records the partial supersession.
- `2026-10-26-reviewer-loop-on-official-sdks` pins the probe's request fields to this shape.

## Confirmation

A later reviewer or agent checks compliance by:

- **The probe request.** It carries `tool_choice` automatic on both wires and exactly one tool, with `strict: true`. No request dbunk sends ever carries `tool_choice` `any`, `required`, or a named tool.
- **A forced-choice endpoint.** A fake Messages endpoint that answers any forced `tool_choice` with a 400 accepts the probe and a full scripted review.
- **Failing fake models.** Each of these is refused with `ReviewerModelUnsupported` before any review event is written:
  - one that answers the probe with text only;
  - one that calls a different tool;
  - one whose arguments fail the probe tool's schema.
- **Retries in a review.** A fake model that ends one reviewer turn without calling the review-response tool gets exactly one corrective retry. A second miss fails the turn.

## Anchors

- `2026-10-24-provider-neutral-reviewer-models`: the Capability probe bullet replaced here, and the rest of that ADR, which stands.
- `2026-10-24-reviewer-loop-with-daimonos-tools`: schema validation with one corrective retry.
- Anthropic, Claude API model reference ("Forced tool use removed"): forced `tool_choice` returns a 400 on Claude Fable 5.1, Claude Mythos 5.1, Claude Opus 5.5 and Claude Sonnet 5.5, and the recommended alternatives.
- `pydantic_ai/profiles/anthropic.py` (pydantic-ai-slim 2.61.0), `supports_forced_tool_choice`: the same split, from a third party's live tests.

## Revisit when

A supported wire drops automatic tool choice or strict schemas, or the vendor restores forced tool use on current models.

## Open questions

None.
