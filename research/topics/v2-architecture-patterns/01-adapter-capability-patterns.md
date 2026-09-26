---
type: exploratory
status: chosen
created: 2026-09-25
updated: 2026-09-25
confidence: medium
confidence_basis: Load-bearing claims are cross-checked against 2+ primary sources (specs, official docs, source protos, crates.io); a few context claims (LSP experimental, VS Code contribution points, Eclipse registry) are single-sourced and are not load-bearing.
assumptions: Synchronous adapter calls; third parties never ship in-process Rust; the core spawns every agent binary; ADR 0001 D1, D2, D6 and D11 stand as written.
sources: ["https://alistair.cockburn.us/hexagonal-architecture", "https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nf-unknwn-iunknown-queryinterface(refiid_void)", "https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/", "https://modelcontextprotocol.io/specification/2025-11-25/basic/index", "https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/", "https://code.visualstudio.com/api/references/contribution-points", "https://code.visualstudio.com/api/advanced-topics/using-proposed-api", "https://help.eclipse.org/latest/topic/org.eclipse.platform.doc.isv/guide/runtime_registry.htm", "https://backstage.io/docs/backend-system/architecture/extension-points", "https://developer.hashicorp.com/terraform/plugin/terraform-plugin-protocol", "https://raw.githubusercontent.com/hashicorp/terraform/main/docs/plugin-protocol/tfplugin6.proto", "https://raw.githubusercontent.com/hashicorp/terraform/main/docs/resource-instance-change-lifecycle.md", "https://github.com/bytecodealliance/cap-std", "https://docs.rs/cap-std/latest/cap_std/", "https://bytecodealliance.org/articles/WASI-0.3", "https://docs.deno.com/runtime/fundamentals/security/", "https://zellij.dev/documentation/plugin-api-permissions.html", "https://zed.dev/docs/extensions/capabilities", "https://developer.android.com/guide/topics/permissions/overview", "https://docs.rs/http/latest/http/struct.Extensions.html", "https://rust-lang.github.io/api-guidelines/future-proofing.html", "https://doc.rust-lang.org/reference/attributes/type_system.html", "https://blog.rust-lang.org/2025/04/03/Rust-1.86.0/", "https://rust-lang.github.io/async-fundamentals-initiative/explainer/async_fn_in_dyn_trait.html", "https://github.com/rust-lang/rust/issues/99301"]
origin_prompt: topics/v2-architecture-patterns/prompts/00-landscape.prompt.md
---

# Cluster A: capability-based adapter architecture (exploratory funnel)

**Purpose.** Choose how Skillsmith v2 adapters expose capabilities, what the core hands them,
and how third parties extend them safely. **Scope:** cluster A of the landscape prompt only.
**Outcome:** the durable recommendation is in
[capability-adapter-patterns-2026-09-25.md](../../reference/capability-adapter-patterns-2026-09-25.md).
This file records every option considered.

Terms used below: **core** means Skillsmith's own library code. An **adapter** is one agent
integration (ADR D2 Tier 0 built-in Rust, Tier 1 `agent.toml`, or Tier 2 JSON-RPC program).
A **capability** is a named function such as `verify.validate` (ADR D11).

## A1. Ports and adapters (hexagonal)

**Canonical pattern.** Cockburn's stated intent is an application "equally driven by users,
programs, automated test or batch scripts, and … developed and tested in isolation from its
eventual run-time devices." *Primary* (driving) actors start work; *secondary* (driven) actors
are called by the application and are naturally replaced by mocks.

**Applied to Skillsmith.** Skillsmith has two directions that are easy to confuse:
- The CLI and embedders are **driving** adapters of the core.
- Agent adapters are **driven** adapters: the core calls them.
- The core also offers **driven ports the adapter consumes**: process execution, the sandbox
  filesystem, and probes. Making these core-owned services, rather than letting adapters use
  `std::process` directly, is what gives the core sandbox control. The object-capability
  section below builds on this.

**Anti-pattern.** Adapters that reach ambient authority (`std::fs`, `Command::new`) directly.
The core then cannot guarantee environment scrubbing, and tests need real agent binaries.

