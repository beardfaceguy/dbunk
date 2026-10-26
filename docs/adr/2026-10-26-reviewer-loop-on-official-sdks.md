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

`2026-10-24-provider-neutral-reviewer-models` requires two wires, Chat Completions and Anthropic Messages, each with a configurable base URL, and never sends sampling parameters. It leaves the client library to this ADR. `2026-10-26-capability-probe-with-auto-tool-choice` requires explicit automatic tool choice on every request that offers tools, and no forced choice. `2026-10-26-share-negotiation-through-extracted-protocol-package` makes the engine Python.

On 2026-10-26 each candidate made a tool call against a local fake endpoint, with no sampling settings. Fields other than `model`, `messages`, `tools` and `system` were recorded.

| Library (version) | Chat Completions | Anthropic Messages |
| --- | --- | --- |
| `openai` 3.37.0 / `anthropic` 1.17.0 | only `tool_choice`, as dbunk set it | `max_tokens` and `tool_choice`, as dbunk set them |
| `langchain-openai` 1.7.2 / `langchain-anthropic` 1.7.6 | `tool_choice` as set, plus `stream: false` | `tool_choice` as set, plus `max_tokens: 4096` by default |
| `pydantic-ai-slim` 2.61.0 (structured-output agent) | `tool_choice: "required"`, `stream: false` | `tool_choice: {"type": "any"}`, `max_tokens: 16384` by default, `stream: false`, at `/v1/messages?beta=true` |

None of them sent `temperature`. The pydantic-ai row reflects its structured-output default. pydantic-ai can also force a named tool through `ModelSettings.tool_choice`. The other facts behind the pydantic-ai rejection come from its source:

- it always calls Messages through `client.beta.messages.create`;
- when a model profile says forcing is unsupported, `_get_tool_choice` silently sends `auto` instead of a requested `required` or named choice, narrowing the tool list for a named one.

The official SDKs have defaults of their own, also read from their source:

- **Retries:** up to 2 automatic retries on connection errors and on 408, 409, 429 and 5xx.
- **Timeout:** 600 seconds.
- **Headers from the environment:** the OpenAI SDK reads `OPENAI_ORG_ID` and `OPENAI_PROJECT_ID` even when a key and base URL are passed, and sends them as headers. Both SDKs merge headers from `OPENAI_CUSTOM_HEADERS` or `ANTHROPIC_CUSTOM_HEADERS`.
- **Telemetry headers:** both send `X-Stainless-*` headers describing the client's operating system, architecture and runtime. The OpenAI SDK also adds per-request `x-stainless-retry-count` and `x-stainless-read-timeout` headers.
- **Tracing:** the `anthropic` SDK resolves OpenTelemetry tracing and trace propagation as on by default. When a tracer provider is present, it records spans through the global provider, which the environment can configure, and it injects `traceparent` and `tracestate` headers into requests. `ANTHROPIC_OPEN_TELEMETRY_TRACES_CONTENT_MODE=content` adds message content to the spans. The `openai` SDK has no tracing.

The OpenAI Agents SDK (`openai-agents` 0.24.0) exports traces to `https://api.openai.com/v1/traces/ingest` unless tracing is disabled, by environment variable or in code. It reaches Messages only through adapter extensions.

rubber_ducky runs its review as a two-node LangGraph graph, whose checkpoint is mirrored to and reconciled against its own SQLite journal. It calls models through LangChain.

Which library runs dbunk's reviewer loop?

## Decision drivers

- Everything dbunk sends to a model endpoint is set by dbunk and checkable in a test: body fields, headers, retries and timeouts. This is the provider ADR's rule against hidden parameters.
- Both wires work on their stable, non-beta endpoints, with a configurable base URL and the request shape `2026-10-26-capability-probe-with-auto-tool-choice` requires.
- Nothing sits between a model's tool call and dbunk's gates.
- Evidence reaches no third party other than the configured model endpoint.
- Few dependencies, and dependencies that change slowly.
- The review log is the durable record. The loop needs no second store.

## Decision

We will write the reviewer loop in Python (3.12 or later) without an agent framework.

### Model clients

- **The SDKs.** The Chat wire uses the `openai` SDK and the Messages wire uses the `anthropic` SDK. Each takes its `base_url` from configuration. On Messages, `max_tokens` comes from configuration too: the Messages API requires it, and it caps output length rather than sampling.
- **One interface.** Each wire sits behind one dbunk interface (`WireClient`, name proposed). The loop never touches an SDK type directly.
- **Set explicitly by dbunk:**
  - `max_retries=0`. The loop decides every retry, and records each one.
  - The request timeout, from configuration.
  - The Anthropic client is constructed with `open_telemetry={"traces": {"enabled": False}, "propagation": False}`, so it records no span and injects no trace header.
  - The full header set. Each client gets an HTTP client whose transport replaces the outgoing headers with dbunk's fixed set. That covers every header the SDK sets per request, headers sourced from the environment (`OPENAI_ORG_ID`, `OPENAI_PROJECT_ID`, any `*_CUSTOM_HEADERS`), `X-Stainless-*`, trace headers, and any header a future SDK version adds.
    - The fixed set is: the credential header, `Content-Type`, `anthropic-version` on Messages only, a dbunk `User-Agent`, and the transport's own `Host`, `Content-Length`, `Accept`, `Accept-Encoding` and `Connection`.
- **Request bodies.** dbunk builds every request body explicitly. Fake-endpoint tests assert the exact body fields and headers of both the probe and a review request on each wire.

### Daimonos and the loop

