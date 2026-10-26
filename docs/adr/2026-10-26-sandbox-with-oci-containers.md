# dbunk runs Daimonos and compute in separate OCI containers through the Docker or Podman CLI, with no network, read-only evidence mounts and resource limits

Status: Proposed
Date: 2026-10-26
Deciders: Patrick Clawson
Consulted:
Informed:
Spec: docs/spec.md
Supersedes:
Superseded by:

## Context

`2026-10-24-reviewer-loop-with-daimonos-tools` requires isolation as the backstop if a dbunk gate fails, and leaves the technology to this ADR, as does `2026-10-24-immutable-evidence-bundle-and-read-only-reviewer`. That isolation needs two filesystem views.

- **Daimonos** sees the bundle and pinned checkouts, mounted read-only. It has its own writable state directory outside the review root, used as its home. It sees no scratch area, and has no network and no credentials.
- **`compute`** sees the review root read-only and the scratch area writable. It sees no Daimonos state and has no network. It runs under a wall-clock timeout, memory and process-count limits, and a cap on captured output. A call that exceeds a limit is killed and recorded as failed, with the limit named.

Symlinks and model-supplied paths are resolved against the same view Daimonos gets. The reviewer must run on Linux and macOS (`docs/spec.md`, Product choices).

Daimonos facts at `d6dd298`:

- It speaks MCP over stdio.
- Its Dockerfile is described as being for introspection services, not the recommended deployment. Its `ENTRYPOINT` is `daimonos --mcp -w /workspace`.
- At startup it reads two files from its workspace root:
  - `agent.env`, from which it sets every absent, non-denylisted environment variable;
  - `daimonos.toml`, whose configuration can register tools.

  It reads the same two kinds of file from its configuration directory.
- Its state directory defaults to `$HOME/.daimonos`.

What provides the reviewer's isolation?

## Decision drivers

- Linux and macOS.
- Isolation is enforced by the operating system, not by dbunk's own code.
- Daimonos and `compute` get two different filesystem views, and dbunk resolves paths in exactly the view the tools get.
- Every limit is enforced, and a hit is observable and attributed to the limit that caused it.
- The environment a recomputation ran in can be identified later.
- No credential reaches either environment.
- Evidence cannot configure the tools that read it.

## Decision

We will run Daimonos and each `compute` call in separate OCI containers, started through the `docker` or `podman` command-line interface. Which binary to use is configuration.

### Images

- dbunk builds a Daimonos image from the pinned Daimonos commit, and a `compute` image. The `compute` image's contents are decided in the reviewer plan; its entrypoint is a dbunk-provided wrapper (below).
- Images are referenced by image ID (the local content digest). Each recorded `compute` call carries the image ID and the dbunk commit that built it.
- Publishing images to a registry, which would give digests others can pull, is not decided here.

### The review root and the path map

- The review root is `/review` in both kinds of container. It holds only mount points:
  - `/review/bundle`, the bundle;
  - `/review/repos/<n>`, one per pinned checkout.

  A checkout's own files, including any `agent.env` or `daimonos.toml`, therefore never sit at the workspace root.
- dbunk keeps a **path map** from each container mount point to its host source. It resolves every path and every symlink in container coordinates:
  - it walks each path one component at a time;
  - it reads each link through the map;
  - it interprets an absolute link target as a container path.
- A path or link target that leaves `/review`, including `/state` and `/scratch`, is outside the review root. This mechanism serves both the preparation scan that raises `ReviewRootEscape` and the path gate.

### Isolation flags

Every container gets:

- `--rm`, `--network none`, `--read-only`, `--cap-drop ALL`, `--security-opt no-new-privileges`;
- with Podman, also `--read-only-tmpfs=false`;
- a label `dbunk.review=<review id>`;
- `--memory` with an equal `--memory-swap`, `--pids-limit` and `--cpus`, all from configuration.

**User.** On Docker the container runs as the invoking host user's UID and GID. On rootless Podman it runs with `--userns=keep-id`. The container passes through no environment beyond the variables named below.

### The Daimonos container

- It is long-lived for one review, and dbunk speaks MCP to it over stdio.
- It is started with `--entrypoint daimonos` and the arguments `--mcp -w /review`.
- Each source is bind-mounted read-only at its mount point.
- A tmpfs is mounted at `/state`, with `HOME=/state` and `XDG_CONFIG_HOME=/state/config`, so Daimonos's state and configuration directories are empty and private to this review.
- No scratch area is mounted.

### Each `compute` container

- One container is started per call.
- It gets the same per-source read-only mounts under `/review`.
- The scratch area is mounted read-write at `/scratch`, which is also the working directory. No Daimonos state is mounted.
- The entrypoint is dbunk's wrapper. It runs the command as a child, captures the child's stdout and stderr through pipes, and stops reading at the output cap. When the child exits, the wrapper reads the container's own cgroup counters, `memory.events` `oom_kill` and `pids.events` `max`. It emits one JSON record with:
  - the exit status;
  - the captured output, encoded inside the record so the command cannot forge the record;
  - whether output was truncated;
  - the counter values.

### Limits

