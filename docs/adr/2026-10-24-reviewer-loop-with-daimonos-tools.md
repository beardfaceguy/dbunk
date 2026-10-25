# dbunk runs its own reviewer agent loop and gets read-only tools from Daimonos over MCP

Status: Proposed
Date: 2026-10-24
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

dbunk will have its own reviewer agent. A reviewer that can only read the text it is handed can check structure but not truth; the manual review of the motivating packet (`docs/spec.md`) needed to open repositories and SDK source to reach its findings. rubber_ducky's automatic `ReviewerAdapter` has no tools by design.

Daimonos provides compact read tools (`read_file`, `search`, `workspace_info`), an MCP server mode (`daimonos --mcp -w <dir>`), a one-shot agent mode, multi-provider routing including OpenRouter, and a per-tool-name allowlist for agent mode (`DAIMONOS_AGENT_ALLOWED_COMMANDS`). At d6dd298 its MCP server does not enforce read-only on its own:

- `tools/list` shows an initial tier of tools, but the server runs any registry tool it is sent by name, listed or not.
- `batch` and Starlark `execute_script` dispatch inner tool calls, including `write_file`, `edit_file` and `exec`, with no allowlist gate.
- Paths are not confined: absolute paths pass through and relative paths (including `..`) are joined to the working directory, so reads can reach any file the process can read.
- Its `git` plugin bundles reads with `commit`, `push`, `checkout` and a `raw` passthrough.

Who owns the reviewer's agent loop, where do its tools come from, and where is read-only enforced?

## Decision drivers

- Read-only and evidence confinement are enforced by dbunk, not assumed from another program's settings.
- Reviewer output is schema-valid, with a corrective retry.
- Unattended runs are possible.
- Tool output stays compact; reviewers reopen a lot of evidence.
- Other conforming reviewers (for example, a harness subagent) remain possible.

## Decision

We will implement dbunk's reviewer as a dbunk-owned loop that uses Daimonos tools over MCP, with enforcement in two layers.

