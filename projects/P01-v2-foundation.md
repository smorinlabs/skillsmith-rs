# P01 — v2 foundation: architecture and spec-first skeleton (v0.1.0)

**References**
- **Trunk:** [PROJECTS.md](../PROJECTS.md)
- **Design:** [ADR 0001 — v2 architecture](../docs/adr/0001-v2-architecture.md)
- **Tracking:** [skillsmith v1 P21 pointer](https://github.com/smorinlabs/skillsmith/blob/main/projects/P21-skillsmith-v2-rust-architecture.md)
- **Prior art:** [v1 ADR 0007 — tool adapter registry](https://github.com/smorinlabs/skillsmith/blob/main/docs/adr/0007-tool-adapter-registry.md)
- **Prior art:** [v1 ADR 0008 — wire contract registry](https://github.com/smorinlabs/skillsmith/blob/main/docs/adr/0008-wire-contract-registry.md)
- **Discussion:** [v1 issue #99 — shared project destination](https://github.com/smorinlabs/skillsmith/issues/99)

**Status:** In progress. Direction approved 2026-09-24 (ADR 0001 decision log). No Rust code is
authorized until the ADR is accepted and T02–T05 are reviewed.

**Goal/Requirement**: Establish the v2 architecture and a Cargo workspace that is proven
compatible with v1's persisted formats before any command is ported.
- Capability model (ADR D1), adapter tiers (D2), locations and bindings (D3), co-ownership (D4).
- Byte-compatible v1 state (D5); cheap agent onboarding (D6); monitoring scope (D7).

**Out of Scope**
- Porting commands; releases; WASM adapters; runtime metrics.

### Tests & Tasks
- [~] [P01-T01] Record ADR 0001 with the decision log.
- [ ] [P01-T02] Analyse the D2 data/code boundary per v1 per-agent behaviour; propose the `agent.toml` schema.
- [ ] [P01-T03] Specify the Tier 2 JSON-RPC adapter protocol (`initialize`, capability exchange, versioning).
- [ ] [P01-T04] Decide the v1/v2 binary name collision (ADR open issue).
- [ ] [P01-T05] Capture v1 golden fixtures for manifest@1, lock@1, plan@1, ledger@2, journal@1.
- [ ] [P01-T06] Scaffold the Cargo workspace (model, formats, adapter, agents, plan, cli).
- [ ] [P01-TS01] `skillsmith-formats` round-trips every v1 golden fixture byte-for-byte.
- [ ] [P01-TS02] Conformance suite runs against every registered adapter, including one Tier 1 test manifest.
- [ ] Regression Test Status

### Deliverable
A workspace whose format crate reads and rewrites v1 state unchanged, plus accepted specs for
the adapter manifest and protocol.

### Automated Verification
- `cargo test --workspace` passes, including v1 fixture round-trips.

### Manual Verification
- Run v1 `skillsmith list --json` before and after a v2 read/write pass on the same data dir; output is identical.
