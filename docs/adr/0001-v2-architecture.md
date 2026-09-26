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
  stdio with an `initialize` capability handshake (LSP 3.17 pattern; MCP removed its handshake
  in its 2026-07-28 spec). A static `agent.toml` beside the program lets listing need no process.

Built-in agents are themselves expressed as a Tier 1 manifest plus Rust code for what data
cannot express. Tier 2's protocol is language-neutral, so TypeScript v1 could load the same
third-party adapters. WASM plugins (Extism / component model) are a possible later tier; they
are deferred because they would sandbox adapter code, while the actual risk is the agent binary
the adapter launches (WASI 0.3.0 shipped 2026-06-11, so toolchain maturity is no longer the
reason).

**Adapter structure (research-backed, 2026-09-26; leaf `capability-adapter-patterns`).**
- `AgentAdapter` is a sealed base role with per-capability accessors returning
  `Option<&dyn Capability>`. The answers are static per adapter; the D6 conformance suite fails
  an adapter whose accessors disagree with its D1 declarations. Three tier wrappers implement
  it (`BuiltIn<H>`, `Manifest`, `External`); built-in agents implement an unsealed
  `BuiltinHooks`. Traits are synchronous.
- **Plan / execute / analyze** for every capability that runs an agent binary: the adapter
  returns an `ExecPlan` (program, args, environment allowlist, timeout) as data; the core runs it
  in a sandbox it owns (throwaway skill copy, temporary `HOME`); the adapter turns the output
  into findings. Adapters never hold a process handle. Tier 2 exposes the same split on the wire
  (`verify.load/plan`, `verify.load/analyze`).
- **Credential boundary:** only allowlisted environment variables reach the agent, and `HOME` is
  always temporary. Declared `exec` permissions (program names) are for display and drift
  checks; the core refuses a plan whose program is undeclared.
- **Escape paths:** every wire object and `ExecPlan` carries an `extensions` map with
  reverse-DNS keys (MCP `_meta` syntax, e.g. `com.example/timeout-hint`; `dev.skillsmith/` is
  reserved). Custom capabilities use the same syntax (`com.example/verify.secrets`) and are
  invoked only by explicit commands. Core ids stay bare.
- **Open:** `ExecPlan` as sketched covers one command run. v1's Codex load check is a JSON-RPC
  exchange over stdin and Muse's runs two commands (P01-T10).

**Tier 2 trust rule.** Finding `skillsmith-agent-<name>` on `PATH` is not enough to run it; the
`initialize` handshake checks protocol compatibility, not provenance. A Tier 2 adapter runs only
if it is listed in the Skillsmith config allowlist (absolute path plus SHA-256 of the
executable), or after the user approves it at first use, which records that allowlist entry. A
changed hash requires approval again. Tier 1 manifests execute no code and need no allowlist.

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

v2 reads and writes v1's data directory, resolved exactly as v1 does
(`packages/core/src/place/paths.ts:4`, `config/runtime.ts:47`, `ports/default.ts:87`):
`$SKILLSMITH_HOME` when set and non-empty (empty counts as unset); otherwise
`$XDG_DATA_HOME/skillsmith`, where an unset `XDG_DATA_HOME` defaults to `$HOME/.local/share` and
an empty one is kept, yielding the relative path `skillsmith`. v2 reads and writes
persisted artifacts byte-compatibly: manifest@1, lock@1, plan@1, ledger@2, journal@1. v2 also
reads ledger@1 so it can migrate older ledgers, matching v1. v2-only
data goes in new, separate files. Consequence: no shared format changes until v1 is retired.
v1 golden fixtures become v2 compatibility tests.

**Write coordination.** Byte compatibility alone allows lost updates if v1 and v2 write at the
same time. v2 therefore takes the same locks v1 takes, with the same on-disk identities. v1 uses
`proper-lockfile@4.1.2` (mkdir-based) with three lock families, all with `realpath: false`:

| Lock | Lock directory | `stale` ms | `update` ms |
|---|---|---|---|
| Ledger | `<data>/placements.json.lock` | 30000 | 5000 |
| Artifact central | `~/.skillsmith/coordination/artifacts-v1/global.lock` (mode 0700) | 2000 | 1000 |
| Artifact per target | `<target>.lock`, taken in sorted order, plus a member-marker JSON | 30000 | 5000 |

