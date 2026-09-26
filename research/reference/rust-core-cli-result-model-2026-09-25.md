---
type: terminal
status: current
created: 2026-09-25
updated: 2026-09-25
confidence: high
confidence_basis: Pattern read from cargo, uv, ruff, rustup, and Terraform source (2026-09-25); the example workspace built, its 2 tests passed, the binary ran with exit codes 2/2/2 for the three demo cases, and cargo clippy confirmed the boundary lints on rustc 1.98.1.
verified_example: true
assumptions: The core is embeddable by third parties; ADR 0001 D2/D7/D8/D11 stand; crate names follow the ADR's proposed workspace; schema-id spelling is illustrative pending an owner decision.
sources:
  - https://github.com/rust-lang/cargo/blob/master/crates/cargo-util-terminal/src/shell.rs
  - https://github.com/rust-lang/cargo/blob/master/src/util/errors.rs
  - https://doc.rust-lang.org/cargo/commands/cargo-metadata.html
  - https://github.com/astral-sh/uv/blob/main/crates/uv/src/commands/mod.rs
  - https://github.com/astral-sh/uv/blob/main/crates/uv/src/commands/install_report.rs
  - https://github.com/astral-sh/ruff/blob/main/crates/ruff_db/src/diagnostic/mod.rs
  - https://docs.astral.sh/ruff/linter/
  - https://github.com/rust-lang/rustup/blob/master/src/lib.rs
  - https://github.com/rust-lang/rust-clippy/blob/master/clippy_lints/src/write/mod.rs
  - https://github.com/rust-lang/rust-clippy/blob/master/clippy_lints/src/exit.rs
  - https://doc.rust-lang.org/rustc/json.html
  - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#diagnostic
  - https://github.com/hashicorp/terraform/blob/main/internal/tfdiags/diagnostics.go
  - https://github.com/dtolnay/anyhow/blob/master/README.md
  - https://github.com/tokio-rs/tracing/blob/master/tracing/README.md
  - https://github.com/hashintel/hash/blob/main/libs/error-stack/CHANGELOG.md
origin_prompt: topics/v2-architecture-patterns/prompts/00-landscape.prompt.md
---

# Rust core/CLI split and result model for Skillsmith v2

Survey and evidence:
[`topics/v2-architecture-patterns/02-core-cli-results-patterns.md`](../topics/v2-architecture-patterns/02-core-cli-results-patterns.md).

## The specific knowledge

**Crate layout.** Dependency edges only point downward.

| Crate | Owns | May depend on | Must not |
|---|---|---|---|
| `skillsmith-model` | D1/D3 types, `Diagnostic`, `Severity`, `Origin` | serde | do I/O |
| `skillsmith-core` | use cases (`verify`, `list`, `agents`, later `plan`/`apply`), `Outcome`, `CheckResult`, `Fatal`, `Event`, `Sink` | model, formats, adapter | print, exit, set a tracing subscriber, use anyhow, use global mutable state |
| `skillsmith-cli` | clap, renderers (human and JSON), schema ids, exit-code mapping, `Sink` implementation, tracing subscriber | core, anyhow | contain decision logic |

**Three outcomes, three channels.**

| Outcome | Rust shape | Exit code |
|---|---|---|
| Ran; only notes or warnings | `Ok(Outcome)`, highest severity below `Error` | 0 |
| Ran; found problems | `Ok(Outcome)` with at least one `Error` diagnostic | 1 |
| Could not run: the whole run (bad config, busy lock, missing skill) or any selected check | `Err(Fatal)`, or a `CheckStatus::CouldNotRun` | 2 |

This matches ruff's exit codes (0 = no violations, 1 = violations found, 2 = abnormal
termination). Add uv-style `External(u8)` only if a command passes through a child process's
exit code.

**Types.** These are trimmed from the compiled example.

