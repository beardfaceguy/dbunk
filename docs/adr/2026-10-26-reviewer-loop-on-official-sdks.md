# dbunk's reviewer loop is plain Python on the official OpenAI and Anthropic SDKs and the MCP Python client, with no agent framework

Status: Proposed
Date: 2026-10-26
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

`2026-10-24-reviewer-loop-with-daimonos-tools` makes dbunk own the reviewer's loop. The loop does these things:

- calls the model;
- passes each tool call through dbunk's tool and path gates before it reaches Daimonos over MCP, or runs `compute`;
- limits turns;
- validates the final response against the schema, with one corrective retry.

`2026-10-24-provider-neutral-reviewer-models` adds requirements on the model calls:

- two wires, Chat Completions and Anthropic Messages, each with a configurable base URL;
- a capability probe that forces a call to one named tool;
- no sampling parameters ever sent.

It leaves the client library to this ADR. `2026-10-26-share-negotiation-through-extracted-protocol-package` makes the engine Python.

What each candidate actually sends was measured on 2026-10-26. Each one made a forced tool call with no sampling settings, against a local fake endpoint. Fields other than `model`, `messages`, `tools` and `system` were recorded:

| Library (version) | Chat Completions | Anthropic Messages |
| --- | --- | --- |
| `openai` 3.37.0 / `anthropic` 1.17.0 | `tool_choice` naming the tool | `max_tokens` (set by the caller), `tool_choice` naming the tool |
| `langchain-openai` 1.7.2 / `langchain-anthropic` 1.7.6 | adds `stream: false` | `max_tokens` defaults to 4096 |
| `pydantic-ai-slim` 2.61.0 | `tool_choice: "required"`, not the named tool | `tool_choice: {"type": "any"}`, sent to the beta endpoint `/v1/messages?beta=true` |

None of them sent `temperature`. The OpenAI Agents SDK (`openai-agents` 0.24.0) exports traces to `https://api.openai.com/v1/traces/ingest` unless `OPENAI_AGENTS_DISABLE_TRACING` is set.

rubber_ducky runs its review as a two-node LangGraph graph whose checkpoint is mirrored to, and reconciled against, its own SQLite journal. Its model calls go through LangChain.

Which library runs dbunk's reviewer loop?

## Decision drivers

- Every field dbunk sends to a model endpoint is set by dbunk and checkable in a test. This is the provider ADR's rule against hidden parameters.
- Both wires support a named forced tool call and a configurable base URL, on their stable, non-beta endpoints.
- Nothing sits between a model's tool call and dbunk's gates.
- Evidence reaches no third party other than the configured model endpoint.
- A public tool with a small, slow-moving dependency set.
- The review log is the durable record. The loop needs no second store.

## Decision

We will write the reviewer loop in Python (3.12 or later) without an agent framework.

- **Chat wire.** The `openai` SDK, with `base_url` from configuration.
- **Messages wire.** The `anthropic` SDK, with `base_url` from configuration. `max_tokens`, which the Messages API requires, comes from configuration too. It caps output length and is not a sampling parameter.
- **One interface.** Each wire sits behind one dbunk interface (`WireClient`, name proposed). The loop never touches an SDK type directly.
- **Daimonos.** The `mcp` Python SDK's stdio client talks to the Daimonos MCP server, which runs inside the sandbox.
- **The loop.** dbunk's own code holds the turn limit, the gates, response validation with one corrective retry, and the record of `compute` calls.
- **Request bodies.** dbunk builds every request body explicitly. A fake-endpoint test asserts the exact fields of the probe and of a review request on each wire.
- **Durability.** A review interrupted mid-turn restarts that reviewer turn from the last message in the log. There is no mid-turn checkpoint.

LangGraph, LangChain, pydantic-ai and the OpenAI Agents SDK are not used.

## Alternatives

### A. LangGraph and LangChain, as rubber_ducky uses — rejected

This would keep one idiom across the decider's tools. It is rejected for two reasons:

- The graph and its checkpoint duplicate the review log. rubber_ducky has to reconcile them.
- LangChain's wrappers choose values dbunk must then override and track across releases. The measured example is a 4096 default for `max_tokens`.

### B. pydantic-ai — rejected

It offers provider-neutral models, structured output and MCP support in one library. It is rejected because of what it sent:

- it forced "any tool" rather than the named probe tool;
- on Messages it called a beta endpoint, which an aggregator or self-hosted Messages server may not serve.

Its own output retries also overlap with dbunk's corrective retry.

### C. The OpenAI Agents SDK — rejected

It is rejected because tracing to OpenAI is on by default, which would send evidence to a second third party unless every deployment disables it. Its Messages-wire support also goes through an adapter.

### D. Rust, next to Daimonos — rejected

It is rejected because the shared v1.3 rules are a Python package.

### E. Plain Python on the official SDKs and the MCP client (chosen)

dbunk sees and tests every request field, and nothing sits between the model and the gates.

## Consequences

- dbunk maintains the loop and two thin wire clients. The provider ADR already accepted the cost of two clients.
- Cost: no framework features such as streaming helpers, tracing or graph visualization. dbunk adds what it needs.
- Cost: SDK upgrades can change request bodies. The fake-endpoint tests catch that.
- Cost: an interrupted turn is redone, and its model calls are paid again.
- Not decided: sync versus async I/O, the CLI shape, and how dbunk is packaged for users.

## Confirmation

A later reviewer or agent checks compliance by:

- The reviewer's dependency manifest names `openai`, `anthropic`, `mcp`, pydantic and the protocol package. It does not name `langgraph`, any `langchain*`, any `pydantic-ai*` or `openai-agents`.
- A fake Chat Completions endpoint receives, for the probe, exactly `model`, `messages`, `tools` and `tool_choice` naming the probe tool.
- A fake Messages endpoint receives, for the probe, exactly `model`, `max_tokens`, `messages`, `tools`, `tool_choice` naming the probe tool, and optionally `system`, at `/v1/messages` with no query string.
- Neither endpoint ever receives `temperature`, `top_p` or `top_k`.
- A model-emitted tool call reaches the tool gate before any MCP request is written. A refused call produces no MCP traffic.
- A schema-invalid reviewer response triggers exactly one corrective retry, and a second invalid response fails the turn.

## Anchors

- `2026-10-24-reviewer-loop-with-daimonos-tools`: the loop's duties and gates.
- `2026-10-24-provider-neutral-reviewer-models`: the two wires, the probe, and no sampling parameters.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/workflow.py::build_review_graph`: the two-node LangGraph graph.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/service.py::_reconcile_graph`: the checkpoint reconciled against the journal.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/adapters.py::ReviewerAdapter`: model calls through LangChain.
- `beardfaceguy/daimonos@d6dd298:src/main.rs::main`: the MCP stdio mode the client speaks to.

## Revisit when

An official SDK stops supporting a named forced tool call or a custom base URL, or the loop's own code grows into a general agent framework.

## Open questions

None.