- **Model calls:** dbunk calls the model (`2026-10-24-provider-neutral-reviewer-models`), validates each response against the review-response schema, and retries once with the validation errors, as rubber_ducky does.
- **Tool gate (dbunk):** the reviewer's tool allowlist is the Daimonos tools `read_file`, `search`, `ls` and `workspace_info` (plus `git_read` once it exists), and one dbunk-provided `compute` tool (name proposed). dbunk shows the model only these. Before forwarding to Daimonos, it refuses every call whose tool name is not on the list. `batch`, `execute_script`, `exec` and `set_cwd` are never on it. The allowlist is explicit and not derived from Daimonos's destructive-tool classification.
- **Computation (dbunk):** `compute` runs a command inside the sandbox below with its working directory in the ephemeral scratch area. It has read-only access to the review root, write access only to scratch, and no network. Each call runs under a wall-clock timeout, memory and process-count limits, and a cap on captured output; a call that exceeds a limit is killed and reported as failed. Every call is recorded with the reviewer response: the command, exit code, whether a limit was hit, and the SHA-256 of its output, so a later reader can replay the recomputation. The limit values come from configuration. `compute` is how the reviewer recomputes derived claims, as `2026-10-24-immutable-evidence-bundle-and-read-only-reviewer` permits. Daimonos `exec` stays excluded.
- **Path gate (dbunk):** for each allowlisted Daimonos tool, dbunk keeps the list of path-bearing arguments taken from the pinned Daimonos tool schemas. It refuses any call carrying an argument it does not recognize. It resolves relative paths against the review root (the bundle and the pinned checkouts); this matches Daimonos's working directory only because `set_cwd` is excluded. It canonicalizes every path, resolving symlinks, and refuses any path outside the review root.
- **Symlinks (dbunk):** when dbunk prepares the review root, it resolves every symlink inside the bundle and checkouts. Any symlink whose target lies outside the review root, is relative and resolves outside it, or cannot be resolved (a dangling link) makes preparation fail with a named error (`ReviewRootEscape`, name proposed). Links are resolved against the same filesystem view Daimonos gets inside the sandbox, so a link cannot resolve to a sandbox-only path such as the state directory. Daimonos `search` follows symlinked files, so an argument-path check alone does not confine it.
- **Isolation (environment):** Daimonos runs in an isolated environment that mounts only the bundle and pinned checkouts, read-only; gives Daimonos its own writable state directory and the reviewer's ephemeral scratch area; has no network; and carries no credentials in its environment. This is the backstop if a dbunk gate fails. The sandbox technology is not decided here.
- **Runtime contract:** what any reviewer receives and may do is owned by `2026-10-24-immutable-evidence-bundle-and-read-only-reviewer`. The review-response schema it returns (v1.3 Review Response plus dbunk's claim references and verification records) is owned by the protocol document under `2026-10-24-verification-layer-on-rubber-ducky-negotiation`. dbunk's loop is one conforming implementation.
- **Version pin:** dbunk pins a Daimonos commit or release and runs its tool-gate tests against it.

Prerequisites before the reviewer ships:

1. Daimonos gains a read-only git tool (`git_read`), or dbunk provides each pinned revision as a checkout so `read_file` suffices.
2. The sandbox is chosen and provides the isolation above, including network-isolated computation for recomputing derived claims.
3. The tool gate, path gate and isolation pass the Confirmation checks against the pinned Daimonos version.

## Alternatives

### A. Reviewer runs only as the calling harness's subagent — rejected as the only path

The cheapest start. Rejected as the only path because it rules out unattended runs and reviewer behavior varies by harness. It remains a conforming alternative under the runtime contract.

### B. Shell out to `daimonos agent` with an allowlist — rejected

Least code, and it brings Daimonos's retries, failover and caching. Rejected because one-shot output is free text that dbunk must parse and re-prompt, enforcement moves outside dbunk, and dbunk couples to the Daimonos CLI.

### C. A minimal confined read-only file server instead of Daimonos — rejected for now

Once dbunk gates tools and paths itself, four confined read tools are cheap to build or adopt. Rejected for now because Daimonos's compact, deduplicated output (read caching and bounded tool output) is expected to reduce reviewer tokens on evidence-heavy reviews. That saving is not yet measured (see Revisit when).

### D. dbunk-owned loop with Daimonos tools, gated by dbunk and isolated (chosen)

dbunk controls tool exposure, paths and output validation while reusing proven tools.

## Consequences

- dbunk owns agent-loop code: tool calling, tool and path gates, turn limits, retries.
- Cost: dbunk depends on an unversioned MCP surface in a personal repository; tool names, tiers and schemas can change, so dbunk pins a version and tests against it.
- Cost: Daimonos's "read" tools still keep state (read cache, index, analytics), so the sandbox needs a writable state directory outside the evidence mounts.
- The three prerequisites gate the reviewer's first release.
- Not decided: the agent-loop library (LangGraph or another), the sandbox technology, and how the negotiation layer is shared. Each gets its own ADR before the reviewer plan. The last is the follow-up ADR that `2026-10-24-defer-shared-negotiation-library` requires before any code validates v1.3 responses, which this loop does. The library ADR must stay consistent with the LangGraph question there.

## Confirmation

A later reviewer or agent checks compliance by:

- The tool list sent to the model equals the allowlist above.
- A model-emitted call to `write_file`, `exec`, `execute_script`, or a `batch` containing `write_file` is refused by dbunk, and Daimonos receives no request.
- `read_file`, `search` and `ls` on an absolute path outside the review root, on `../`, or on a symlink escaping it are refused by dbunk, as is a call carrying an unrecognized argument.
- Preparing a review root whose bundle or checkout contains a symlink to a target outside the root, a relative link that resolves outside it, or a dangling link fails with `ReviewRootEscape`, so no in-root `search` or `ls` can return content or metadata from an out-of-root target.
- `compute` can write to scratch, cannot write to the review root, and cannot open a network connection.
- A `compute` command that runs past its timeout, or exceeds its memory or output limit, is killed and recorded as failed with the limit named.
- Every `compute` call in a recorded review has its command, exit code and output digest in the review response.
- With the dbunk gates disabled in a test, a write into the evidence mounts fails and a network request fails inside the sandbox.
- Every recorded reviewer response validates against the review-response schema.
- An integration test runs the reviewer against a workspace served by the pinned `daimonos --mcp`.

## Anchors

- `beardfaceguy/daimonos@d6dd298:src/mcp.rs::dispatch_tool_inner` — runs any named registry tool and activates it if unlisted.
- `beardfaceguy/daimonos@d6dd298:src/tools.rs::initial_exposed_tools` — `tools/list` shows an initial tier, not every tool.
- `beardfaceguy/daimonos@d6dd298:src/mcp.rs::handle_call_tool_request` (the `batch` branch) and `src/script.rs::dispatch_tool_by_name` — inner tool calls with no gate.
- `beardfaceguy/daimonos@d6dd298:src/session.rs::resolve_path` — no path confinement.
- `beardfaceguy/daimonos@d6dd298:src/ops/file_ops.rs::grep` — walks the tree and follows symlinked files (test `grep_through_symlink`).
- `beardfaceguy/daimonos@d6dd298:src/safety.rs::SafetyPolicy`, `::DESTRUCTIVE_TOOLS` — agent-mode gating by tool name; the destructive list omits tools such as `curl`.
- `beardfaceguy/daimonos@d6dd298:src/plugins/git.rs::GitPlugin` — reads and writes share one tool.
- `beardfaceguy/rubber_ducky@0ed4576:src/rubber_ducky/core/adapters.py::ReviewerAdapter`, `::_participant_messages` — the no-tools reviewer and its corrective retry.

## Revisit when

Daimonos ships a supported, versioned library or MCP contract with built-in confinement, or a measured comparison of reviewer token use on a reference packet, with and without Daimonos, shows its saving does not outweigh the cost of the gates and sandbox.

## Open questions

None.
