Research the architecture patterns a Rust CLI + embeddable library named Skillsmith v2 should
be designed against. Skillsmith installs, verifies, and syncs "agent skills" (folders with
SKILL.md) across AI coding agents (Claude Code, Codex, OpenCode, Kilo Code, Muse, and third-party
agents added later). Scope: architecture and design patterns only, not library shopping, except
where a Rust crate is the canonical embodiment of a pattern.

Constraints: Rust workspace; Apache-2.0; macOS/Linux first; runs alongside a TypeScript v1 that
shares on-disk state byte-for-byte (versioned JSON/TOML artifacts, mkdir-based lock via the npm
proper-lockfile convention); adapters launch third-party agent binaries, so isolation and
credential stripping are security requirements; third parties can add agents via a data
manifest (agent.toml) or an external JSON-RPC-over-stdio program.

Cover three clusters:
A. Capability-based adapter architecture: ports & adapters / hexagonal; role interfaces and
   capability discovery (COM QueryInterface-style accessors, LSP capability negotiation);
   extension points and contribution models (VS Code, Eclipse, Backstage); plugin boundaries
   (in-process traits, data manifests, out-of-process protocols, WASM); object-capability
   security for services handed to adapters (cap-std, WASI); declared-permission escape hatches
   (Deno, Zellij, Android); data escape hatches (LSP experimental, MCP _meta, typed extension
   maps like http::Extensions); namespaced custom capabilities; Rust idioms (trait objects vs
   enums vs generics, sealed traits, #[non_exhaustive]).
B. Library/CLI separation and results as data: functional core / imperative shell; clean/onion
   architecture use cases; command vs query separation; error and diagnostic models (thiserror
   vs anyhow, miette, rustc/LSP Diagnostic, stable diagnostic codes as in ruff/clippy);
   accumulating non-fatal findings vs fatal errors; progress/event reporting (tracing, observer,
   cargo's Shell abstraction); how cargo, uv, ruff, rustup structure lib vs CLI crates.
C. State, installation, and coexistence: shared install locations with multiple consumers and
   ownership (Nix profiles and GC roots, dpkg file ownership and diversions, update-alternatives,
   GNU stow, Homebrew); plan/apply and journaled transactions (Terraform, package managers);
   cross-language file locking; schema/codec versioning with strict unknown-field rejection;
   strangler-fig migration and parallel run; conformance/TCK test suites for third-party
   implementations (test262, LSP/MCP conformance, Jakarta TCK) and golden tests.

For each cluster: canonical patterns, how exemplary projects apply them, known anti-patterns
and pitfalls, what changed recently (is anything current best practice now considered an
anti-pattern?), and which apply directly to Skillsmith vs need adaptation. Recommend a concrete
set of patterns per cluster with reasoning and a ranked runner-up; conditional recommendations
are welcome. State confidence and its basis for each recommendation, surface source
disagreements, and state assumptions. Cite primary sources (official docs, source code,
specs) with URLs.
