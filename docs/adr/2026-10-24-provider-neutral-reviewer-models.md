# dbunk reaches reviewer models over Chat Completions or Anthropic Messages, with wire, base URL, credential, provider and model all set explicitly

Status: Proposed
Date: 2026-10-24
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

The decider reports that their team routes model traffic through OpenRouter and switches models often (requester context). The motivating investigation (`docs/spec.md`) saw 20 distinct model identifiers on one API key in 30 days. That shows many models in use, not necessarily switching. The same investigation was evaluating a move of some workloads to a self-hosted model.

Model endpoints speak one of two widely used wires:

- **OpenAI-compatible Chat Completions with a configurable base URL.** OpenRouter, many direct providers and common self-hosted servers serve it.
- **Anthropic Messages.** Anthropic serves it directly, and some aggregators serve it under `anthropic/` model slugs.

Structured-output and tool-calling support varies by model and endpoint, sometimes for the same model on different endpoints.

rubber_ducky selects its reviewer by explicit provider and model, with no default for either, taken together from one precedence source. It records provider and model in its audit and names its credential by environment variable, with a generic default. Its reviewer has no tool calling, so it is precedent for selection and audit only.

How does dbunk choose and call the reviewer's model?

## Decision drivers

- Switching models is configuration, not code.
- Self-hosted and aggregator endpoints work without a code change.
- The protocol and schemas stay free of provider-specific fields.
- Every review records which model produced it.
- A model that cannot do what the reviewer needs fails before the review starts, not mid-review.

## Decision

We will reach reviewer models over two wires, and the configuration names which one to use:

- **`chat`:** OpenAI-compatible Chat Completions with a configurable base URL. This covers OpenRouter, direct OpenAI-compatible providers, and self-hosted endpoints that serve that wire.
- **`messages`:** Anthropic Messages with a configurable base URL. This covers direct Anthropic accounts and aggregators that serve Messages.

Selection and checks:

- **Explicit selection.** The reviewer is selected by explicit `provider`, `model`, `base_url`, `wire` and `api_key_env`, with no built-in default for any of them. All come together from one source: an explicit argument, the environment, or a config file. The wire is never inferred from the model name. `provider` is an operator-chosen label recorded in the audit; it does not choose the wire, the endpoint or the credential. dbunk re-implements this selection pattern from rubber_ducky and does not import it (`2026-10-24-defer-shared-negotiation-library`).
- **Credentials.** `api_key_env` names the environment variable holding the credential. It has no default. Credential values never appear in config, audit or packets.
- **OpenRouter as the shipped example.** OpenRouter is the provider in the shipped example configuration and documentation. There is no runtime fallback to it.
- **Audit:** each reviewer response records `provider`, `model`, `wire` and the capability-probe outcome.
- **Capability probe:**
  - Before writing any review event, dbunk sends one probe request that forces a tool call and requires a schema-conforming structured response, using the most deterministic settings the endpoint accepts.
  - If the probe fails, the review does not start and dbunk raises a named error (`ReviewerModelUnsupported`, name proposed).
  - There is no metadata lookup, no override and no retry.
  - The probe behaves the same for aggregator, direct and self-hosted endpoints.

The client library is not decided here.

## Alternatives

### A. One provider's SDK — rejected

The obvious shortcut. Rejected because every model switch outside that provider becomes a code change.

### B. OpenRouter only, hardcoded — rejected

Rejected because it blocks direct provider accounts, other aggregators and self-hosted endpoints.

### C. A multi-provider wrapper library as the contract (as rubber_ducky does with LangChain) — rejected

A wrapper may still implement the two wires as the client. Rejected as the contract because provider support and capability behavior would then follow the library's release cycle, while every target here already speaks one of two documented wires.

### D. Infer the wire from the model name — rejected

Rejected because `anthropic/` is an aggregator slug convention: direct Anthropic model IDs carry no prefix, and an aggregator may serve Claude over Chat Completions. The model string would silently choose the protocol.

### E. Delegate model calls to Daimonos — rejected

Rejected by `2026-10-24-reviewer-loop-with-daimonos-tools`: dbunk owns the loop and its output validation.

### F. Chat Completions or Anthropic Messages, everything set explicitly in configuration (chosen)

Reaches aggregator, direct and self-hosted endpoints by configuration alone.

## Consequences

- Reviewer features are limited to what both wires support.
- Cost: one extra model call per review start for the capability probe.
- Cost: endpoints that speak neither wire need an adapter and a new ADR.
- Cost: operators must set five values; no setting is guessed.
- Cost: when an aggregator is used, evidence that may include private source passes through a third party with per-upstream retention terms; direct or self-hosted endpoints avoid that.
- Cost and latency must be read per provider.
- Not decided: the client library (tied to the agent-loop ADR named in `2026-10-24-reviewer-loop-with-daimonos-tools`).

## Confirmation

A later reviewer or agent checks compliance by:

- `grep -riE 'openrouter|anthropic|openai' PROTOCOL.md schema/` returns nothing. Generic `provider`, `model` and `wire` audit fields are allowed, and worked examples under `examples/` are excluded.
- Starting a review with no configured provider, model, base URL, wire or `api_key_env` fails with a named error, and no example or default fills it in.
- The shipped example configuration names OpenRouter.
- A direct Anthropic model ID (no `anthropic/` prefix) with an Anthropic base URL and wire `messages` is called over Messages, and an `anthropic/*` slug with wire `chat` is called over Chat Completions: the model name never selects the wire.
- The same scripted review runs through a fake Chat Completions endpoint and a fake Anthropic Messages endpoint with no protocol or schema difference. The audit metadata differs only in `provider`, `model`, `wire` and the probe outcome.
- A fake model that returns no tool call to the probe, and one that returns a tool call with schema-invalid output, are both refused with `ReviewerModelUnsupported` before any review event is written, and the probe outcome appears in the audit.

## Anchors

- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/reviewer_config.py::ReviewerModelConfig`, `::ReviewerModelFactory` — explicit provider/model selection, credential named by environment variable.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/adapters.py::ReviewerAdapter` — rubber_ducky's reviewer uses structured output but no tool calling.
- `docs/spec.md` — the motivating investigation.

## Revisit when

The decider's team standardizes on a single provider, a target endpoint speaks neither wire, or a self-hosted model becomes the primary reviewer.

## Open questions

None.