Lock directory names derive from the target path the way `proper-lockfile` does with
`realpath: false`: Node's `path.resolve`, which makes the path absolute and resolves `.` and `..`
lexically without following symlinks. v2 must apply the same lexical normalization;
`std::path::absolute` keeps `..` and `canonicalize` follows symlinks, and either would give a
different lock directory than v1 for some paths.

v2 implements the convention in-house (acquire by `mkdir`; stale when mtime is older than
`stale`; the holder refreshes mtime every `update` ms; release by `rmdir`). Effective values
follow the 4.1.2 source, `stale = max(stale, 2000)`, not the README's 5000 ms minimum. A compiled
Rust sketch and `proper-lockfile` 4.1.2 excluded each other in a local run (2026-09-25). Evidence that the identity matters: v1 once locked a sidecar target, creating
`placements.json.lock.lock`, and old and new binaries then ran without excluding each other
(the "split-brain" described at `ledger.ts:424`). A concurrency test runs v1 and v2 against one
data directory and proves mutual exclusion.

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

### D12 — Core library returns data; the CLI renders

(Research-backed, 2026-09-26; leaf `rust-core-cli-result-model`.)
- Crates: the core library has no `clap`, no printing, no `process::exit`; enforced by the
  dependency graph, clippy lints (`print_stdout`, `print_stderr`, `exit`), and
  `disallowed-methods` for `std::io::stdout` / `stderr` (the lints alone miss
  `writeln!(std::io::stdout())`).
- Every use case returns `Result<Outcome<T>, Fatal>`. `Fatal` (thiserror) means it could not run.
  `Outcome` carries the value and diagnostics; each check yields a `CheckResult` with status
  `Ran`, `Skipped`, or `CouldNotRun`, so no check stops another (D11).
- Public enums (`Severity`, `Origin`, `CheckStatus`, and later additions) are
  `#[non_exhaustive]`. Enums that cross the Tier 2 wire deserialize tolerantly: an unknown value
  maps to an `Unknown` variant (`#[serde(other)]`) instead of failing, so additive evolution
  holds for adapters too.
- `Diagnostic` is a serde type with a stable namespaced code (`spec/...`, `agent/<id>/...`),
  severity, message, subject, origin (core or adapter), and optional structured data.
- Progress goes through a typed event `Sink` passed into the core; this is the D7 event
  interface. `tracing` is used for logs only. anyhow is used only in the CLI.
- `--json` documents carry a v2 schema id and evolve additively (unlike persisted state, D5).
  Exit codes: 0 ran, 1 problems found, 2 could not run.

### D8 — v2 command output is new and not backward compatible

v2's command output (`--json` results, human-readable rendering, exit-code tables) uses new
schemas, separate from v1's CLI wire registry (v1 ADR 0008: `agents@2`, `list@3`, `verify@1`,
...). v2 does not reproduce v1 output. The two are mutually exclusive by construction: every v2
JSON document carries a v2-only schema identifier, so a consumer cannot mistake one for the
other. This does not change D5: persisted state files stay byte-compatible with v1.

### D9 — Command names during and after the overlap

- v2 ships two Cargo binary targets from one library: `sks` and `skillsmith`
  (`src/bin/sks.rs` calls `skillsmith_cli::run()`).
- During the overlap, the `skillsmith` target has `required-features = ["long-name"]`, so a plain
  `cargo install` provides only `sks` and cannot replace v1's `skillsmith` command.
- At the cutover the gate is removed. Homebrew then installs `sks` as a symlink to `skillsmith`;
  release tarballs ship the symlink (a copy on Windows).
- The CLI passes the invoked name (`argv[0]`) to clap's `bin_name`, so help text shows the name
  the user typed.
- Not yet checked: whether `sks` is free on crates.io, npm, and Homebrew.

### D10 — Initial command subset

The first milestone builds three read-only commands:

