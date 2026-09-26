# Framing — Skillsmith v2 architecture patterns (2026-09-25)

Trigger: by-hand (user asked to research architecture patterns for v2). Template: best-practice
survey (1) combined with architecture selection (2).

Selected constraints (relevance-filtered):
- Stack: Rust (Cargo workspace), CLI binary `sks` plus embeddable library crate.
- Coexists with TypeScript v1 (smorinlabs/skillsmith) sharing on-disk state byte-compatibly
  (manifest@1, lock@1, plan@1, ledger@1/2, journal@1); v1 locks via proper-lockfile mkdir of
  `placements.json.lock`.
- Security posture binds: adapters run third-party agent binaries; third parties may add
  adapters (data manifest or external JSON-RPC program); credentials must never leak into
  verification runs (v1 Claude Code gap).
- License: Apache-2.0 (dependencies must be compatible).
- Platforms: macOS, Linux; Windows secondary.

Design already agreed (ADR 0001, PR #1): D1 two-axis capability support (agent support x
implementation status) with derived states; D2 adapter tiers (built-in Rust, agent.toml data,
external JSON-RPC program); D3 first-class Locations + Bindings; D4 co-owned shared placements
with reachability GC; D5 shared v1 state + same lock; D8 new CLI output; D11 verify levels
lint/validate/load, lint shared + adapter rules; capabilities implemented inside adapters.
Open: core services vs adapter responsibilities; escape hatches (declared permissions,
extensions map, namespaced custom capabilities); library/CLI split with typed results,
diagnostics and errors as data; progress events.

Light framing sources already gathered this session: LSP/MCP capability negotiation, MDN BCD,
Terraform provider protocol, Nix GC roots, Homebrew keg/opt, Zed/Zellij/Extism plugins, mise
registry, Deno permissions.