## A2. Role interfaces and capability discovery

| Mechanism | How it works | Lesson for Skillsmith |
|---|---|---|
| COM `QueryInterface` | Ask an object for an interface id; get a pointer or `E_NOINTERFACE`. The rules require the set to be **static**, reflexive, symmetric and transitive, and forbid ACL checks inside `QueryInterface`. | Accessors returning `Option<&dyn Cap>` must give a stable answer per adapter. Authorization belongs elsewhere. |
| LSP 3.17 | `initialize` exchanges `ClientCapabilities`/`ServerCapabilities`. Receivers "should ignore" unknown properties. `experimental?: LSPAny` holds untyped extras. Dynamic registration exists. | Ignore unknown fields on the wire, but that conflicts with v1's strict codecs, so apply it to the protocol only and never to persisted state. |
| MCP 2025-11-25 → 2026-07-28 | The handshake plus capabilities was replaced by per-request capabilities in `_meta`, plus `server/discover`. Extensions are negotiated in an `extensions` map with reverse-DNS ids (SEP-2133). | See the "Recent changes" section. The reverse-DNS naming scheme is directly reusable. |
| Terraform plugin protocol v6 | `ServerCapabilities` and `ClientCapabilities` are **boolean feature flags** (`plan_destroy`, `move_resource_state`, `deferral_allowed`, …). | Flat, additive, named flags age well and are a good model for protocol-level features, as distinct from agent capabilities. |

**Options considered for the Rust surface:**
1. *Fat trait with `NotSupported` defaults.* This is v1's boolean in disguise. It cannot express
   D1's two axes and it hides the capability set. **Rejected.**
2. *Base trait plus fixed accessor methods* (`fn validator(&self) -> Option<&dyn AgentExec>`).
   These are typed and discoverable, and the static set can be checked by the D6 conformance
   suite. **Chosen.**
3. *String-keyed registry of `Box<dyn Any>`* (a literal `QueryInterface` port). Open-ended, but
   loses types, and the capability list is core-owned anyway. **Rejected** while the list is
   closed; it fits custom capabilities only if they ever run in-process.
4. *Enum of capabilities with payload* (`enum Cap { Validate(..), Load(..) }`). Closed and
   exhaustive, but every adapter must match all variants. **Rejected** for the adapter API; it
   is fine as the internal dispatch table.

## A3. Extension points and contribution models

- **VS Code:** declarative `contributes` in `package.json` lets VS Code show commands before it
  activates the extension (lazy activation through `onCommand:`). Unstable APIs are gated behind
  `enabledApiProposals` and "should not be used in published extensions". This is a gated
  escape hatch.
- **Eclipse:** `plugin.xml` extension points are read from the registry without loading plug-in
  classes. Declaration is separated from activation. (Single source; context only.)
- **Backstage:** a plugin creates typed extension points (`createExtensionPoint<T>({ id })`).
  Modules extending that plugin register into it and are "completely initialized before our
  plugin gets initialized". Extension points are visible only to modules of that plugin.

**Lesson.** Every system separates a **static declaration** (read without running code) from
**activation**. Skillsmith already has the declaration: the Tier 1 manifest. Apply it to Tier 2
too. A Tier 2 program ships an `agent.toml` beside it, so `sks agents` lists it without executing
an unapproved binary. This dovetails with ADR D2's Tier 2 trust rule. Backstage's typed
extension-point object is the right model if core commands ever accept contributed lint rules
from outside an adapter. Not needed now.

## A4. Plugin boundaries

| Boundary | Isolation | Cost | Exemplar | Fit |
|---|---|---|---|---|
| In-process trait | none | lowest | rustc lints, cargo subcommands linked in | Tier 0 only |
| Data manifest | total (no code) | expressiveness | VS Code `contributes`, mise registry | Tier 1 |
| Out-of-process protocol | process-level; the plugin still has ambient OS authority | serialisation, versioning | LSP, Terraform gRPC, MCP stdio | Tier 2 |
| WASM component | strong (WASI capabilities) | toolchain, host API design | Zed, Zellij, Extism | deferred |

