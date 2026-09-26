---
decided: 2026-09-26
status: decided            # Q16.A: adopted into ADR 0001 D2, D5, D12
---

# Decision: which architecture patterns should Skillsmith v2 design against?

## Choice

Adopt the three recommended pattern sets together (conditional items noted):

**A. Adapters and capabilities** (confidence medium; leaf: capability-adapter-patterns)
1. Base `AgentAdapter` role plus per-capability accessors returning `Option<&dyn Cap>`; the set
   of answers is static per adapter and checked against D1 by the conformance suite.
2. Plan / execute / analyze for every capability that runs an agent binary: the adapter returns
   an `ExecPlan` as data, the core runs it in a sandbox it owns, the adapter analyzes the output.
3. P01-T02 boundary proposal: Tier 1 data can express one argv template, an env allowlist, a
   timeout, and exit-code/regex analysis; anything with branching, structured parsing, or
   multiple runs is code (Tier 0 Rust or Tier 2 JSON-RPC).
4. Declared `exec` permissions for display and drift checks; the credential boundary is the
   core's environment scrub (allowlist) and temporary `HOME`, not the permission list.
5. Namespaced escape hatches: `extensions` maps with reverse-DNS keys (MCP `_meta` syntax) and
   custom capability ids in the same syntax.
Rust idioms: sealed `AgentAdapter` with three tier wrappers; unsealed `BuiltinHooks` for
built-ins; `#[non_exhaustive]`; synchronous traits.

**B. Core library, CLI, and results as data** (confidence high; leaf: rust-core-cli-result-model)
- Functional core, thin CLI shell. Use cases return `Result<Outcome<T>, Fatal>`; each check
  yields a `CheckResult` (`Ran` / `Skipped` / `CouldNotRun`), so no check short-circuits another.
- Serde `Diagnostic` with stable namespaced codes (`spec/...`, `agent/<id>/...`) and an origin.
- thiserror in the core, anyhow only in the CLI.
- Typed event `Sink` for progress (the D7 event interface); `tracing` for logs only.
- Schema-stamped, additively evolving `--json` output; exit codes 0 (ok) / 1 (problems) /
  2 (could not run).
- Boundary enforced by the dependency graph plus clippy lints and `disallowed-methods` for
  `std::io::stdout` / `stderr` in the core crate (clippy's `print_stdout` alone misses
  `writeln!(std::io::stdout())`, verified).

**C. State, installation, coexistence** (confidence medium; leaf: shared-install-state-patterns)
- Ledger root sets with reachability GC for co-owned placements (Nix GC roots).
- Scan-then-write collision planning with sticky manual overrides and foreign-file records
  (GNU stow, dpkg).
- Preimage compare-and-swap per file with the shared `journal@1` (Terraform stale-plan check).
- An in-house mkdir lock compatible with v1's `proper-lockfile@4.1.2`, covering all three v1
  lock families with v1's exact parameters (cross-locking verified locally).
- Strict serde codecs proven by byte-exact `insta::glob!` fixtures.
- One conformance suite over the adapter registry, checks tagged by D1 capability, with
  per-adapter expected-failure baselines (test262 `features`, MCP conformance).
- Read-only parallel run of v1 and v2 before any v2 write.

## Why

Each set answers a constraint the owner stated: capabilities live in adapters without trusting
adapters with isolation (v1's Claude Code credential gap); the library returns only data, with
warnings and errors as part of the interface; v1 and v2 share state safely (D5).

## Runner-up (ranked)

- A: protocol-everywhere (built-ins also speak the JSON-RPC schema in-process). Switch if Tier 2
  adapters dominate.
- B: ruff's single `Diagnostic` model without `CheckResult`. Switch only if `verify` stays the
  only multi-check command.
- C: OS `File::lock` for v2-only files. Switch after v1 retires.

## Why not the others

- Single fat adapter trait returning `NotSupported`: v1's boolean again; hides capabilities.
- `Box<dyn Any>` capability registry: loses typing.
- WASM adapters now: sandboxes adapter code, not the agent binary, which is the actual risk.
- miette as public type; error-stack (API churn); tracing as the results channel (lossy).
- Rendering inside the core (cargo `Shell` style): blocks embedding.
- stow-style inferred ownership; protobuf-style unknown-field preservation; RFC 8785 JCS;
  dual-executing writes; lock force-unlock without a holder nonce.

## Findings that change ADR 0001

- (Resolved in ADR 0001 D5 on 2026-09-26.) D5 originally named one lock; v1 has three lock families. The `proper-lockfile` README states a 5000 ms
  minimum `stale`, but the 4.1.2 source clamps at 2000 ms.
- D2 cites "the LSP/MCP `initialize` pattern"; MCP's 2026-07-28 spec removed the handshake (LSP
  3.17 keeps it). D2's WASM rationale ("toolchain still stabilising") is dated: WASI 0.3.0
  shipped 2026-06-11.
- Open gap: v1's Codex load check is a JSON-RPC conversation over stdin and Muse's runs two
  commands; `ExecPlan` as sketched covers one argv run, so it needs stdin/scripted exchange and
  multi-step support, or those stay Tier 0/2 code.

## Chain
- Prompt: prompts/00-landscape.prompt.md
- Framing: prompts/00-landscape.framing.md
- Output: 01-adapter-capability-patterns.md, 02-core-cli-results-patterns.md,
  03-state-install-coexistence-patterns.md
- Leaves: ../../reference/capability-adapter-patterns-2026-09-25.md,
  ../../reference/rust-core-cli-result-model-2026-09-25.md,
  ../../reference/shared-install-state-patterns-2026-09-25.md
