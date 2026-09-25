# ADR 0001 — Skillsmith v2 architecture

- **Status:** Proposed (direction approved 2026-09-24; see "Decision log")
- **Date:** 2026-09-24
- **Supersedes:** none. **Relates to:** skillsmith (v1) ADR 0007 (tool adapter registry),
  ADR 0008 (wire contract registry), v1 issue #99, v1 project P21.

## Context

Skillsmith v1 (TypeScript, [smorinlabs/skillsmith](https://github.com/smorinlabs/skillsmith))
installs, verifies, and syncs agent skills (`SKILL.md` folders) across Claude Code, Codex, Kilo
Code, OpenCode, and Muse. v2 is a Rust rewrite that runs alongside v1. Two problems motivate a
second-generation architecture rather than a port:

1. **Commands vs. agents.** A command such as `verify` should automatically serve every agent
   that has the capability it needs, and report why others are excluded. v1 records support as
   `supported: boolean` plus remediation text (`packages/core/src/agents/adapter-types.ts:32`),
   which cannot distinguish "Skillsmith has not built this" from "the agent cannot do this".
2. **Shared install locations.** Several agents read, and sometimes write, the same directory.
   v1 models roots per agent only. At project scope Codex and Muse both write
   `<project>/.agents/skills`; v1 deferred Muse project installs rather than arbitrate
   ownership (v1 issue #99, `packages/core/src/agents/muse/placement.ts:16`).

Evidence gathered on 2026-09-23 from v1 code, ADRs, and git history:

- v1's registry (ADR 0007) removed runtime agent-name branching, enforced by static gates.
- Onboarding cost remains high: adding Muse took two PRs touching 100+ files, of which about 15
  were Muse-owned. The rest were five hand-maintained agent-ID unions
  (`artifacts/plan-types.ts:7`, `artifacts/ledger-types.ts:4`, `contracts/v1/index.d.ts`,
  `contracts/v2/index.d.ts`, `undo/types.ts:23`), CI tool setup, and per-agent golden fixtures.
- v1 persisted codecs reject unknown fields recursively (ADR 0008).
- Agents' documented discovery rules are richer than v1 models: Codex and OpenCode walk from the
  working directory up to the repo root; `OPENCODE_CONFIG_DIR` is additive, not a replacement;
  Muse reads legacy roots behind a rollout gate; duplicate-name handling differs per agent
  (Claude Code: strict precedence; Codex: duplicates coexist; OpenCode: names must be unique;
  Kilo Code: project shadows user).

## Decisions

### D1 — Capability support is two axes; the displayed state is derived

Each (agent, capability) pair records two independent facts:

- **Agent support** (what the agent can do), shaped after MDN browser-compat-data:
  version added, version removed, required gates (env var, rollout flag), notes. Unknown
  versions are explicit.
- **Implementation status** (what Skillsmith has built): done, partial with named gaps, or not
  yet with a tracking reference.

Plus a **contract version** for Skillsmith's own capability contract.

| Agent support | Implementation | Derived state |
|---|---|---|
| yes (from version x) | done | `Supported { since }` |
| yes | partial | `Partial { missing }` |
| yes | not yet | `NotImplemented { tracking }` |
| no | any | `Unsupported { reason }` |
| unknown | any | `Unknown` — resolved by a local probe |
| not declared | any | `NotDeclared` — dispatched as unsupported, displayed distinctly |

Commands declare required capabilities; a dispatcher resolves eligible agents. No command names
an agent.

### D2 — Three adapter tiers behind one `AgentAdapter` trait

- **Tier 0, built-in:** Rust modules compiled into the binary.
- **Tier 1, data manifest:** `agent.toml` declaring locations, detection, and the D1 support
  table. Executes no code.
- **Tier 2, external executable:** `skillsmith-agent-<name>` on `PATH`, speaking JSON-RPC over
  stdio with an `initialize` capability handshake (LSP/MCP pattern).

Built-in agents are themselves expressed as a Tier 1 manifest plus Rust code for what data
cannot express. Tier 2's protocol is language-neutral, so TypeScript v1 could load the same
third-party adapters. WASM plugins (Extism / component model) are a possible later tier for
sandboxing; not adopted now because the 2026 toolchain is still stabilising.

**Open (P01-T02):** the exact data/code boundary. Candidates from v1: inventory collision
resolution (likely data), deep verification and Muse enablement probing (likely code).

### D3 — Locations are first-class; agents bind to them

A `Location` is a scoped pattern: fixed path, env-overridable path, additive path, or
walk-up-to-repo-root. An agent relates to a location through a `Binding` carrying `role`
(canonical write, read, compat read), `precedence`, `collision_policy`, and `gates`.

Install planning selects the minimal set of write locations covering the requested agents and
reports incidental visibility, for example:

```
install foo --agents codex,muse        (project scope)
  write  .agents/skills/foo            codex: canonical write; muse: canonical write
  also visible to: opencode, kilo-code (compat read)
```

### D4 — Shared write locations are co-owned

The placement ledger records every agent a placement serves. Uninstalling for one agent removes
that agent from the record; files are deleted only when no agent still references the
placement (reachability, as in Nix GC roots, not counters). This resolves v1 issue #99.

### D5 — v1 and v2 share state; v2 matches v1 formats exactly

v2 reads and writes v1's data directory (`$SKILLSMITH_HOME` or `$XDG_DATA_HOME/skillsmith`) and
persisted artifacts byte-compatibly: manifest@1, lock@1, plan@1, ledger@2, journal@1. v2 also
reads ledger@1 so it can migrate older ledgers, matching v1. v2-only
data goes in new, separate files. Consequence: no shared format changes until v1 is retired.
v1 golden fixtures become v2 compatibility tests.

### D6 — Adding an agent is cheap by construction

- Agent IDs are strings validated by the registry at load time, not compiled unions (required
  anyway for third-party adapters).
- One conformance suite runs against every registered adapter and checks declared support
  against observed behaviour.
- Golden fixtures cover cross-cutting behaviour (formats, plan/apply semantics), not per-agent
  CLI output.
- Target: a new built-in agent is one manifest, an optional module, and a fixture directory; a
  third-party agent is one file.

### D7 — Monitoring scope

In scope now: (1) support coverage — `agents matrix` generated from D1 data and the D6 suite;
(2) agent drift — re-verify declared facts locally or in CI against new agent releases;
(3) installation health — `doctor`, aware of shared locations and collision policies.
Deferred: (4) runtime metrics; the core exposes an event interface so they can be added later.

### Migration approach

Spec-first: capture v1 behaviour as golden fixtures, then build a Cargo workspace that passes
them. Proposed crates: `skillsmith-model` (D1/D3 types), `skillsmith-formats` (v1-compatible
codecs), `skillsmith-adapter` (trait, manifest loader, JSON-RPC client), `skillsmith-agents`
(one module per built-in agent), `skillsmith-plan` (coverage and ownership), `skillsmith-cli`.

## Open issues

- **Binary name collision.** With shared state (D5), v1 and v2 cannot both install a command
  named `skillsmith`. The crates.io name `skillsmith` is reserved (0.0.0, owner smorin).
- **Unverified facts** (not relied on): Codex `~/.codex/skills/.system`; the OpenCode and Kilo
  Code env vars that disable external skill roots (Kilo's may be
  `OPENCODE_DISABLE_EXTERNAL_SKILLS`); the list of other agents reading `~/.agents/skills`.
- **v1 location gaps** found during research (walk-up, additive `OPENCODE_CONFIG_DIR`, Muse
  legacy roots) are v1 issues; v2 models them from the start.

## Decision log

| ID | Question | Answer | Date |
|---|---|---|---|
| Q1 | Share state with byte-compatible formats? | Q1.A — yes (D5) | 2026-09-24 |
| Q2 | Which monitoring readings are designed now? | Q2.A — coverage, drift, health; metrics deferred (D7) | 2026-09-24 |
| Q3 | Approve D1–D4 and D6 as direction? | Q3.A — approved | 2026-09-24 |
| Q4 | Where does the v2 design live? | Q4.A — this repo; v1 P21 points here | 2026-09-24 |
| Q5 | Repo visibility and license? | Q5.A — public, Apache-2.0 | 2026-09-24 |

Earlier alignment answers (2026-09-23): capability abstraction via traits plus declarative
data (explore further); third parties must be able to add agents; migration approach (iii),
spec-first; the proposal is iterated inline before being written to files.

## Alternatives considered

- **D1:** a single closed enum (merges agent and implementation facts); v1's boolean.
- **D2:** built-in only (fails the third-party requirement); WASM first (toolchain maturity).
- **D3:** per-agent roots plus a shared-root table (reintroduces duplicated lists that ADR 0007
  removed); a general graph solver (unneeded machinery).
- **D4:** single owner (v1's de facto stance, blocks #99); hard error on shared writes.
- **D5:** separate data directory until cutover; shared manifest/lock with separate ledgers.
