# P01 — v2 foundation: architecture and spec-first skeleton (v0.1.0)

**References**
- **Trunk:** [PROJECTS.md](../PROJECTS.md)
- **Design:** [ADR 0001 — v2 architecture](../docs/adr/0001-v2-architecture.md)
- **Tracking:** [skillsmith v1 P21 pointer, PR #113](https://github.com/smorinlabs/skillsmith/pull/113) — update to the `main` file path once merged
- **Prior art:** [v1 ADR 0007 — tool adapter registry](https://github.com/smorinlabs/skillsmith/blob/main/docs/adr/0007-tool-adapter-registry.md)
- **Prior art:** [v1 ADR 0008 — wire contract registry](https://github.com/smorinlabs/skillsmith/blob/main/docs/adr/0008-wire-contract-registry.md)
- **Discussion:** [v1 issue #99 — shared project destination](https://github.com/smorinlabs/skillsmith/issues/99)

**Status:** In progress. Direction approved 2026-09-24 (ADR 0001 decision log). No Rust code is
authorized until the ADR is accepted and T02–T05 are reviewed.

**Goal/Requirement**: Establish the v2 architecture and a Cargo workspace that is proven
compatible with v1's persisted formats before any command is ported.
- Capability model (ADR D1), adapter tiers (D2), locations and bindings (D3), co-ownership (D4).
- Byte-compatible v1 state (D5); cheap agent onboarding (D6); monitoring scope (D7).
- New, non-v1-compatible command output (D8); `sks` / `skillsmith` naming (D9).

**Out of Scope**
- Commands beyond `agents`, `list`, and `verify --check lint` (D10); any write to shared state;
  releases; WASM adapters; runtime metrics.

## Tests & Tasks
- [~] [P01-T01] Record ADR 0001 with the decision log.
- [ ] [P01-T02] Analyse the D2 data/code boundary per v1 per-agent behaviour; propose the `agent.toml` schema. Prerequisite for `verify --check load`, the next milestone (D10).
- [ ] [P01-T03] Specify the Tier 2 JSON-RPC adapter protocol (`initialize`, capability exchange, versioning).
- [x] [P01-T04] Decide the v1/v2 binary name collision — ADR D9 (Q7.A, 2026-09-25).
- [ ] [P01-T05] Capture v1 golden fixtures for manifest@1, lock@1, plan@1, ledger@2, journal@1.
- [x] [P01-T07] Decide the initial v2 command subset — ADR D10 (Q8.A, 2026-09-25): read-only `agents`, `list`, `verify --check lint`.
- [x] [P01-T09] Name the verification levels — ADR D11 (Q9.A, 2026-09-25): `--check lint` / `--check load`; extended by Q10.A to `lint` / `validate` / `load`, all selected checks always run.
- [ ] [P01-T10] Extend `ExecPlan` for stdin exchanges and multi-step runs (Codex JSON-RPC, Muse two runs), or keep those checks in Tier 0/Tier 2 code (ADR D2 open item).
- [x] [P01-T11] Research architecture patterns — `research/topics/v2-architecture-patterns/DECISION.md`; adopted as ADR D2, D5, D12 (Q16.A, 2026-09-26).
- [ ] [P01-T08] Define v2 output schemas with v2-only identifiers (D8).
- [ ] [P01-T06] Scaffold the Cargo workspace (model, formats, adapter, agents, plan, cli).
- [ ] [P01-TS01] `skillsmith-formats` round-trips every v1 golden fixture byte-for-byte.
- [ ] [P01-TS03] Concurrency test: v1 and v2 on one data dir exclude each other via `placements.json.lock` (D5).
- [ ] [P01-TS02] Conformance suite runs against every registered adapter, including one Tier 1 test manifest.
- [ ] Regression Test Status

## Deliverable
A workspace whose format crate reads and rewrites v1 state unchanged, plus accepted specs for
the adapter manifest and protocol.

## Automated Verification
- `cargo test --workspace` passes, including v1 fixture round-trips.

## Manual Verification
- Run v1 `skillsmith list --json` before and after a v2 read-only pass (`sks list`) on the same data dir; output is identical.
- P01 format round-trips run on isolated fixture copies, never on a live data dir; shared-state writes wait for milestone 3 (ADR D10).
