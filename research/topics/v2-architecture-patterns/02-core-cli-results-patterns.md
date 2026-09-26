---
type: exploratory
status: chosen
created: 2026-09-25
updated: 2026-09-25
confidence: high
confidence_basis: Crate layouts, lint configs, and result/diagnostic types read from cargo, uv, ruff, rustup, and Terraform source on 2026-09-25; the recommended boundary and Outcome model compiled, ran, and were clippy-checked on rustc 1.98.1.
assumptions: Skillsmith v2 follows ADR 0001 as proposed (D2 adapter tiers, D7 event interface, D8 new versioned output, D11 no-short-circuit verify); the core crate is meant to be embeddable by third parties, unlike cargo's library.
sources:
  - https://github.com/rust-lang/cargo/blob/master/src/lib.rs
  - https://github.com/rust-lang/cargo/blob/master/crates/cargo-util-terminal/src/shell.rs
  - https://github.com/rust-lang/cargo/blob/master/src/util/errors.rs
  - https://github.com/rust-lang/cargo/blob/master/src/util/machine_message.rs
  - https://github.com/rust-lang/cargo/blob/master/src/diagnostics/mod.rs
  - https://github.com/rust-lang/cargo/blob/master/clippy.toml
  - https://doc.rust-lang.org/cargo/commands/cargo-metadata.html
  - https://doc.rust-lang.org/cargo/reference/external-tools.html
  - https://github.com/astral-sh/uv/blob/main/crates/uv/src/printer.rs
  - https://github.com/astral-sh/uv/blob/main/crates/uv/src/commands/mod.rs
  - https://github.com/astral-sh/uv/blob/main/crates/uv/src/commands/install_report.rs
  - https://github.com/astral-sh/uv/blob/main/crates/uv-resolver/src/resolver/reporter.rs
  - https://github.com/astral-sh/uv/blob/main/crates/uv-warnings/src/lib.rs
  - https://github.com/astral-sh/uv/blob/main/Cargo.toml
  - https://github.com/astral-sh/ruff/blob/main/crates/ruff_db/src/diagnostic/mod.rs
  - https://github.com/astral-sh/ruff/blob/main/crates/ruff_db/src/diagnostic/render/json.rs
  - https://github.com/astral-sh/ruff/blob/main/crates/ruff_linter/src/rule_redirects.rs
  - https://github.com/astral-sh/ruff/blob/main/clippy.toml
  - https://github.com/astral-sh/ruff/blob/main/Cargo.toml
  - https://docs.astral.sh/ruff/linter/
  - https://github.com/rust-lang/rustup/blob/master/src/lib.rs
  - https://github.com/rust-lang/rustup/blob/master/src/process.rs
  - https://github.com/rust-lang/rustup/blob/master/src/utils/notify.rs
  - https://github.com/rust-lang/rustup/pull/4501
  - https://github.com/rust-lang/rust-clippy/blob/master/clippy_lints/src/exit.rs
  - https://github.com/rust-lang/rust-clippy/blob/master/clippy_lints/src/write/mod.rs
  - https://doc.rust-lang.org/rustc/json.html
  - https://rust-lang.github.io/rust-project-goals/2024h2/annotate-snippets.html
  - https://github.com/rust-lang/rust-project-goals/issues/123
  - https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#diagnostic
  - https://github.com/hashicorp/terraform/blob/main/internal/tfdiags/diagnostics.go
  - https://github.com/hashicorp/terraform/blob/main/internal/command/views/json_view.go
  - https://github.com/dtolnay/anyhow/blob/master/README.md
  - https://github.com/dtolnay/thiserror/blob/master/README.md
  - https://github.com/zkat/miette
  - https://github.com/hashintel/hash/blob/main/libs/error-stack/CHANGELOG.md
  - https://github.com/tokio-rs/tracing/blob/master/tracing/README.md
  - https://doc.rust-lang.org/book/ch12-03-improving-error-handling-and-modularity.html
  - https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell
  - https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
  - https://martinfowler.com/bliki/CommandQuerySeparation.html
origin_prompt: topics/v2-architecture-patterns/prompts/00-landscape.prompt.md
---

# Cluster B — library/CLI separation and results as data

**Purpose.** Decide how Skillsmith v2 splits the embeddable library from the `sks` binary, and
how operations report success, findings, failures, and progress. The durable answer is in
[`research/reference/rust-core-cli-result-model-2026-09-25.md`](../../reference/rust-core-cli-result-model-2026-09-25.md).
This file records the survey behind it.

