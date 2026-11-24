# MagicMtn threat model

## What this project does and where untrusted input enters

MagicMtn is a distributed inference and sandboxed compute platform for AI agents. Agents and their users create sandboxes, run commands and read and write files in them, and route inference to model providers. This repository contains the engine (`core`), the single-user local server (`local`), the `magic` CLI (`cli`), the provider SDK (`sdk/rust`), the runtime that runs inside each sandbox (`runtime`), and the agent integrations for Pi and OpenCode (`interfaces`).

Untrusted input enters in these places:

- **Code and data inside a sandbox.** Sandboxes run commands chosen by an AI agent, which may be following attacker-controlled instructions from a repository, web page, or tool result. Treat everything a sandbox produces as hostile: process output, file contents and names, exit states, terminal escape sequences, WebSocket and stream frames, and metadata reported by the sandbox runtime.
- **HTTP API requests.** The engine serves `/v1/sandboxes`, `/v1/agents`, `/v1/harnesses`, `/v1/models`, `/v1/compute`, `/v1/images`, `/v1/inference-resources`, `/v1/provider-connections`, and `/v1/provider-runners`. Callers are remote clients holding a bearer token, with attacker-controlled paths, headers, query strings, and JSON bodies. An authenticated caller in one organization or workspace is untrusted with respect to every other organization and workspace.
- **Local server requests.** `magic server start` listens on a loopback address and authenticates with a bearer credential stored in the local directory. Other local processes without that credential, and web pages in the user's browser that can send requests to `127.0.0.1`, are untrusted.
- **Provider runners.** Remote runners claim jobs, send heartbeats, and post results through `/v1/provider-runner/*`. A runner is trusted only for the jobs bound to it; its results and claims are otherwise untrusted.
- **Downloaded artifacts.** The CLI downloads release manifests, CLI binaries, provider packages (gzip tar archives), the provider registry index, and container images. The network and the hosting bucket are untrusted; integrity rests on the pinned trust root in `release/cli-trust.json`, signatures, and checksums.
- **Model and provider responses.** Inference responses and streamed events from model providers are untrusted data.
- **MCP and agent clients.** `magic` exposes sandbox tools over MCP and to the Pi and OpenCode integrations. Tool arguments come from a model and are untrusted.

Trusted: the operator who deploys the engine, the user's own account on their machine, the configuration and credential files in `~/.config/magicmtn`, the database, installed provider binaries after their package has been verified, and the cloud control planes the providers call.

## Components that matter most / least

Most important:

- **Sandbox isolation and lifecycle** (`core/src/sandboxes*`, `core/src/sandbox_lifecycle.rs`, `core/src/sandbox_storage*`, `core/src/compute*.rs`, `local/src`). A sandbox must not reach the host, another sandbox, or another tenant's resources through the engine, and must be cleaned up when it expires or is terminated.
- **Authentication, authorization, and tenant scoping** (`core/src/auth.rs`, `core/src/registry_scope.rs`, `core/src/workspace.rs`, `local/src/lib.rs`). Policies are deny-by-default. Every resource lookup must be scoped to the caller's organization and workspace, including when a database connection is reused.
- **Credential handling** (`core/src/credential_vault`, `core/src/provider_connections.rs`, `core/src/redaction.rs`, `cli/src/authentication.rs`, `cli/src/configuration.rs`). Provider credentials, API tokens, and the local server credential must never be returned by the API, written to logs or error reports, or exposed to a sandbox that was not granted them.
- **Provider runner protocol and provider framework** (`core/src/provider_runners.rs`, `sdk/rust/magicmtn-provider-framework`, `sdk/rust/magicmtn-contracts`, `sdk/rust/magicmtn-sdk`). This includes the JSON RPC framing between the host and provider processes, job claiming, and result submission.
- **CLI update and provider installation** (`cli/src/release.rs`, `cli/src/update.rs`, `cli/src/provider_registry.rs`, `scripts/install-public.sh`, `scripts/install-checksum.sh`). This covers signature and checksum verification, manifest lifetime checks, archive extraction, and file permissions of installed binaries.
- **Terminal, file transfer, and streaming paths** (`cli/src/sandbox_shell.rs`, `cli/src/sandbox_scp.rs`, `cli/src/sandbox_scp.py`, `cli/src/pi`, `local/src/shell.rs`). These parse data that comes from a sandbox and act on the user's machine.
- **In-sandbox runtime and agent integrations** (`runtime`, `interfaces/pi`, `interfaces/opencode`, `cli/src/mcp.rs`). These handle the broker, the MCP bridge, GitHub credential helpers, and harness launch.

