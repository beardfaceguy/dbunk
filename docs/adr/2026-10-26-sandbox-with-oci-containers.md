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

`2026-10-24-reviewer-loop-with-daimonos-tools` requires isolation as the backstop if a dbunk gate fails. It leaves the technology to this ADR, as does `2026-10-24-immutable-evidence-bundle-and-read-only-reviewer`. The isolation needs two views of the filesystem.

**Daimonos** sees:

- the bundle and pinned checkouts, mounted read-only;
- its own writable state directory outside the review root, used as its home;
- no scratch area, no network and no credentials.

**`compute`** sees:

- the review root, read-only;
- the scratch area, writable;
- no Daimonos state and no network.

`compute` also runs under a wall-clock timeout, memory and process-count limits, and a cap on captured output. Symlinks are resolved against the same view Daimonos gets.

The reviewer must run on Linux and macOS (`docs/spec.md`, Product choices).

Daimonos facts at `d6dd298`:

- It speaks MCP over stdio and ships a Dockerfile, described there as for introspection services, not the recommended deployment.
- At startup it reads `agent.env` from the workspace root and from its config directory, and sets every variable it finds that is absent and not on its denylist.
- Its state directory defaults to `$HOME/.daimonos`.

What provides the reviewer's isolation?

## Decision drivers

- Linux and macOS.
- Isolation is enforced by the operating system, not by dbunk's own code.
- Daimonos and `compute` get two different filesystem views.
- Memory, process and time limits are enforced.
- The environment a recomputation ran in can be identified later.
- No credential reaches either environment.
- Evidence cannot configure the tools that read it.

## Decision

We will run Daimonos and each `compute` call in separate OCI containers, started through the `docker` or `podman` command-line interface. Which binary to use is configuration, and both take the flags below.

- **Images.** Images are referenced by digest, never by tag.
  - A Daimonos image is built from the pinned Daimonos commit.
  - A `compute` image's contents are decided in the reviewer plan.
  - Each recorded `compute` call carries the digest of the image it ran in.
- **The review root.** dbunk creates the review root as a directory holding only mount points: `/review/bundle`, and `/review/repos/<n>` for each pinned checkout. A checkout's own files, including any `agent.env`, therefore never sit at the workspace root.
- **The Daimonos container** is long-lived for one review and speaks MCP over stdio. It is started with:
  - `--rm -i --network none --read-only --cap-drop ALL --security-opt no-new-privileges`;
  - a non-root `--user`;
  - the bundle and each checkout bind-mounted read-only under `/review`;
  - a `tmpfs` at `/state`, with `HOME=/state`;
  - `--memory` and `--pids-limit`;
  - working directory `/review`;
  - no environment passed through beyond `HOME` and explicit Daimonos settings.
- **Each `compute` container** is started per call with:
  - the same isolation flags;
  - the review root mounted read-only;
  - the scratch area mounted read-write at `/scratch`, also the working directory;
  - no Daimonos state;
  - `--memory`, `--pids-limit` and `--cpus` from configuration.

  dbunk enforces the wall-clock timeout by killing the container. It captures stdout and stderr and caps them itself.
- **Scratch** is a host temporary directory, created per review and deleted when the review ends.
- **Model calls** never run in a container. The model-call channel is the dbunk process on the host, which holds the model credential.
- **macOS.** On macOS the containers run in the Docker Desktop or Podman machine VM. Bind mounts come from host paths shared into that VM.

## Alternatives

### A. No sandbox, relying on dbunk's gates — rejected

This is the obvious shortcut. It is rejected by the reviewer-loop ADR, which requires isolation as a backstop.

### B. bubblewrap or nsjail — rejected

These are light, need no daemon, and work unprivileged on Linux. They are rejected because they do not run on macOS.

### C. bubblewrap on Linux and `sandbox-exec` on macOS — rejected

This is rejected because it means two mechanisms with different semantics to test, and `sandbox-exec` is deprecated by Apple.

### D. A microVM or user-space kernel (Firecracker, gVisor) by default — rejected

These give stronger isolation than shared-kernel containers. They are rejected as the default because they are Linux-only (Firecracker also needs KVM). gVisor stays available as a container runtime (see Revisit when).

### E. A hosted sandbox service — rejected

This is rejected because evidence would go to another third party, and the service would need credentials of its own.

### F. OCI containers through the Docker or Podman CLI (chosen)

One mechanism on both platforms, with two filesystem views, enforced limits, and an image digest that names each environment.

## Consequences

- Cost: users need Docker or Podman. On macOS that means a background VM.
- Cost: starting a container for each `compute` call adds latency. It is not yet measured.
- Cost: dbunk builds, pins and distributes two images.
- Cost: containers share the host kernel. Escapes via the kernel are outside what this isolation defends against.
- Cost: bind mounts can surface user-ID mismatches. On macOS, sharing large checkouts into the VM is slower than native access.
- Not decided: the `compute` image's contents, where the images are published, and which runtime CI uses.

## Confirmation

A later reviewer or agent checks compliance by running these checks with Docker and Podman on Linux, and Docker on macOS:

- Inside the Daimonos container, a network request fails, a write under `/review` fails, `/scratch` does not exist, and the environment holds no credential variable.
- A `compute` call can write to `/scratch`, cannot write under `/review`, cannot read `/state`, and cannot open a network connection.
- A `compute` call that exceeds its timeout, memory or process limit is killed and recorded as failed, with the limit named.
- Every recorded `compute` call carries an image digest, and every image reference in configuration is a digest.
- The review root's top level holds only `bundle` and `repos`. A checkout with an `agent.env` at its own root sets no variable in Daimonos.
- No container from a finished review remains, and its scratch directory is deleted.
- The reviewer-loop ADR's gates-disabled checks pass inside these containers.

## Anchors

- `2026-10-24-reviewer-loop-with-daimonos-tools`: the Isolation, Computation and Symlinks bullets this ADR implements.
- `beardfaceguy/daimonos@d6dd298:Dockerfile`: the existing image build and its stdio usage.
- `beardfaceguy/daimonos@d6dd298:src/env_file.rs::load_default`: `agent.env` read from the workspace root and the config directory.
- `beardfaceguy/daimonos@d6dd298:src/paths.rs::state_dir`: state under `$HOME/.daimonos`.
- `docs/spec.md`: Linux and macOS.

## Revisit when

Users need Windows, a measured `compute` latency makes per-call containers impractical, or the threat model grows to include kernel escapes, which would make gVisor or a microVM the default.

## Open questions

None.