**Recommendation in one line.** Use a functional-core / imperative-shell split. Core use cases
return `Result<Outcome<T>, Fatal>`: the `Err` side covers "could not run at all", and
`Outcome.diagnostics` carries findings. Progress goes through an injected `Sink` trait.
`tracing` is for logs only. Enforce the boundary with Cargo crate edges plus clippy
`disallowed-methods`. Every `--json` document carries a schema identifier and evolves
additively.

## 1. Canonical patterns

| Pattern | Primary source | What it means for a Rust CLI |
|---|---|---|
| Functional core / imperative shell | Bernhardt, *Boundaries* (2012) | Decisions are pure functions over values. I/O, terminal, and process exit live in a thin outer layer. |
| Use cases / application services (clean, onion, hexagonal) | Martin, *The Clean Architecture*; Cockburn | Each command maps to one core use-case function that takes a request struct and returns a response struct. The CLI is one adapter; an embedder is another. |
| Thin `main` | Rust Book ch. 12.3 | `main.rs` parses arguments, calls `lib::run`, and maps errors to exit codes. |
| Command–query separation | Meyer via Fowler bliki | Read-only queries (`agents`, `list`, `verify`) return data with no side effects. Commands (`install`) return a plan or a result of applying one. This fits ADR D10's milestone order. |
| Accumulate, don't short-circuit | Terraform `tfdiags.Diagnostics` | Functions return `(value, diags)`. Callers check `diags.HasErrors()`. `?` is reserved for errors that stop the whole operation. |

## 2. How the exemplars split library and CLI

| Project | Crate layout | Where rendering lives | Output channel | Boundary enforcement |
|---|---|---|---|---|
| **cargo** | One package with `src/lib.rs` plus `src/bin/cargo`. The library says it is "primarily for use by Cargo and not intended for external use". `ops::*` holds operations; "each command is a thin wrapper around ops". | `Shell` was extracted in 2026 to the `cargo-util-terminal` crate (`status`, `warn`, `note`, `print_json`, `print_report` via annotate-snippets). The library reaches it through `GlobalContext`. | `Shell` is an injected sink. `Shell::from_write` captures output in tests. | Workspace lints set `print_stdout`/`print_stderr` = warn. `clippy.toml` `disallowed-methods` bans `std::env::var` in favor of `GlobalContext::get_env`. |
| **uv** | About 70 `uv-*` library crates. The `uv` crate is the CLI; its `commands/` directory holds per-command orchestration. | `Printer` enum (`Silent`/`Quiet`/`Default`/`Verbose`/`NoProgress`) in the CLI crate. JSON report types (`install_report.rs`) are also CLI-owned. | Per-library `Reporter` traits (`on_progress`, `on_download_start`, ...). The CLI implements them with indicatif. | Workspace lints set `print_stdout`, `print_stderr`, and `exit` = warn. |
| **ruff** | `ruff` (binary), `ruff_linter`, `ruff_workspace`, `ruff_server` (LSP), and `ruff_db` (shared `Diagnostic` type plus renderers). | `ruff_db::diagnostic::render::{full, concise, json, json_lines, github, gitlab, junit, pylint, rdjson, azure}`. The data type and renderers live in one library crate; the binary picks a `DiagnosticFormat`. | Diagnostics are values. The LSP server and CLI consume the same type. | Lints set `print_stdout`, `print_stderr`, and `exit` = warn. `disallowed-methods` routes `std::fs`/`std::env` through a `System` trait in ty crates. |
| **rustup** | One package with a `[lib]` plus `src/bin`. The CLI modules sit in `src/cli/` inside the same library. | `src/cli/*`. | User-facing notifications moved onto `tracing` (PR #4501, 2025-10, "Remove more Notification variants"). Only a `NotificationLevel` mapping remains. | `lib.rs`: `#![cfg_attr(not(test), warn(clippy::print_stdout, clippy::print_stderr))]` with the comment "We use the logging system instead of printing directly." `Process` enum (`OsProcess`/`TestProcess`) injects env and stdio for tests. |

**Reading.** All four forbid ad-hoc printing in library code with the same clippy lints. uv
and ruff also forbid `exit`. None exposes a stable embeddable library API: cargo disclaims it,
and uv and ruff publish crates without semver promises. Skillsmith v2 differs because it wants
an embeddable core. So it needs a stricter version of what these projects do, not a copy.

## 3. Errors and diagnostics

**Errors vs findings.** An *error* means the operation could not produce its value. A
*diagnostic* is a finding about the subject, which here is a skill.

- **thiserror vs anyhow.** anyhow's README: "Use Anyhow if you don't care what error type your
  functions return... common in application code. Use thiserror if you are a library."
  thiserror's README makes the reciprocal recommendation. uv follows this split
  (`UvError` in thiserror wrapping `anyhow::Error`). cargo does not: it uses
  `CargoResult = anyhow::Result` throughout its library, which is acceptable only because the
  library is internal. Current versions: thiserror 2.0.21 (2026-09-23), anyhow 1.0.104.
- **miette.** Its `Diagnostic` trait provides code, severity, help, URL, and labels, plus a
  graphical/narratable/JSON reporter. It is library-compatible. Currency: last release 7.6.0 on
  2025-04-27, last feature commit 2025-09; activity is low.
- **error-stack.** `Report<C>` with attachments. Version 0.6 (2025-08) added `Report<[C]>` for
  grouped errors. It then broke its API again in 0.7 (2026-03) and 0.8 (2026-07). Three breaking
  releases in about 11 months is the reason not to put it in a public API.
- **rustc diagnostics.** JSON fields: `$message_type`, `message`, `code{code,explanation}`,
  `level`, `spans[]`, `children[]`, `rendered`. The docs state: "New fields may be added.
  Enumerated fields ... may add new values." Stable codes (`E0308`) link to long explanations.
  Moving rustc's renderer to annotate-snippets has been a project goal (2024H2, 2025H1) and is
  in progress. Whether it is the default today is unconfirmed (medium confidence).
- **LSP `Diagnostic`.** `range`, `severity` (1–4), `code`, `codeDescription.href`, `source`,
  `message`, `tags`, `relatedInformation`, `data`. `data` is an opaque round-trip slot. It is the
  model for an adapter-extension field (see cluster A).
- **ruff's unified `Diagnostic`.** Fields: `DiagnosticId` (`Lint(name)`, `Io`, `InvalidSyntax`,
  `Panic`, ...), `Severity {Info, Warning, Error, Fatal}`, annotations, sub-diagnostics, fix,
  and documentation URL. A file that cannot be read becomes a diagnostic with id `Io`; it is
  not a returned error. Codes stay stable, and `rule_redirects.rs` maps old codes to new ones.