**Key insight: plan / execute / analyze.** The dangerous act in Skillsmith is running the
*agent* binary (credentials, `HOME`), not running adapter logic. Terraform separates
`PlanResourceChange` from `ApplyResourceChange`. Terraform Core enforces that "any attribute
that had a known value in the Final Planned State must have an identical value in the new state."
The core stays the arbiter of what happened. Skillsmith applies this as follows:
1. The adapter's `plan` returns an `ExecPlan` as data.
2. The core executes that plan in a sandbox it owns.
3. The adapter's `analyze` turns the outcome into findings.

This works identically for all three tiers. It also answers P01-T02, the data/code boundary.
Tier 1 can express `plan` as an argv template, env allowlist and timeout, and `analyze` as an
exit-code map plus regex rules. Anything beyond that is Tier 0 or Tier 2 code.

## A5. Object-capability security for services handed to adapters

- **cap-std** (4.0.3): a `Dir` handle replaces paths. Access outside the directory, including
  through `..` and symlinks, returns `PermissionDenied`. Ambient access requires an explicit
  `ambient_authority()` token. The README's caveat: "not a sandbox for untrusted Rust code". It
  guards against malicious *paths*, not malicious *code*.
- **WASI** is the same model enforced by a VM. WASI 0.3.0 shipped on 2026-06-11 with native
  async; Wasmtime 46+ implements it.

**Applicability.** Pass Tier 0 adapters a `Sandbox` holding cap-std `Dir`s and a core
`ProcessRunner`, never raw paths. This is a *discipline* for trusted code: Rust cannot stop
Tier 0 code from calling `std::fs`. A lint or review rule can back it up. For Tier 1 and Tier 2,
the process boundary plus plan/execute makes the discipline structural.

## A6. Declared-permission escape hatches

| System | Declared where | Granted how | Notable caveat |
|---|---|---|---|
| Deno | CLI flags, scoped (`--allow-run=git`) | flags or interactive prompt; `--deny-*` overrides | "`--allow-run` … escaping the sandbox entirely"; treat `run` and `ffi` as `--allow-all` |
| Zellij | plugin calls `request_permission` | user prompt; 14 typed permissions (`RunCommands`, `FullHdAccess`, …) | coarse permission kinds |
| Zed | `extension.toml` capabilities | user `granted_extension_capabilities` with `process:exec` `command`/`args` globs | matches Skillsmith's need almost exactly |
| Android | manifest | install-time (normal, signature) vs runtime (dangerous) vs special | "request only the permissions that it needs … as late … as possible" |

**Sources agree** that exec permissions do not confine what the executed program does (Deno
explicitly; Zed implicitly, because the allowlist is by command). **Consequence:** declared
`exec` permissions in Skillsmith are a *display, approval and drift-check* mechanism. The
credential boundary is the core-owned environment scrub and temporary `HOME` on every exec.

## A7. Data escape hatches and namespacing

- **LSP `experimental?: LSPAny`:** a single untyped bucket. The risk is key collisions between
  vendors.
- **MCP `_meta`:** keys are an optional prefix plus a name. The prefix is dot-separated labels
  followed by `/`, "SHOULD use reverse DNS". Any prefix whose second label is `modelcontextprotocol`
  or `mcp` is reserved, and implementations "MUST NOT make assumptions" about values at reserved
  keys.
- **`http::Extensions`:** "A type map of protocol extensions", keyed by `TypeId`, with values
  that are `Clone + Send + Sync + 'static`. It works only in-process and is invisible to
  serialisation.

**Chosen:** `extensions: map<string, json>` on wire objects, using MCP key syntax, with
`dev.skillsmith/` reserved. The core preserves and ignores unknown keys. Custom capabilities
reuse the syntax (`com.example/verify.secrets`); core ids stay bare. **Interaction with
D5/ADR 0008:** strict unknown-field rejection still applies to persisted v1 artifacts. The
extensions map is the one sanctioned place for unknown data in v2-only files and protocol
messages.

## A8. Rust idioms