- **Daimonos.** The `mcp` Python SDK's stdio client talks to the Daimonos MCP server, which runs inside the sandbox.
- **The loop.** dbunk's own code holds the turn limit, the gates, response validation with one corrective retry, and the record of `compute` calls.
- **Durability.** A review interrupted mid-turn restarts that reviewer turn from the last message in the log. There is no mid-turn checkpoint. A restarted turn runs the capability probe again, because the endpoint or its configuration may have changed. Partial `compute` records from the interrupted turn are discarded. If the re-run probe fails, the review stops with `ReviewerModelUnsupported`: a probe record is written to the audit and no further review event is written.

### Not used

LangGraph, LangChain, pydantic-ai and the OpenAI Agents SDK.

## Alternatives

### A. LangGraph and LangChain, as rubber_ducky uses — rejected

This would keep one idiom across the decider's tools. It is rejected for two reasons. The graph and its checkpoint duplicate the review log, which rubber_ducky has to reconcile. And LangChain's wrappers add values of their own, such as `stream: false` and a 4096 `max_tokens` default. dbunk would have to override those and track them across releases, on top of the SDK defaults it already has to override.

### B. pydantic-ai — rejected

It offers provider-neutral models, structured output and MCP support in one library. It is rejected for three reasons:

- it always calls Messages on the beta endpoint, which an aggregator or self-hosted Messages server may not serve;
- it silently sends `auto` in place of a requested forcing choice when its model profile says forcing is unsupported. That is a hidden behaviour of exactly the kind dbunk's request-shape rule forbids;
- its own output retries and `max_tokens` default overlap with what dbunk sets.

### C. The OpenAI Agents SDK — rejected

Its Messages-wire support goes only through adapter extensions. Its tracing to OpenAI is on by default: dbunk could disable it in code, but any path that missed the switch would send evidence to a second third party. That is a fail-open default.

### D. A hand-written stdio JSON-RPC client instead of the `mcp` SDK — rejected

This would avoid `mcp`'s server-side dependencies. It is rejected because dbunk would then own MCP's protocol negotiation and version changes.

### E. Rust, next to Daimonos — rejected

It is rejected because the shared v1.3 rules are a Python package.

### F. Plain Python on the official SDKs and the MCP client (chosen)

dbunk sees and tests every request field, header and retry, and nothing sits between the model and the gates.

## Consequences

- dbunk maintains the loop and two thin wire clients. The provider ADR already accepted the two clients.
- Cost: dbunk overrides SDK defaults (retries, timeout, headers, tracing), and tests have to catch an SDK upgrade that adds a new one.
- Cost: no framework features such as streaming helpers, tracing or graph visualization. dbunk adds what it needs.
- **Cost: dependency weight.** `mcp` 2.3.0 brings server-side dependencies dbunk does not use, including `starlette`, `uvicorn`, `sse-starlette`, `pyjwt[crypto]`, `python-multipart` and `jsonschema`. Both `mcp` and `anthropic` bring `opentelemetry-api`.
- Cost: an interrupted turn is redone, including the probe, and its model calls are paid again.
- Not decided: sync versus async I/O, the CLI shape, and how dbunk is packaged for users.

## Confirmation

A later reviewer or agent checks compliance by:

- **Dependencies.** The reviewer's manifest names `openai`, `anthropic`, `mcp`, pydantic and the protocol package. Neither the manifest nor the lockfile names `langgraph`, any `langchain*`, any `pydantic-ai*` or `openai-agents`.
- **Request bodies.** For each wire, a fake endpoint records the probe and a review request, and their body fields equal the fixed sets for that wire.
  - **Chat:** `model`, `messages`, `tools`, `tool_choice`.
  - **Messages:** `model`, `max_tokens`, `messages`, `tools`, `tool_choice`, plus `system` when present. The path is `/v1/messages`, with no query string.
- **No sampling parameters.** Neither endpoint ever receives `temperature`, `top_p` or `top_k`.
- **Headers.** The request headers equal dbunk's fixed set.
  - Setting `OPENAI_ORG_ID`, `OPENAI_PROJECT_ID`, `OPENAI_CUSTOM_HEADERS` or `ANTHROPIC_CUSTOM_HEADERS` in the test environment changes no request.
  - No `X-Stainless-*` header is sent.
  - With a recording tracer provider installed and `ANTHROPIC_OPEN_TELEMETRY_TRACES_CONTENT_MODE=content` set, no span is recorded and no `traceparent` or `tracestate` header is sent.
- **No automatic retry.** A probe answered with a 5xx produces exactly one request to the endpoint.
- **Gates first.** A model-emitted tool call reaches the tool gate before any MCP request is written. A refused call produces no MCP traffic.
- **Corrective retry.** A schema-invalid reviewer response triggers exactly one corrective retry. A second invalid response fails the turn.
- **Restart.** Killing the loop mid-turn and restarting it re-runs the probe and then the turn from the last logged message. No partial `compute` record survives, and the log holds no duplicate response. If the restarted probe fails, the audit gains a probe record and no further review event.

## Anchors

- `2026-10-24-reviewer-loop-with-daimonos-tools`: the loop's duties and gates.
- `2026-10-24-provider-neutral-reviewer-models`: the two wires, and no sampling parameters.
- `2026-10-26-capability-probe-with-auto-tool-choice`: the request shape the probe and the review use.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/workflow.py::build_review_graph`: the two-node LangGraph graph.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/service.py::_reconcile_graph`: the checkpoint reconciled against the journal.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/adapters.py::ReviewerAdapter`: model calls through LangChain.
- `beardfaceguy/daimonos@d6dd298:src/main.rs::main`: the MCP stdio mode the client speaks to.

## Revisit when

An official SDK stops supporting a custom base URL or explicit header control. A supported wire drops automatic tool choice. Or the loop's own code grows into a general agent framework.

## Open questions

None.