```rust
#[derive(Serialize, Deserialize)] #[serde(rename_all = "lowercase")]
pub enum Severity { Info, Warning, Error }            // Ord: Info < Warning < Error

#[derive(Serialize, Deserialize)] #[serde(tag = "kind", rename_all = "snake_case")]
pub enum Origin { Spec, Agent { agent: String }, Core }   // D11: spec rule vs agent rule

#[derive(Serialize, Deserialize)] #[non_exhaustive]
pub struct Diagnostic {
    pub code: String,        // stable, namespaced: "spec/name-mismatch", "agent/codex/unknown-key"
    pub severity: Severity,
    pub message: String,
    pub origin: Origin,
    pub path: Option<String>,
    pub data: Option<serde_json::Value>, // LSP-style opaque slot for adapter extensions (D2)
}

#[serde(tag = "status")]
pub enum CheckStatus { Ran, Skipped { reason: String }, CouldNotRun { reason: String } }
pub struct CheckResult { pub check: String, pub subject: String,
                         #[serde(flatten)] pub status: CheckStatus, pub diagnostics: Vec<Diagnostic> }

pub struct Outcome<T> { pub value: T, pub diagnostics: Vec<Diagnostic> }

#[derive(thiserror::Error)] #[non_exhaustive]
pub enum Fatal { #[error("skill directory not found: {0}")] SkillNotFound(String),
                 #[error("state lock held by another process: {0}")] LockBusy(String) }

#[serde(tag = "event")] #[non_exhaustive]
pub enum Event<'a> { CheckStarted { check: &'a str, subject: &'a str },
                     CheckFinished { check: &'a str, subject: &'a str, findings: usize } }
pub trait Sink: Send + Sync { fn event(&self, e: &Event<'_>); }   // D7 event interface
```

**Use-case signature.** Every command has one:
`fn verify(req, &dyn Sink) -> Result<Outcome<Vec<CheckResult>>, Fatal>`.

- Inside `verify`, a check that returns `Err` becomes `CouldNotRun`, and the loop continues.
  This is how ADR D11's no-short-circuit rule is met.
- `?` is used only for `Fatal`.
- `Outcome::verdict()` folds the results into `Clean`, `Notes`, `Problems`, or `Incomplete`.

**JSON output (ADR D8).** The CLI wraps each result in the following document:

```json
{ "schema": "sks:verify@1", "verdict": "incomplete", "value": [...], "diagnostics": [] }
```

- Changes within `@1` are additive: consumers ignore unknown fields and unknown enum values,
  as with cargo metadata `version: 1` and rustc JSON.
- Fatal errors are rendered as `{schema, error: {code, message, chain}}`.
- A progress stream, if one is ever needed, is JSON lines with an `event` discriminator and a
  first line that declares the version. cargo uses a `reason` field; Terraform declares
  `"ui": "1.3"`.
- **Persisted state is different (ADR D5).** State files keep v1's strict rule: unknown fields
  are rejected.

**Boundary enforcement** in `skillsmith-core`.

```toml
# crates/core/Cargo.toml
[lints.clippy]
print_stdout = "deny"
print_stderr = "deny"
exit = "deny"
dbg_macro = "deny"
disallowed_methods = "deny"
```

```toml
# crates/core/clippy.toml
disallowed-methods = [
  { path = "std::io::stdout", reason = "core returns data; the CLI renders" },
  { path = "std::io::stderr", reason = "core returns data; the CLI renders" },
  { path = "std::process::exit", reason = "only the CLI picks exit codes" },
]
```

Add `std::env::var` to that list and route environment reads through an injected `Env` port.
cargo (`GlobalContext::get_env`) and rustup (the `Process` enum) do this for testability.

## Minimal example

Built at `/private/tmp/claude-501/.../scratchpad/rm`: a workspace with `sks-core` (lib) and
`sks-cli` (bin `sks`), rustc 1.98.1, edition 2024.

- `cargo test -p sks-core`: 2 passed.
  - The test asserts that check 2 runs after check 1 fails.
  - It asserts the verdicts `Problems`, `Incomplete`, and `Notes`, and that an empty skill
    name returns `Err(Fatal)`.
  - It asserts that `Diagnostic` survives a JSON round trip.
- `sks foo` printed `lint [ran]`, one `spec/name-mismatch` error, and
  `load [could not run: agent binary not found]`. Exit code 2.
- `sks foo --json` printed the schema-stamped document above. Exit code 2.
- `sks` with no skill printed `error: skill directory not found:`. Exit code 2.
- `cargo clippy` with a planted `writeln!(std::io::stdout(), ..)`, `println!`, and
  `std::process::exit` in core flagged all three. The clean tree has 0 warnings.