Less important but in scope: shell completions, uninstall, opt-in error reporting (`cli/src/telemetry.rs`), and the image and provider registry build scripts in `scripts`.

Out of scope: third-party dependencies (report those upstream), `target`, `node_modules`, Bazel build rules and lockfiles, the TLA+ model in `formal`, test fixtures in `tests/fixtures`, and vulnerabilities in the sandbox technologies themselves (Apple Container, microVM monitors, container runtimes, cloud provider services). A flaw in how MagicMtn configures or invokes one of those technologies is in scope.

## How to exercise it

- The workspace is built with debug profiles under `/src/target/debug`. The CLI is `/src/target/debug/magic`.
- Run the Rust tests offline with `cargo test --locked -p magicmtn-cli -p magicmtn-local -p magicmtn-engine --lib --bins --tests -- --test-threads=1`. Tests marked `#[ignore]` need a disposable PostgreSQL database.
- The local server uses bundled SQLite and needs no external service: set `MAGICMTN_CONFIG_DIRECTORY` and `MAGICMTN_LOCAL_DIRECTORY` to scratch directories and run `magic server start --bind 127.0.0.1:PORT`. The credential the CLI uses is written to `connection.json` in the local directory. `local/tests/sandbox.rs` and `cli/tests` show how the test suites drive the API and the CLI.
- `sdk/examples/minimal-provider` is a complete provider built on the SDK, with a conformance test. Use it to exercise the provider framework's JSON RPC host and runner without a real compute backend.
- `cli/tests/*.py` drive the terminal handoff, Pi stream, and shell setup with `MAGICMTN_TEST_MAGIC=/src/target/debug/magic`.
- Run the Pi extension tests with `npm --prefix interfaces/pi test`, the runtime JavaScript tests with `npm --prefix runtime test`, and the runtime Python tests with `/opt/runtime-venv/bin/python -m unittest discover -s runtime/tests`.
- Real sandbox backends (Apple Container, microVMs, AWS) are unavailable inside the scan environment. Use the example provider, and reason about backend invocation from the source.

## How we rate severity

- **Critical:** sandbox escape to the host or to the engine through MagicMtn code; unauthenticated remote code execution or command injection in the engine, local server, or CLI; authentication bypass on the engine API; cross-organization or cross-workspace access to sandboxes, credentials, or agent definitions; disclosure of stored provider credentials or the credential vault's key material; a bypass of release or provider package verification that lets a network attacker install or run code.
- **High:** a sandbox or a malicious remote server executing code or writing arbitrary files on the user's machine through the CLI, file transfer, terminal handoff, MCP bridge, or agent integrations; a web page or unprivileged local process driving the local server without its credential; privilege escalation within an organization beyond the caller's policy actions; a provider runner acting on jobs not bound to it; path traversal during archive extraction; secrets written to logs, error reports, or API responses; SQL injection.
- **Medium:** denial of service of the engine or local server by an authenticated caller or a sandbox, including panics, unbounded memory or disk use, and lifecycle states that leave sandboxes running past their lifetime; terminal escape sequence injection from sandbox output without code execution; information disclosure limited to non-secret metadata of another tenant; time-of-check to time-of-use races on files in the configuration directory by another local user.
- **Low:** issues that need the operator's or the user's own privileges, hardening gaps without a demonstrated impact, and denial of service of the CLI by input the user chose.

Memory-safety bugs in `unsafe` Rust are high at a minimum and critical when reachable from a remote or sandbox-controlled input.

## Anything to leave alone

- Running attacker-chosen commands inside a sandbox is the product's purpose. Report it only when the effect crosses the sandbox boundary.
- The local server is single-user. A process that already runs as the same user and can read `~/.config/magicmtn` is not an attacker.
- `--network default` gives a sandbox internet access by design.
- Do not report missing rate limiting, missing TLS on the loopback listener, or dependency version advisories without a reachable code path in this repository.
- Features gated behind `acceptance-failpoints` and `development-server` exist for testing and are not enabled in releases.
