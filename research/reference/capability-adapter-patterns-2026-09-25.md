---
type: terminal
status: current
created: 2026-09-25
updated: 2026-09-25
confidence: medium
confidence_basis: Each pattern is cross-checked against 2+ primary specs or docs (COM, LSP, MCP, Terraform, cap-std, Deno, Zed, Rust Reference/API guidelines); the combination for Skillsmith is a design inference, not a proven deployment.
verified_example: true
assumptions: Adapter calls are synchronous (std::process, no async runtime); third parties extend only via agent.toml or a JSON-RPC program, never in-process Rust; the core owns every spawn of an agent binary.
sources: ["https://alistair.cockburn.us/hexagonal-architecture", "https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nf-unknwn-iunknown-queryinterface(refiid_void)", "https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/", "https://modelcontextprotocol.io/specification/2025-11-25/basic/index", "https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/", "https://raw.githubusercontent.com/hashicorp/terraform/main/docs/plugin-protocol/tfplugin6.proto", "https://raw.githubusercontent.com/hashicorp/terraform/main/docs/resource-instance-change-lifecycle.md", "https://github.com/bytecodealliance/cap-std", "https://docs.deno.com/runtime/fundamentals/security/", "https://zed.dev/docs/extensions/capabilities", "https://zellij.dev/documentation/plugin-api-permissions.html", "https://docs.rs/http/latest/http/struct.Extensions.html", "https://rust-lang.github.io/api-guidelines/future-proofing.html", "https://doc.rust-lang.org/reference/attributes/type_system.html", "https://rust-lang.github.io/async-fundamentals-initiative/explainer/async_fn_in_dyn_trait.html", "https://bytecodealliance.org/articles/WASI-0.3"]
origin_prompt: topics/v2-architecture-patterns/prompts/00-landscape.prompt.md
---

# Capability-based adapter patterns for Skillsmith v2

Funnel with all options considered:
[01-adapter-capability-patterns.md](../topics/v2-architecture-patterns/01-adapter-capability-patterns.md).
Decisions referenced: ADR 0001 D1 (two-axis support), D2 (three tiers), D6 (conformance suite),
D11 (verify levels), open task P01-T02 (data/code boundary).

## The specific knowledge

Five patterns, used together:

1. **Base role plus capability accessors (COM `QueryInterface` shape).** `AgentAdapter` carries
   only `descriptor()`. Each capability is a separate small trait reached through an accessor
   that returns `Option<&dyn Capability>`. COM's rule applies: the set of answers is **static**
   per adapter ("if the call fails the first time, then it must fail on all subsequent calls").
   The D6 conformance suite checks accessor presence against D1 declarations: `None` where D1
   says `Supported`, or `Some` where D1 says `No`, is a conformance failure.
2. **Plan / execute / analyze for every capability that runs an agent binary.** The adapter
   returns an `ExecPlan` (program, args, env allowlist, timeout) as data; the core executes it in
   a sandbox it owns (throwaway skill copy, temporary `HOME`, scrubbed environment); the adapter
   interprets the `ExecOutcome` into findings. The adapter never holds a process handle.
   Precedent: Terraform `PlanResourceChange`/`ApplyResourceChange`, where Core checks applied
   values against planned values attribute by attribute.
3. **Proposed P01-T02 boundary.** Tier 1 `agent.toml` can express `plan` as an argv template
   with sandbox placeholders (`{skill_copy}`, `{home}`), an env allowlist and a timeout, and
   `analyze` as an exit-code map plus regex-to-finding rules. Anything needing branching, parsing
   structured output, or multiple runs is code: Tier 0 Rust or Tier 2 JSON-RPC. The same three
   methods exist on the wire (`verify.validate/plan`, `verify.validate/analyze`).
4. **Declared permissions as a display and drift check.** Manifests declare `exec` permissions
   (program names, Zed `process:exec` style); the core refuses an `ExecPlan` whose program is
   undeclared and shows the list at Tier 2 approval time. This is **not** the credential
   boundary. The environment scrub and temporary `HOME` are (see Gotchas).
5. **Namespaced data escape hatches.** Every wire object and `ExecPlan` has
   `extensions: map<string, json>`. Keys use MCP `_meta` syntax: reverse-DNS prefix plus `/`
   (`com.example/timeout-hint`); `dev.skillsmith/` is reserved. The core preserves and ignores
   unknown keys. Custom capabilities use the same syntax (`com.example/verify.secrets`); core
   capability ids stay bare (`verify.validate`). Core commands dispatch only core ids; custom ids
   are listed by `agents` and invoked only by an explicit command.