| Limit | Enforced by | Recorded as hit when |
| --- | --- | --- |
| Wall clock | dbunk kills the container at the deadline | the deadline passes |
| Output | the wrapper stops reading, and dbunk kills the container | the wrapper reports truncation |
| Memory | the kernel OOM killer | `oom_kill` is above zero, even if a child process rather than the command was killed |
| Processes | the kernel refuses `fork` | `pids.events` `max` is above zero, even if the command then exited 0 |

A call with any limit hit is recorded as failed, with that limit named.

### Scratch and model calls

- **Scratch** is a fresh directory per review under a configurable location. It defaults to `$HOME/.cache/dbunk/scratch`, a path that Docker Desktop and Podman machine both share into their VM by default. It is deleted when the review ends.
- After a crash, dbunk removes every container that carries its review label.
- **Model calls** never run in a container. The model-call channel is the dbunk process on the host, which holds the model credential.

On macOS the containers run in the Docker Desktop or Podman machine VM.

## Alternatives

### A. No sandbox, relying on dbunk's gates — rejected

This is the obvious shortcut. It is rejected by the reviewer-loop ADR, which requires isolation as a backstop.

### B. bubblewrap or nsjail — rejected

These are light, need no daemon, and work unprivileged on Linux. They are rejected because they are Linux-only and do not run on macOS.

### C. bubblewrap on Linux and `sandbox-exec` on macOS — rejected

This is rejected because it means two mechanisms with different semantics to test, and Apple marks `sandbox-exec` as deprecated.

### D. A microVM or user-space kernel (Firecracker, gVisor) by default — rejected

These give stronger isolation than shared-kernel containers. They are rejected as the default because they are Linux-only (Firecracker also needs KVM). gVisor stays available as a container runtime; see Revisit when.

### E. A hosted sandbox service — rejected

This is rejected because evidence would go to another third party, and the service would need credentials of its own.

### F. OCI containers through the Docker or Podman CLI (chosen)

One mechanism on both platforms, with two filesystem views, observable limits, and an image ID that names each environment.

## Consequences

- Cost: users need Docker or Podman. On macOS that means a background VM.
- Cost: starting a container for each `compute` call adds latency. It is not yet measured.
- Cost: dbunk builds two images and maintains the `compute` wrapper. Until images are published, a recorded image ID can only be rebuilt from its source commit, and a rebuild may not reproduce the same ID.
- Cost: containers share the host kernel. Escapes through the kernel are outside what this isolation defends against.
- Cost: limit detection depends on cgroup v2 inside the container.
- Cost: Docker and Podman differ in defaults and user mapping, so both need testing.
- Cost: on macOS, sharing large checkouts into the VM is slower than native access.
- Not decided: the `compute` image's contents, image publication, and which runtime CI uses.

## Confirmation

A later reviewer or agent checks compliance with Docker and Podman on both Linux and macOS:

- **Daimonos container:** a network request fails, a write under `/review` fails, `/scratch` does not exist, and the environment holds no credential variable.
- **Workspace:** Daimonos reports `/review` as its workspace. A checkout carrying `agent.env` or `daimonos.toml` at its own root sets no variable in Daimonos and registers no tool.
- **Cross-mount links:** a relative link in `/review/repos/1` pointing to `../../bundle/<file>` resolves inside the review root. A link pointing to `/state`, to `/scratch` or above `/review` makes preparation fail with `ReviewRootEscape`. The path gate refuses `/review/../state`.
- **`compute` view:** a call can write to `/scratch`, cannot write under `/review`, cannot read `/state`, and cannot open a network connection.
- **Limits:**
  - a call past its deadline is recorded as failed with `timeout`;
  - a call printing past the output cap is recorded with `output`;
  - a call whose child process is OOM-killed is recorded with `memory`;
  - a call that forks past the process limit is recorded with `pids`, even if it exits 0.
- **Records:** every recorded `compute` call carries an image ID and a source commit.
- **Cleanup:** after a review, including one killed mid-way and then cleaned up, no container labelled with its review ID remains, and its scratch directory is gone.
- **Gates disabled:** the reviewer-loop ADR's gates-disabled checks pass inside these containers.

## Anchors

- `2026-10-24-reviewer-loop-with-daimonos-tools`: the Isolation, Computation, Path gate and Symlinks bullets this ADR implements.
- `beardfaceguy/daimonos@d6dd298:Dockerfile`: the existing image, its stdio usage and `ENTRYPOINT`.
- `beardfaceguy/daimonos@d6dd298:src/env_file.rs::load_default`: `agent.env` read from the workspace root and the configuration directory.
- `beardfaceguy/daimonos@d6dd298:src/config.rs::search_candidates`: `daimonos.toml` read from the workspace root.
- `beardfaceguy/daimonos@d6dd298:src/paths.rs::state_dir`: state under `$HOME/.daimonos`.
- `docs/spec.md`: Linux and macOS.

## Revisit when

Users need Windows, a measured `compute` latency makes per-call containers impractical, or the threat model grows to include kernel escapes, which would make gVisor or a microVM the default.

## Open questions

None.
