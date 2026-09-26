---
type: exploratory
status: chosen
created: 2026-09-25
updated: 2026-09-25
confidence: medium
confidence_basis: Lock and codec findings are high (read from v1 source and proper-lockfile 4.1.2 source, lock behaviour run locally); ecosystem patterns rest on official docs/man pages, one source each for several; conformance and migration patterns are analogy, not measurement.
assumptions: v1 keeps proper-lockfile 4.1.2 and its current lock options through the overlap; v2 writes no shared state before milestone 3 (ADR 0001 D10); macOS/Linux local filesystems (not NFS) are the primary targets.
sources: [https://github.com/moxystudio/node-proper-lockfile, https://api.github.com/repos/moxystudio/node-proper-lockfile, https://doc.rust-lang.org/std/fs/struct.File.html, https://man7.org/linux/man-pages/man2/flock.2.html, https://nix.dev/manual/nix/2.28/package-management/garbage-collector-roots, https://nix.dev/manual/nix/2.28/package-management/profiles, https://man7.org/linux/man-pages/man1/dpkg-divert.1.html, https://man7.org/linux/man-pages/man1/dpkg.1.html, https://man7.org/linux/man-pages/man1/update-alternatives.1.html, https://www.gnu.org/software/stow/manual/html_node/Conflicts.html, https://docs.brew.sh/Formula-Cookbook, https://docs.brew.sh/Manpage, https://developer.hashicorp.com/terraform/language/state/locking, https://developer.hashicorp.com/terraform/cli/commands/apply, https://discuss.hashicorp.com/t/question-on-error-saved-plan-is-stale/52912, https://serde.rs/container-attrs.html, https://www.rfc-editor.org/rfc/rfc8785, https://protobuf.dev/programming-guides/proto3/, https://martinfowler.com/bliki/StranglerFigApplication.html, https://github.com/github/scientist, https://github.com/tc39/test262/blob/main/INTERPRETING.md, https://github.com/modelcontextprotocol/conformance, https://github.com/Microsoft/language-server-protocol/issues/353, https://jakarta.ee/committees/specification/tckprocess/, https://insta.rs/docs/advanced/]
origin_prompt: topics/v2-architecture-patterns/prompts/00-landscape.prompt.md
---

# Cluster C — state, installation, and coexistence patterns

**Purpose.** Pick the patterns Skillsmith v2 should use for (1) directories that several agents
share, (2) plan/apply with crash recovery, (3) locking that excludes v1, (4) strict versioned
codecs, (5) running v1 and v2 side by side, and (6) conformance testing of adapters. The durable
answer is in `research/reference/shared-install-state-patterns-2026-09-25.md`. This file records
the survey and the reasoning.

**Headline findings.**

1. The v1 lock parameters that ADR 0001 D5 left "to be read in P01-T05" are now read (below).
2. **v1 has three lock families, not one.** D5 names only `placements.json.lock`. v1's artifact
   coordinator also takes a central lock and per-file "compatibility" locks. v2's milestone-3
   writes must take whichever of these v1 takes for the same files. This is an open finding for
   the ADR author, not resolved here.
3. No Rust crate implementing the proper-lockfile convention was found (search on 2026-09-25).
   v2 should implement it in-house. In a local run, a compiled Rust acquire sketch and
   proper-lockfile 4.1.2 excluded each other.

## 1. Shared install locations with several consumers

| System | Ownership record | Removal rule | Conflict rule |
|---|---|---|---|
| Nix store + profiles | GC roots: symlinks under `/nix/var/nix/gcroots` | Delete what no root reaches, including dependencies | Content-addressed paths never collide |
| dpkg | Per-package file lists in the dpkg database | File removed with its owning package | Second owner is an error unless `--force-overwrite` or a diversion |
| dpkg-divert | Diversion record: path, divert-to, exempt package | Removing the diversion moves the file back (`--rename`) | Aborts if the destination exists |
| update-alternatives | Link group state in the alternatives admin dir | Removing the last alternative removes the link | Highest priority wins in automatic mode; manual mode is sticky |
| GNU stow | None stored; a symlink pointing into the stow dir is owned | Unstow removes only links that point into its package | Two-phase: scan all conflicts, abort before any change |
| Homebrew | Cellar keg per version; `opt/<name>` symlink to active keg | `brew unlink` removes prefix symlinks | Keg-only formulae are never linked; `brew link --overwrite` deletes existing files |

Sources: Nix GC roots and profiles manuals; `dpkg-divert(1)`, `dpkg(1)`,
`update-alternatives(1)` man pages; GNU stow manual "Conflicts"; Homebrew Formula Cookbook and
Manpage.

**Canonical pattern.** An explicit ownership record plus reachability-based deletion. Nix is the
purest form: the root set is the only truth, and deletion is a sweep of what the roots do not
reach. Stow is the opposite: no stored state, ownership inferred from the symlink target. Stow's
approach cannot tell two co-owners apart, so it does not fit D4.

**How each maps to Skillsmith.**

- **Nix roots → D4 co-ownership.** Each (agent, placement) binding in the ledger is a root. A
  placement's files are deleted only when its root set is empty. This is what ADR 0001 D4 already
  says; the survey confirms reachability over reference counts. Counts drift after a crash;
  a root set can be recomputed from the ledger.
- **Stow two-phase → plan before touching disk.** Compute every collision for the whole
  operation first; refuse the operation before the first write if any collision is unresolved.
  Stow added this in 2.0 because partial installs left inconsistent trees.
- **update-alternatives automatic/manual → collision policy with a sticky user override.** When
  two agents' preferred placements conflict, Skillsmith picks by declared precedence (automatic).
  An explicit user choice switches that location to manual and is never silently undone.
- **dpkg-divert → a "foreign file" record.** When a location already holds a file Skillsmith did
  not place, Skillsmith should neither overwrite it (Homebrew `--overwrite`, dpkg
  `--force-overwrite`) nor adopt it silently. It records the file as foreign and reports it.
- **Homebrew `opt` → a stable path per logical skill.** Useful only if v2 later keeps versioned
  copies; not needed for milestone 3.

**Anti-patterns.** Force-overwrite flags as the default conflict path. Reference counters
without a recomputable root set. Inferring ownership from path shape alone (stow) when several
owners are possible.

## 2. Plan/apply and journaled transactions

**Terraform.** `plan -out` saves a plan; `apply <plan>` treats the file as approval. State
carries a `lineage` and a `serial`; applying a saved plan after another write bumped the serial
fails with "Saved plan is stale" (HashiCorp Discuss answer by apparentlymart; Terraform issues
#25981, #27827). State locking is automatic on every write. `force-unlock` requires the lock ID
as a nonce "ensuring that locks and unlocks target the correct lock", and the docs warn that
unlocking another holder's lock "could cause multiple writers".

**dpkg.** Packages move through explicit intermediate states (`half-installed`,
`half-configured`). An interrupted run is resumed with `dpkg --configure -a`. A frontend lock
serialises package managers; `DPKG_FRONTEND_LOCKED` lets a frontend hold it across dpkg calls.

**Nix profiles.** Build the new generation completely, then switch one symlink; the switch "is
atomic on Unix". Old generations remain GC roots, so rollback is another symlink switch.

**v1 already has most of this.** v1's `journal@1` (`packages/core/src/artifacts/journal-types.ts`)
records, per resource, the observed actual state before the change: presence, content hash,
symlink target, and a `repositoryRevision`. That is Terraform's serial idea applied per file: an
apply whose preimage no longer matches is stale.

**Canonical pattern for v2.** Plan (pure, records expected preimages) → take locks → re-read and
compare preimages (compare-and-swap) → append journal intent → perform each step as
write-temp, fsync, rename → record completion → release. Recovery on the next run reads the
journal and either rolls forward or rolls back to the recorded preimages, like
`dpkg --configure -a`.

**Anti-patterns.** Applying a saved plan without revalidating preimages. Recovery that assumes
the process that crashed was v2 (the journal must be readable by both implementations, which D5
already requires). A `force-unlock` without a holder nonce.

## 3. Cross-language file locking

### What v1 actually does (read 2026-09-25 from v1 source)

| Lock | Target passed to proper-lockfile | Lock dir on disk | `stale` | `update` | Retry delays (ms) | Where |
|---|---|---|---|---|---|---|
| Ledger | `<data>/placements.json` | `placements.json.lock` | 30000 | 5000 | 100, 200, 400, 800, 1600 | `ports/default.ts:49,408-412`; `place/ledger.ts:403-428` |
| Artifact central | `~/.skillsmith/coordination/artifacts-v1/global` | `global.lock`, mode 0700 | 2000 | 1000 | 0, 100, 200, 400, 800, 800 | `artifacts/coordinator.ts:413`; `artifacts/coordinator-types.ts:7-9`; `artifacts/node-coordinator.ts:663` |
| Artifact compatibility | each target file, acquired in sorted canonical order | `<target>.lock`, plus a member-marker JSON | 30000 | 5000 | same as central | `artifacts/coordinator.ts:300-345`; `artifacts/node-coordinator.ts:562-600` |

All three pass `realpath: false` and `retries: 0` (v1 runs its own retry ladder). v1 pins
`proper-lockfile` at exactly `4.1.2` (`packages/core/package.json`).

### What proper-lockfile 4.1.2 does (read from `lib/lockfile.js`, `lib/mtime-precision.js`)

- **Lock identity.** The lock is the directory `<file>.lock`, where `<file>` is
  `path.resolve(file)` when `realpath: false`. Symlinks are **not** followed; the target need not
  exist.
- **Acquire.** `mkdir(<file>.lock)`. On success, set the dir mtime to
  `ceil(now/1000)*1000 + 5` ms and read it back; if the stored mtime is a whole second, the
  filesystem has second precision.
- **Contention.** On `EEXIST`, `stat` the dir. If `mtime < now - stale`, `rmdir` it and retry
  once with staleness checks disabled. Otherwise fail with `ELOCKED`. The acquirer decides
  staleness; the holder never sees it happen.
- **Refresh.** Every `update` ms: `stat`; if the mtime is not the one it last wrote, the lock is
  compromised. Otherwise `utimes` to now (rounded up to the second on second-precision
  filesystems). On a failed refresh, retry after 1000 ms unless the dir is gone (`ENOENT`) or
  `lastUpdate + stale < now`; either condition marks the lock compromised.
- **Clamping.** `stale = max(stale, 2000)`; `update = max(min(update ?? stale/2, stale/2), 1000)`.
- **Compromise.** Calls `onCompromised(err)` with code `ECOMPROMISED`. The default throws. Because
  it runs inside a timer callback, the default is an uncaught exception, which exits Node. v1's
  ledger lock uses this default; the artifact coordinator overrides it.
- **Release.** `rmdir`. On process exit, `signal-exit` `rmdirSync`s every held lock. `SIGKILL`
  leaves the directory; a peer reclaims it after `stale`.

**Source disagreement.** The README says `stale` has a minimum of 5000 ms. The 4.1.2 source
clamps at 2000 ms, and v1's central lock passes exactly 2000. The source wins for interop.
Verified locally: a foreign lock dir aged 1.5 s was not reclaimed with `stale: 500` (clamped to
2000); aged 2.5 s it was.

**Maintenance status.** The GitHub API reports the repo not archived but last pushed 2023-10-25
with 21 open issues: dormant. Maintained forks exist (`@alcalzone/proper-lockfile`,
`proper-lockfile2`). A v1 switch to a fork could change behaviour, so v2's compatibility tests
must pin to the version v1 actually ships.

### Rust interop

- No crate implements this convention (crates.io search, 2026-09-25; the `lockfile` crate uses
  create-file semantics). Implement in-house.
- `std::fs::File::lock` / `try_lock` (stable since Rust 1.89) use `flock(LOCK_EX)` on Unix and
  `LockFileEx` on Windows. On local Linux, flock and fcntl locks do not interact; on NFS since
  Linux 2.6.12 flock is emulated with fcntl byte-range locks and they do interact (`flock(2)`).
  None of these share identity with a mkdir lock dir, so they cannot exclude v1. They are fine
  for v2-only files.
- flock is released by the kernel when the process dies; mkdir locks are released only by the
  staleness timeout. That is the cost of matching v1.

**Anti-pattern (seen in v1's history).** Locking a sidecar path instead of the target: passing
`placements.json.lock` as the target created `placements.json.lock.lock`, and old and new
binaries stopped excluding each other (`place/ledger.ts:420-426`). The Rust equivalent is calling
`canonicalize()` on the target: through a symlinked data dir it produces a different lock path.

## 4. Schema and codec versioning with strict unknown-field rejection

- **serde.** `#[serde(deny_unknown_fields)]` makes deserialization "always error ... when
  encountering unknown fields". It is per container: every nested struct needs it. It is "not
  supported in combination with `flatten`". A `serde_json::Value` field accepts anything, so it
  defeats strictness.
- **Byte-exact re-serialization.** v1 records per-codec `presentation.encode:
  canonical|compatibility`, `terminalLf`, and `unknownFields: 'reject-recursive'`
  (`packages/core/src/artifacts/codec.ts`). v2's formats crate should mirror this descriptor.
  Rust must match JavaScript's output: declaration-order keys, integers not floats (serde_json
  writes an `f64` 1 as `1.0`; JavaScript writes `1`), JavaScript string escaping, and the same
  indentation and final newline. Fixtures prove it: decode then encode must return the same bytes.
- **JCS (RFC 8785)** sorts keys by UTF-16 code units and prints numbers as ECMAScript does. It is
  contrast only; v1 is not JCS, and switching would break D5.
- **Protobuf** is the opposite policy: never reuse field numbers, reserve deleted ones, and
  preserve unknown fields on re-serialization. That suits wire protocols that evolve
  independently. Skillsmith's persisted files are owned by two cooperating writers, so rejecting
  unknown fields is safer: a v2 that silently drops a v1-only field would corrupt v1's state.
  The consequence is that neither side can add a field to a shared format until v1 retires
  (already stated in D5).

## 5. Strangler fig and parallel run

- Fowler's strangler fig (updated 2024-08-22) grows the new system on seams and accepts
  "transitional architecture ... that will go away once the modernization is complete".
- GitHub's Scientist runs control and candidate, compares, and publishes mismatches, but warns
  it "is only safe for wrapping methods that aren't changing data".

**Mapping.** D10's order is already the safe one: read-only commands first (Scientist-safe),
writes last, after the cross-implementation lock test. For reads, a parallel-run harness can run
`skillsmith list --json` (v1) and `sks list --json` (v2) over the same data dir and compare
normalised models. Because D8 makes the output schemas differ, the comparison must be over
decoded models, not bytes. For writes, the parallel run is the P01-TS03 concurrency test plus
round-trip fixtures: v1 writes, v2 reads and rewrites, v1 reads back.

## 6. Conformance suites and golden tests

| Suite | Mechanism worth copying |
|---|---|
| test262 | Per-test frontmatter: `features` lets a host skip what it does not support; `negative` declares expected errors |
| MCP conformance (`modelcontextprotocol/conformance`) | Scenarios with named checks; a per-SDK YAML baseline of expected failures (`<scenario>:<check-id>`); failure in baseline = exit 0, new failure = exit 1; a composite GitHub Action for SDK repos |
| Jakarta EE TCK | "Specifications are the sole source of truth"; a challenge process and exclude lists; self-certification with published results |
| LSP | No official conformance suite (microsoft/language-server-protocol issue #353); community tools such as pytest-lsp |
| insta `glob!` | One test runs a closure per matching input file and names each snapshot after it |

**Mapping to D6.** One suite parameterized over the adapter registry. Each check is tagged with
the D1 capability it exercises, like test262 `features`; an adapter that does not declare a
capability skips its checks. Checks for a declared capability must pass, or be listed in the
adapter's baseline file with a tracking reference, like MCP's baseline. The same harness should
run against Tier 2 external adapters (a third party's CI can use it, as MCP SDKs do). Golden
fixtures use `insta::glob!` over `fixtures/<artifact>@<version>/*`, so v1's fixtures become v2's
compatibility tests without per-file test code.

## Recommendation and runner-up

**Recommended set (confidence medium-high; basis: v1 and library source read and run, ecosystem
patterns from primary docs).** Ledger root sets with reachability GC (Nix); two-phase
plan-then-apply with no write if any collision is unresolved (stow); sticky manual override of
collision policy (update-alternatives); preimage compare-and-swap plus a shared `journal@1`
(Terraform serial, dpkg resume); an in-house proper-lockfile-compatible mkdir lock that reproduces
all three v1 lock families exactly; strict serde codecs with byte-exact round-trip fixtures; a
capability-tagged conformance suite with baselines; read-only parallel run before any writes.

**Runner-up: v2 uses OS locks (`File::lock`) plus a v1-owned "bridge" lock.** v2 would take a
flock on a new v2-only file and, in addition, the mkdir lock only when writing shared files.
Rejected for now: it adds a second lock ordering to reason about and still needs the mkdir
implementation. It becomes the better choice after v1 retires (conditional: when v1 is gone,
switch to `File::lock` for its automatic release on crash).

**Why not the others.** Stow-style inferred ownership cannot represent co-owners. Protobuf-style
unknown-field preservation risks silent divergence between two writers. JCS would break byte
compatibility with v1. Scientist-style dual execution of writes is unsafe by its own docs.

## Open items for the ADR author

- D5 says "the same lock". v1 has three lock families (§3). Decide which v2 writes take which
  locks, and extend P01-TS03 to cover the artifact coordinator locks.
- v1's ledger lock uses the default `onCompromised` (process exit on compromise). v2 should
  instead abort before its commit rename; decide whether v1 should change too.
- Record in the ADR that v1 pins `proper-lockfile@4.1.2`, and that a v1 change of lock library
  or options is a breaking change for coexistence.