| Command | Proves |
|---|---|
| `agents` | D1 capability states and the D7 coverage matrix; D2 adapter tiers |
| `list` | D3 location model and D5 reads of v1 state (`placements.json`, agent skill roots) |
| `verify --check lint` | Capability-driven dispatch: the command names no agent |

Milestone order (user instruction, 2026-09-25): (1) the three commands above; (2) load
verification, `verify --check load`, which first needs the D2 data/code boundary (P01-T02);
(3) `install` and `uninstall`, after the v1/v2 lock concurrency test (P01-TS03) passes. v2 writes
no shared state before milestone 3.

### D11 — Verification levels are named for what they do

`verify` takes one flag, `--check <level>`, instead of v1's `--static` / `--deep`. Each level
includes the levels before it:

| Level | What runs | Capability |
|---|---|---|
| `lint` (default) | Skillsmith's own checks of `SKILL.md` against the skills spec; no agent runs | `verify.lint` |
| `validate` | `lint`, plus each agent's own validator (v1's static check, e.g. `claude plugin validate`, `muse skills validate`) | `verify.validate` |
| `load` | `validate`, plus each agent's binary loading the skill from an isolated throwaway copy; no model call (v1's deep check) | `verify.load` |

**No short-circuit (user instruction, 2026-09-25).** Every check in the selected levels runs
even when an earlier one fails, so one run reports all findings. The verdict combines them.

Correction: an earlier draft said v1's static check runs no agent. It runs each agent's own
validator; v1 has no agent-independent check, which `lint` adds.

**Capabilities live in the adapter (2026-09-25).** `verify.lint` is defined once in the core
from the skills spec; an adapter may add agent-specific lint rules (for example frontmatter keys
only that agent accepts; v1 reserved `agents/<tool>/frontmatter.ts` for this but never filled it).
`verify.validate` and `verify.load` are implemented entirely by each adapter, which runs the
agent and interprets its output. Lint findings record whether a spec rule or an agent rule
produced them. The core/adapter interface boundary is still being designed.

A later level that exercises the skill through a model (for example `invoke`) can be added as a
new value without a new flag.

### Migration approach

Spec-first: capture v1 behaviour as golden fixtures, then build a Cargo workspace that passes
them. Fixtures cover persisted formats and plan/apply semantics, not CLI output (D8). Proposed crates: `skillsmith-model` (D1/D3 types), `skillsmith-formats` (v1-compatible
codecs), `skillsmith-adapter` (trait, manifest loader, JSON-RPC client), `skillsmith-agents`
(one module per built-in agent), `skillsmith-plan` (coverage and ownership), `skillsmith-cli`.

## Open issues

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
| Q6 | Apply the review findings (Tier 2 trust, write coordination, fixes)? | Q6.A — applied | 2026-09-25 |
| Q7 | Command names? | Q7.A — `sks` during the overlap, both names at cutover (D9) | 2026-09-25 |
| Q8 | Initial command subset? | Q8.A — read-only `agents`, `list`, static `verify` (D10) | 2026-09-25 |
| Q9 | Verification level names? | Q9.A — `--check lint` / `--check load` (D11) | 2026-09-25 |
| Q11 | Where does a data-only agent's load-check reader come from? | Superseded: capabilities live in the adapter (D11) | 2026-09-25 |
| Q14 | Sandbox-only process access? | Superseded by Q15, then by Q16 | 2026-09-25 |
| Q15 | Sandbox default with declared permissions? | Superseded by Q16 (research changed the role of permissions) | 2026-09-26 |
| Q17 | Startup announcement text? | Q17.A — revised text with `sks` and the research index | 2026-09-26 |
| Q16 | Adopt the researched pattern sets? | Q16.A — adopted into D2, D5, D12 | 2026-09-26 |
| Q13 | Agent-specific lint rules? | Q13.A — shared spec rules plus optional adapter rules (D11) | 2026-09-25 |
| Q10 | What does `lint` do? | Q10.A — three levels `lint`/`validate`/`load`, no short-circuit (D11) | 2026-09-25 |
| — | Milestone order (user instruction) | load verification before `install` / `uninstall` (D10) | 2026-09-25 |
| — | Output compatibility (user instruction) | v2 output is new and not compatible with v1 (D8) | 2026-09-25 |

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