Rust idioms: trait objects for the registry (tiers are chosen at runtime). Seal `AgentAdapter`
so its only implementors are three tier wrappers inside `skillsmith-adapter`: `BuiltIn<H>`,
`Manifest` and `External`. Tier 0 modules in `skillsmith-agents` implement the unsealed
`BuiltinHooks`. Use `#[non_exhaustive]` on public enums and on structs the core builds
(`Sandbox`, `ExecOutcome`). `ExecPlan` is built by adapters, so it is `#[non_exhaustive]` only
because it has a constructor. Keep the traits synchronous: async `fn` in `dyn` traits is still a
draft RFC.

```rust
mod sealed { pub trait Sealed {} }
#[non_exhaustive] pub enum Support { Yes { since: Option<String> }, No { reason: String }, Unknown }
#[non_exhaustive] pub struct ExecPlan {
    pub program: String,          // must match a declared `exec` permission
    pub args: Vec<String>,
    pub env_allow: Vec<String>,   // names only; the core scrubs everything else
    pub timeout: Duration,
    pub extensions: BTreeMap<String, String>, // reverse-DNS keys; the core ignores them
}
impl ExecPlan { pub fn new(program: &str, args: Vec<String>) -> Self { /* defaults */ }
                pub fn allow_env(mut self, name: &str) -> Self { /* push */ } }
#[non_exhaustive] pub struct Sandbox { pub skill_copy: PathBuf, pub home: PathBuf } // core-owned
pub trait LintRules { fn lint(&self, skill_md: &str) -> Vec<Finding>; }
pub trait AgentExec {
    fn plan(&self, sb: &Sandbox) -> Result<ExecPlan, String>;
    fn analyze(&self, out: &ExecOutcome) -> Vec<Finding>;
}
pub trait BuiltinHooks {            // unsealed; implemented in skillsmith-agents
    fn descriptor(&self) -> &Descriptor;
    fn lint_rules(&self) -> Option<&dyn LintRules> { None } // adds to core verify.lint
    fn validator(&self) -> Option<&dyn AgentExec> { None }  // verify.validate
    fn loader(&self) -> Option<&dyn AgentExec> { None }     // verify.load
}
pub trait AgentAdapter: sealed::Sealed { /* same four methods, no defaults */ }
pub struct BuiltIn<H>(pub H);
impl<H> sealed::Sealed for BuiltIn<H> {}
impl<H: BuiltinHooks> AgentAdapter for BuiltIn<H> { /* delegate to self.0 */ }
```

## Minimal example

The following code compiled, and its test passed, with `rustc 1.98.1 --edition 2021 --test`.
It needs the sketch's elided bodies filled in plus these definitions:
`struct Descriptor { id: String, declared: BTreeMap<String, Support> }`,
`#[non_exhaustive] struct ExecOutcome { status: Option<i32>, stdout: String, stderr: String, timed_out: bool }`,
`#[non_exhaustive] enum Finding { Error { code: String, message: String }, Warning { .. } }`,
and `struct Demo { d: Descriptor }`. The core dispatches by capability and injects the executor,
so tests substitute a fake executor (a Cockburn driven port). The test registers
`Box::new(BuiltIn(Demo { .. }))`.

```rust
impl BuiltinHooks for Demo {
    fn descriptor(&self) -> &Descriptor { &self.d }
    fn validator(&self) -> Option<&dyn AgentExec> { Some(self) }
}
impl AgentExec for Demo {
    fn plan(&self, sb: &Sandbox) -> Result<ExecPlan, String> {
        let args = vec!["skills".into(), "validate".into(), sb.skill_copy.display().to_string()];
        Ok(ExecPlan::new("demo-agent", args).allow_env("PATH"))
    }
    fn analyze(&self, out: &ExecOutcome) -> Vec<Finding> {
        match (out.timed_out, out.status) {
            (true, _) => vec![Finding::Error { code: "exec.timeout".into(), message: "timed out".into() }],
            (_, Some(0)) => vec![],
            _ => vec![Finding::Error { code: "demo.invalid".into(), message: out.stderr.clone() }],
        }
    }
}
pub fn run_validate(adapters: &[Box<dyn AgentAdapter>], sb: &Sandbox,
                    exec: impl Fn(&ExecPlan, &Sandbox) -> ExecOutcome) -> Vec<(String, Vec<Finding>)> {
    adapters.iter().filter_map(|a| {
        let v = a.validator()?;                       // capability, not agent name
        let f = match v.plan(sb) { Ok(p) => v.analyze(&exec(&p, sb)),
            Err(e) => vec![Finding::Error { code: "plan.failed".into(), message: e }] };
        Some((a.descriptor().id.clone(), f))
    }).collect()
}
```