- **cargo's own lints** (`src/diagnostics/`): "Lints are generally preferred because of the
  level of control for users." A hard-coded diagnostic is used only for critical errors.

**Three outcomes.** ruff's exit codes are the cleanest precedent. `0` means no violations
("ran with notes" still exits 0). `1` means violations found. `2` means abnormal termination
(bad config, bad CLI options, internal error). uv's `ExitStatus` adds `External(u8)` to pass
through a child process's code. Terraform encodes the same idea in data: one `Diagnostics`
list, and `HasErrors()` decides.

## 4. Progress and events

- **cargo `Shell`**: an injected, verbosity-aware writer. `print_json` emits machine messages
  tagged with a `reason` discriminator (`compiler-message`, `build-finished`, ...) as JSON lines.
- **uv `Reporter` traits**: typed callbacks per library crate. The CLI supplies the rendering.
- **tracing**: its README says "Libraries should only rely on the `tracing` crate" and "should
  NOT call `set_global_default()`".
- **Terraform `-json`**: a line-delimited event stream whose first message declares
  `"ui": "1.3"` (`JSON_UI_VERSION`).

**Disagreement (recorded, not resolved by the sources).** rustup routes user-facing messages
through `tracing` (#4501). uv and cargo keep an explicit channel. Recommendation: use a typed
`Sink` for product events and `tracing` for diagnostic logs. Reason: a subscriber's filter
(`RUST_LOG`, `EnvFilter`) can silently drop events, so `tracing` cannot be a reliable results
channel. ADR D7's "event interface for later metrics" is exactly this `Sink`.

## 5. Output schemas and serializable errors

- **cargo metadata**: `"version": 1`. Adding fields, adding enum values, and changing opaque
  representations are all declared compatible.
- **rustc JSON**: additive evolution, stated explicitly.
- **uv**: its JSON reports carry `"schema": {"version": "preview"}`. Seen in
  `install_report.rs` and `audit/json.rs`; other uv JSON outputs were not checked.
- **ruff JSON**: no schema field. The shape changes under `preview` (the `code` field becomes
  optional and `severity` is shown). This is the pitfall: consumers cannot detect the change.
- **Serializable errors.** `anyhow::Error` and `thiserror` enums are not `Serialize`. Convert
  fatal errors to a serializable `{code, message, chain}` in the CLI. Findings are `Diagnostic`
  values that derive both `Serialize` and `Deserialize`. Deserialize is needed because Tier 2
  adapters (external JSON-RPC programs) return diagnostics over the wire.

**The ADR D5 and D8 compatibility rules differ; do not conflate them.**

| Artifact | ADR decision | Evolution rule |
|---|---|---|
| Persisted state (`manifest@1`, `ledger@2`, ...) | D5 | Strict: reject unknown fields, matching v1 ADR 0008 |
| Command output (`--json`) | D8 | Every document carries a v2-only `schema` id. Changes within a version are additive, and consumers ignore unknown fields (cargo and rustc precedent). |

## 6. Anti-patterns and pitfalls

1. **Print-then-error.** cargo's `AlreadyPrintedError` exists because library code printed a
   message and then returned an error. It is a sentinel that compensates for a leak.
2. **Process exit in the library.** cargo exports `exit_with_error` from `lib.rs`. Tolerable
   only because cargo's library is internal.
3. **Global output state.** `uv-warnings` has a process-global `ENABLED: AtomicBool` and
   deduplication set. Two embedders in one process share it.
4. **Unversioned JSON.** See ruff above.
5. **`print_stdout` is macro-only.** Clippy's `write/mod.rs` lints `print!`/`println!` calls
   only. Verified on rustc 1.98.1: `writeln!(std::io::stdout(), ..)` passes `print_stdout`, and
   only `disallowed-methods` on `std::io::stdout` catches it.
6. **Short-circuiting multi-check runs with `?`.** This violates ADR D11. A per-check failure
   must become data.
7. **Using `Result<T, Vec<Diagnostic>>` for findings.** It loses the value, such as a partial
   inventory, when problems exist.

## 7. What changed recently

- cargo extracted `Shell` into `cargo-util-terminal` (MSRV 1.98; progress detection moved into
  `Shell` 2026-07-20). It renders reports through annotate-snippets.
- rustup moved notifications to `tracing` (2025-10).
- error-stack 0.6 → 0.8 API churn (2025-08 to 2026-07).
- miette activity is low since 2025.
- ruff and ty share the `ruff_db` diagnostic model, including the `Fatal` severity.
- Nothing in cluster B that was best practice is now an anti-pattern. The one drift is
  rustup's move toward tracing-as-UI, which this survey advises against for Skillsmith's
  results.

## 8. Applicability to Skillsmith

- **Direct.** Thin binary; use-case functions; clippy print/exit lints; `disallowed-methods`;
  stable namespaced codes with a redirect table; exit codes 0/1/2.
- **Adapt.** The core must be embeddable, which exceeds cargo, uv, and ruff. So no anyhow, no
  exit, and no global state in the core, and `Diagnostic` must round-trip JSON for Tier 2.
  - D11: each check yields a `CheckResult` whose status is `Ran`, `Skipped` (the adapter lacks
    the capability), or `CouldNotRun` (for example, the agent binary is missing). A run-level
    `Fatal` covers a missing skill or a busy lock.
  - D11 origin: a finding records whether a spec rule or an agent rule produced it. Codes are
    namespaced as `spec/<rule>` and `agent/<id>/<rule>`.
  - D2: `Diagnostic.data`, an LSP-style extension slot, is where cluster A's escape hatch
    plugs in.
  - v1 continuity: the TypeScript core already bans `console` and `process.exit` and returns
    `Result<T, SkillSmithError>` (repo `CLAUDE.md`, ESLint-enforced). v2 carries the same rule
    into clippy.

## 9. Recommendation (ranked)

1. **Chosen: Outcome + typed Sink + a thin shell.** Crates: `skillsmith-model`,
   `skillsmith-core` (use cases, `Outcome`, `Diagnostic`, `Fatal` in thiserror, `Sink`), and
   `skillsmith-cli` (anyhow, clap, renderers, schema-stamped JSON, exit mapping).
   Confidence: high. Basis: four exemplars converge on this, and it compiled and passed tests
   here.
2. **Runner-up: the ruff model.** One rich `Diagnostic` type with `Fatal` severity and
   `Io`-class ids, and no separate `Outcome` or `CheckResult`. It is simpler and proven at
   scale. It ranks second because D11 needs per-check status (`Skipped` vs `CouldNotRun`) and
   capability-exclusion reasons that are awkward to express as diagnostics. Choose it if verify
   remains the only multi-check command.
3. **Rejected.**
   - miette as the public diagnostic type: low activity, and it couples to its rendering.
   - error-stack: API churn.
   - `tracing` as the results channel: filterable and lossy.
   - Rendering inside the core, as rustup does: blocks embedding.