CLI shell, verbatim in substance:

```rust
fn run() -> anyhow::Result<ExitCode> {
    let outcome = sks_core::verify(&skill, CHECKS, &StderrSink)?;   // Fatal -> anyhow
    let verdict = outcome.verdict();
    render(&outcome, verdict, json)?;                                // only place that writes stdout
    Ok(match verdict { Clean | Notes => 0, Problems => 1, Incomplete => 2 }.into())
}
fn main() -> ExitCode { run().unwrap_or_else(|e| { eprintln!("error: {e:#}"); ExitCode::from(2) }) }
```

## Gotchas

- **`print_stdout` catches only `print!` and `println!`.** Verified: `writeln!(std::io::stdout())`
  passes it. `disallowed-methods` is the lint that catches it.
- **clippy uses the nearest `clippy.toml` and does not merge files.** Verified: the per-crate
  file was honored when a workspace-root `clippy.toml` also existed. So the core file must
  repeat any workspace-wide entries.
- **`[lints]` is per package.** Lints set in the core's manifest do not apply to the CLI.
  Printing in the CLI stays legal, which is intended.
- **Incomplete outranks Problems in the exit code.** When both occur, the exit code is 2, but
  the JSON still lists the findings. Document this in the exit-code table.
- **`Skipped` is not failure.** An agent that lacks the capability is excluded with a reason
  (D1). It must not raise the verdict.
- **Never print and then return `Err`.** cargo needs an `AlreadyPrintedError` sentinel because
  it did this.
- **No global state in the core.** uv's `uv-warnings` process-global `AtomicBool` breaks two
  embedders sharing one process.
- **The core must not call `tracing::subscriber::set_global_default`.** The tracing README
  forbids it for libraries. Filtering can drop events, so results never travel through tracing.
- **`#[non_exhaustive]` on public enums and structs** lets `Diagnostic`, `Event`, and `Fatal`
  grow without a breaking change. The cost is that external code builds them through
  constructors.
- **Codes are API.** Keep a redirect table for renamed codes, as ruff does in
  `rule_redirects.rs`.

## Currency notes

| Crate | Version | Status |
|---|---|---|
| thiserror | 2.0.21 | Current (2026-09-23) |
| anyhow | 1.0.104 | Current |
| serde_json | 1.0.151 | Current |
| tracing | 0.1.44 | Current |
| miette | 7.6.0 | Last release 2025-04; low activity |
| error-stack | 0.8.0 | Breaking releases in 2025-08, 2026-03, and 2026-07 |

- cargo moved `Shell` to `cargo-util-terminal` in 2026.
- rustup moved notifications onto `tracing` (#4501, 2025-10).
- rustc's move to annotate-snippets rendering is in progress. Whether it is the default today
  is unconfirmed (medium confidence).

## Recommendation

**Chosen.** Adopt the following. Confidence: high.

- `Result<Outcome<T>, Fatal>` use cases, with per-check `CheckResult`.
- A serde `Diagnostic` with stable namespaced codes and an `Origin`.
- `Fatal` defined with thiserror in the core; anyhow only in the CLI.
- A typed `Sink` for events; `tracing` for logs only.
- Schema-stamped, additive `--json` output.
- Exit codes 0/1/2.
- Clippy lints plus `disallowed-methods` on the core crate.

**Runner-up: ruff's single-diagnostic model.** It has one `Diagnostic` type with
`Fatal` severity and `Io`-class ids, and no `CheckResult`. It is simpler. Switch to it only if
`verify` stays the sole multi-check command and per-check `Skipped`/`CouldNotRun` status is
unnecessary.

**Why not:**

| Alternative | Reason |
|---|---|
| miette `Diagnostic` as the public type | Couples data to its renderer; low activity. It is fine as an optional CLI renderer. |
| error-stack | API churn |
| `tracing` as the UI or results channel (rustup's direction) | Lossy under filters |
| anyhow in the core | Not matchable or serializable by embedders |
| `Result<T, Vec<Diagnostic>>` | Loses the value when problems exist |
| Rendering in the core (cargo's `Shell` via `GlobalContext`) | Acceptable for an internal library, but blocks embedding |