## Gotchas

- **A Tier 2 adapter process is not sandboxed by this design.** It has ambient authority from
  its first instruction. cap-std says it "is not a sandbox for untrusted Rust code". Deno treats
  `--allow-run` as equivalent to `--allow-all`. The plan/execute split protects the *agent binary
  run* only. Tier 2 safety rests on the D2 allowlist (path plus SHA-256). Launch the Tier 2
  process itself with a scrubbed environment too.
- **Running the agent binary is full authority.** Declared `exec` permissions limit *which*
  program runs, not what it does. Credential safety comes from the core-owned env scrub and
  temporary `HOME` on every exec. Test it: the conformance suite should plant a canary variable
  and a canary credentials file and assert that neither reaches the child.
- **Dynamic capability sets break caching and the D1 matrix.** A capability that depends on the
  agent version belongs in D1 `Support` (`Unknown` resolved by probe), not in an accessor that
  flips between `Some` and `None`.
- **Listing must not execute.** A Tier 2 program should ship a static `agent.toml` next to it, so
  `sks agents` can describe it before approval (the VS Code and Eclipse lazy-activation lesson).
- **Sealing blocks every other crate, including your own.** A sealed `AgentAdapter` cannot be
  implemented from `skillsmith-agents`, hence the `BuiltinHooks` wrapper. Likewise, a
  `#[non_exhaustive]` struct cannot be built with a struct literal outside its crate, so
  adapter-built types need constructors. A single-file compile does not catch either problem.
- **Do not use a `TypeId` map (`http::Extensions`) on the wire.** It is useful only between
  in-process Rust components; it cannot cross JSON-RPC and cannot be displayed.
- **Plans must reference sandbox paths only.** The core should reject any `ExecPlan` argument
  that resolves outside `Sandbox`, except the program path. Open the sandbox as a cap-std `Dir`
  so that `..` and symlink escapes fail with `PermissionDenied`.

## Currency notes

- **MCP dropped its `initialize` handshake** in the 2026-07-28 specification. Capabilities now
  travel in `_meta` on every request, and there is a new `server/discover` method. LSP 3.17 still
  uses a handshake. ADR D2's "(LSP/MCP pattern)" is therefore half stale. The recommendation does
  not change: a one-shot stdio child still benefits from one `initialize` call. Add a static
  `describe` answer (the `agent.toml`) so listing is stateless.
- **MCP formalised extensions** (SEP-2133): reverse-DNS ids negotiated through an `extensions`
  map. This supports pattern 5.
- **WASI 0.3.0 shipped on 2026-06-11**; Wasmtime 46+ implements it. Crates.io on 2026-09-25 shows
  wasmtime 49.0.1 and extism 1.30.0. ADR D2's "toolchain still stabilising" is dated. Keep WASM
  deferred but re-evaluate it when a third party needs sandboxed *adapter logic*.
- **Trait upcasting is stable** (Rust 1.86); no `as_x()` helpers are needed.
- **async `fn` in `dyn` traits is still not stable**; it has draft-RFC status.

## Recommendation

**Adopt patterns 1 to 5 together.** Confidence: medium-high for patterns 1, 2 and 5 (multiple
independent precedents); medium for pattern 3 (the boundary line is a judgment call to check
against v1's Muse and Claude Code validators).

| Rank | Option | Why not first |
|---|---|---|
| Runner-up | **Protocol-everywhere:** built-ins also speak the JSON-RPC schema in-process (the LSP/Terraform-mux style). | One code path and free wire conformance, but every built-in pays serialisation cost and typed Rust errors are lost. Choose it if Tier 2 adoption dominates. |
| 3 | Single fat trait whose methods return `NotSupported`. | This is v1's boolean with more steps: it cannot tell "not built" from "agent can't", and it hides the capability set from the D1 matrix. |
| 4 | String-keyed `Box<dyn Any>` registry. | Loses static typing; the capability list is core-owned anyway. |
| 5 | WASM adapters now. | They sandbox adapter code, but that code is not the threat; the agent binary is, and WASM does not sandbox it. |
