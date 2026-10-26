# dbunk's capability probe and reviewer calls use automatic tool choice and the review's own tools, never a forced tool call

Status: Proposed
Date: 2026-10-26
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes: 2026-10-24-provider-neutral-reviewer-models, in part: the first sub-bullet and the rationale of its Capability probe bullet, and its Confirmation item on fake models refused by the probe. Each is listed in the Decision.
Superseded by:

## Context

`2026-10-24-provider-neutral-reviewer-models` requires a capability probe before any review event is written. Its Capability probe bullet has five sub-bullets:

1. The probe "forces a call to one tool and validates the call's arguments against that tool's schema", because "one request covers both tool calling and structured output".
2. The probe, like every review request, sends no sampling parameters.
3. A failed probe raises `ReviewerModelUnsupported`.
4. There is no metadata lookup, no override and no retry.
5. The probe behaves the same for aggregator, direct and self-hosted endpoints.

Its Confirmation has one item on refusals: a fake model that returns no tool call, and one that returns a schema-invalid call, are both refused before any review event is written, and the probe outcome appears in the audit.

Anthropic's documentation says that four models answer forced tool use with a 400: Claude Fable 5.1, Claude Mythos 5.1, Claude Opus 5.5 and Claude Sonnet 5.5. "Forced" means `tool_choice` `{"type": "any"}` or `{"type": "tool", "name": ...}`, and the 400 also applies on token counting and batches. The documentation recommends `{"type": "auto"}` plus an instruction naming the tool, and warns that `auto` does not guarantee a call, so the caller must check that one was made. Claude Fable 5 and Claude Opus 5 accept forcing. Earlier Anthropic models reject it when extended thinking is on, according to pydantic-ai's model profiles, which also record the 5.x split from live tests.

As written, the approved probe would refuse the current flagship Claude models.

Strict tool schemas (`strict: true`) were considered alongside automatic choice, and rejected. They accept only a subset of JSON Schema. dbunk's review-response schema already uses features outside that subset: numeric bounds, `uniqueItems`, conditionals, optional properties and cross-file references. Daimonos's tool schemas are not dbunk's to change. Some models, deployments and gateways do not support strict at all.

The reviewer loop already validates every response against the review-response schema, with one corrective retry (`2026-10-24-reviewer-loop-with-daimonos-tools`).

How does dbunk test, and then use, a model's ability to call the review's tools with valid arguments, without forcing a call?

## Decision drivers

- Current models on every supported wire can be used.
- The probe still refuses a model or endpoint that cannot call the review's tools with valid arguments, before any review event is written.
- Passing the probe predicts the review: the probe sends what the review sends.
- No request field beyond what each wire defines, and no per-model table.

## Decision

We will replace sub-bullet 1 and its rationale. Sub-bullets 2 to 5 stand unchanged:

- no sampling parameters;
- `ReviewerModelUnsupported` on failure;
- no metadata lookup, no override and no retry;
- the same behaviour for aggregator, direct and self-hosted endpoints.

The replacement is as follows.

### Requests

- Every request that offers tools sends automatic tool choice explicitly: `"auto"` on Chat Completions, `{"type": "auto"}` on Messages.
- dbunk never sends a forced tool choice.
- No request carries `strict`.
- **Tool schemas.** Tool definitions carry dbunk's own JSON Schemas and the pinned Daimonos schemas, each as one self-contained schema. dbunk bundles each schema, and validation meaning is unchanged:
  - every reference to another file, such as `defs.schema.json#/$defs/claim_id`, is inlined into the schema's own `$defs`, and the `$ref` is rewritten to point there;
  - `$defs`, local `$ref`s and every validation keyword are kept;
  - only the top-level `$schema`, `$id` and `title` are dropped.

  dbunk's validator checks arguments against the same bundled schema, and applies JSON Schema draft 2020-12 explicitly, since the bundled schema no longer names its dialect. If an inlined definition's name clashes with a key already in the schema's local `$defs`, bundling fails with an error. Names are never silently renamed or merged.

### The probe

- **When.** Before writing any review event, dbunk sends one probe request.
- **What it offers.** The probe request offers exactly the tool definitions a review request offers: the review-response tool, the allowlisted Daimonos tools and `compute`.
- **What it asks.** It carries a system instruction to call the review-response tool once, with a minimal valid review response that the instruction supplies.
- **When it passes.** The response must contain exactly one tool call, to the review-response tool, whose arguments validate against the full review-response schema in dbunk's own validator. Text or thinking blocks alongside that call are allowed.
- **When it fails.** Any of these fails the probe:
  - no call;
  - two or more calls;
  - a call to another tool;
  - arguments that do not validate;
  - an endpoint error, including a rejected tool definition.
- **The result.** A failure raises `ReviewerModelUnsupported`, and the probe outcome is recorded in the audit, as before.
- **Not a review.** The probe's call is discarded and never becomes a review event.

### The final response

This extends `2026-10-24-reviewer-loop-with-daimonos-tools`; it does not supersede it.

- A reviewer turn ends when the model's response contains a call to the review-response tool and no other tool call.
- A response that ends the turn in any other way gets the loop's one corrective retry. That covers text only, a review-response call mixed with evidence-tool calls, and arguments that fail the schema.
- **Answering every call.** The retry message answers every tool call in the failed response, as each wire requires: a `tool_result` for each `tool_use` on Messages, and a `tool` message for each `tool_call` on Chat Completions. A call that was not run gets an error result saying why. The same pairing holds whenever a conversation continues after a response that contained tool calls.
- A second failure fails the turn.