| Idiom | Use in Skillsmith | Source |
|---|---|---|
| `Box<dyn AgentAdapter>` registry | yes: tiers are chosen at runtime | — |
| Generics or `impl Trait` | inside the core (injected executor) for tests; not in the registry | — |
| Enums | closed internal sets (tier, derived D1 state) | — |
| Sealed traits (C-SEALED) | seal `AgentAdapter`. Sealing blocks *every* other crate, including `skillsmith-agents`, so its only implementors are the tier wrappers `BuiltIn<H>`, `Manifest` and `External` in `skillsmith-adapter`. Tier 0 modules implement an unsealed `BuiltinHooks` trait that `BuiltIn<H>` wraps. Unsealing later is non-breaking; sealing later is breaking. | Rust API guidelines |
| `#[non_exhaustive]` | all public enums (`Support`, `Finding`) and core-built structs (`Sandbox`, `ExecOutcome`). Outside the crate, struct literals fail and enum matches need `_`. `ExecPlan` is adapter-built, so give it `new()` plus `with_*` or `allow_*` setters before marking it. | Rust Reference |
| Trait upcasting | stable since 1.86; removes `as_supertrait()` helpers | Rust 1.86 release notes |
| Async `dyn` traits | avoid: still a draft RFC. Use a synchronous trait with `std::process` and timeouts. | async-fundamentals explainer |
| `Error::provide` / generic member access | would give a std `request_ref::<T>()`, but still unstable (#99301). Don't depend on it. | rust-lang/rust#99301 |

## Recent changes (what is now considered dated or an anti-pattern)

1. **MCP removed its stateful `initialize` handshake** (2026-07-28 spec; release candidate on
   2026-05-21, final on 2026-07-28). Capabilities now travel on every request's `_meta`, with an
   optional `server/discover`. **Source disagreement:** LSP 3.17 still requires a handshake.
   ADR D2's "(LSP/MCP pattern)" is now half stale. **Effect on the recommendation:** none for a
   one-shot stdio child; one `initialize` call is still cheapest. Add the static `agent.toml`
   description so listing needs no process.
2. **Extensions became formal in MCP** (SEP-2133), which supports reverse-DNS namespacing.
3. **WASI 0.3.0 is final** (2026-06-11). ADR D2's "toolchain still stabilising" is dated. WASM
   remains deferred for a different reason (see Recommendation).
4. **Trait upcasting stable** (Rust 1.86, 2025-04-03).

## Applicability summary

| Pattern | Direct or adapted | Adaptation |
|---|---|---|
| Hexagonal driven ports | direct | adapters consume core ports (exec, sandbox) as well as implementing one |
| QueryInterface accessors | adapted | fixed typed accessors, not id lookup; static set checked by D6 |
| LSP/MCP negotiation | adapted | handshake for Tier 2 plus static manifest; flags for protocol features only |
| Terraform plan/apply | adapted | plan, execute, analyze; the core owns execute |
| cap-std / WASI | adapted | discipline for Tier 0; structural for Tier 1 and Tier 2 via the process boundary |
| Zed/Deno permissions | adapted | display and drift check, not a security boundary |
| MCP `_meta` naming | direct | extensions map plus custom capability ids |

## Recommendation

1. **Chosen:** base `AgentAdapter` with static `Option<&dyn Cap>` accessors, plus
   plan/execute/analyze for agent-running capabilities, plus core-owned sandbox services, plus
   declared `exec` permissions (for display only), plus a reverse-DNS extensions map, plus sealed
   traits, `#[non_exhaustive]` and synchronous calls. Confidence: medium-high, because each part
   has 2+ independent precedents. The combination is untested.
2. **Runner-up: protocol-everywhere.** Built-ins implement the JSON-RPC schema in-process, so
   there is one code path and wire conformance comes free. Choose it if most adapters end up
   being Tier 2. Why not now: serialisation overhead and loss of typed errors for the common case.
3. **Why not WASM first:** it sandboxes adapter *logic*, which is not the threat. The agent
   binary runs natively regardless. Re-evaluate only if third parties need untrusted adapter
   logic that Tier 1 cannot express.
4. **Conditional:** if in-process third-party Rust adapters are ever allowed, unseal the traits
   and add a Backstage-style typed extension-point registry.