### The approved ADR on acceptance

When this ADR is accepted, the provider ADR's header changes as follows:

- its status becomes `Approved (Capability probe sub-bullet 1 and one Confirmation item superseded by 2026-10-26-capability-probe-with-auto-tool-choice)`;
- its superseded-by field becomes `2026-10-26-capability-probe-with-auto-tool-choice (in part)`.

## Alternatives

### A. Keep the forced-call probe, and declare models that reject it unsupported — rejected

This is the obvious shortcut: no change to an approved ADR. It is rejected because it excludes the current flagship models of one of the two wires.

### B. Force the call where the model accepts it, and use auto elsewhere — rejected

This would keep the stronger guarantee on older models. It is rejected in both of its forms:

- Choosing by model needs a capability table, which the provider ADR forbids ("no metadata lookup").
- Trying a forced call and falling back to auto on a 400 is a retry of the probe, which the provider ADR also forbids. It is the same silent downgrade for which the sibling SDK ADR rejects pydantic-ai.

Either form also gives the probe and the review two different request shapes.

### C. Strict tool schemas with automatic choice — rejected

This would give schema-valid arguments wherever strict is supported. It is rejected for two reasons. dbunk's schemas and Daimonos's schemas fall outside strict's JSON Schema subset, so each would need a derived strict copy. And strict is unavailable on some models, deployments and gateways, so the probe would refuse endpoints that can review perfectly well with dbunk's own validation.

### D. Structured outputs for the final response — rejected

This is rejected mainly because the evidence tools and the verdict would then use two different mechanisms, so the probe could no longer test what the review uses.

### E. Automatic tool choice with the review's own tools, validated by dbunk (chosen)

One request shape on both wires, matching the vendor's guidance. dbunk's own validation is the schema guarantee.

## Consequences

- dbunk works with any model and endpoint that supports tool calling with automatic choice and accepts dbunk's tool definitions. That includes the current Claude models that reject forcing.
- Cost: the probe is one request with no retry. A capable model that answers it with text on that one request is refused, and the operator must start the review again.
- Cost: during a review, a text-only or invalid answer costs one corrective retry. Schema validity rests entirely on dbunk's validator.
- Cost: the probe sends the full tool set, so it costs more tokens than a single small tool would.
- The probe asks the model to copy a supplied payload. It therefore predicts the review's request shape, and the calling path: that the endpoint accepts the tool definitions, that the model calls a tool under automatic choice, and that arguments arrive intact. It does not test whether the model can write a good review response itself; dbunk's validator and the corrective retry cover that during the review.
- The provider ADR's Alternative D (Chat Completions only) rejected Anthropic's compatibility layer partly because it ignores `strict`. The reviewer no longer depends on `strict`, so that part of the reason is outdated. D stays rejected on its other reason: the vendor documents the layer as mainly for testing, not production.
- Cost: an approved ADR is partly superseded, and its header changes on acceptance.
- `2026-10-26-reviewer-loop-on-official-sdks` pins the request fields to this shape.

## Confirmation

A later reviewer or agent checks compliance by:

- **No forcing:** no request dbunk sends, on either wire, carries `tool_choice` `any`, `required` or a named tool, or a `strict` field. Every request offering tools carries automatic tool choice explicitly.
- **Same tools:** the probe request's tool definitions equal a review request's tool definitions.
- **Bundled schemas:**
  - no tool definition sent on either wire contains a `$ref` to another file;
  - every bundled schema accepts and rejects the fixtures under `schema/fixtures/` exactly as its source schema does;
  - every inlined `$defs` entry is JSON-equal to the definition it came from;
  - review-response fixtures that fail only on an inlined definition (a malformed claim ID, a malformed packet digest, whitespace-only text) are rejected by the bundled schema;
  - a schema whose inlined definition name clashes with a local `$defs` key fails to bundle.
- **Forced-choice endpoint:** a fake Messages endpoint that answers any forced `tool_choice` with a 400 accepts the probe and a full scripted review.
- **Probe refusals:** each of these fake models or endpoints is refused with `ReviewerModelUnsupported` before any review event is written, and the probe outcome appears in the audit:
  - a model that answers with text only;
  - one that calls a different tool;
  - one that makes two calls;
  - one whose arguments fail the review-response schema;
  - an endpoint that rejects a tool definition.
- **Mixed content:** a fake model that sends text and one valid review-response call passes the probe.
- **Retries in a review:** a fake model that ends a reviewer turn without a sole valid review-response call gets exactly one corrective retry. A second miss fails the turn. The fake endpoints reject any request that leaves a tool call unanswered, and the retry request passes them.

## Anchors

- `2026-10-24-provider-neutral-reviewer-models`: the Capability probe bullet and the Confirmation item, partly replaced here.
- `2026-10-24-reviewer-loop-with-daimonos-tools`: schema validation with one corrective retry, extended here.
- `schema/review-response.schema.json`: the schema features that rule out strict.
- Anthropic documentation:
  - Tool use, "Tool choice": the 400 on forced tool use for these models, and checking that a call was made under `auto`;
  - the migration guides for Claude Opus 5.5 and Claude Sonnet 5.5, "forced tool use is rejected";
  - Structured outputs, "JSON Schema limitations".
- pydantic-ai-slim 2.61.0, `pydantic_ai/profiles/anthropic.py::supports_forced_tool_choice`: the same split, from a third party's live tests.

## Revisit when

A supported wire drops automatic tool choice, or the vendor restores forced tool use on current models.

## Open questions

None.
