---
type: terminal
status: current
created: 2026-09-25
updated: 2026-09-25
confidence: medium
confidence_basis: Lock parameters and proper-lockfile behaviour are high (read from v1 source and proper-lockfile 4.1.2 source, core behaviour run locally); install and conformance patterns come from official docs and man pages, mapped to Skillsmith by analogy.
verified_example: true
verified_example_note: proper-lockfile 4.1.2 from the v1 checkout was run under Node on 2026-09-25 — lock dir created empty, target not created, mtime refreshed, second acquire ELOCKED, release removes dir, stale:500 clamped to 2000 ms, 2.5 s-old dir reclaimed, non-empty stale dir fails ENOTEMPTY. The Rust lock sketch below was compiled (rustc, edition 2021) and run against proper-lockfile 4.1.2 on the same target — each side got ELOCKED/None while the other held the lock. The insta round-trip test is illustrative and was not compiled.
assumptions: v1 keeps proper-lockfile 4.1.2 and its current options through the overlap; local macOS/Linux filesystems; v2 writes no shared state before milestone 3 (ADR 0001 D10).
sources: [https://github.com/moxystudio/node-proper-lockfile, https://doc.rust-lang.org/std/fs/struct.File.html, https://man7.org/linux/man-pages/man2/flock.2.html, https://nix.dev/manual/nix/2.28/package-management/garbage-collector-roots, https://man7.org/linux/man-pages/man1/update-alternatives.1.html, https://man7.org/linux/man-pages/man1/dpkg-divert.1.html, https://www.gnu.org/software/stow/manual/html_node/Conflicts.html, https://developer.hashicorp.com/terraform/language/state/locking, https://discuss.hashicorp.com/t/question-on-error-saved-plan-is-stale/52912, https://serde.rs/container-attrs.html, https://protobuf.dev/programming-guides/proto3/, https://github.com/github/scientist, https://github.com/tc39/test262/blob/main/INTERPRETING.md, https://github.com/modelcontextprotocol/conformance, https://insta.rs/docs/advanced/]
origin_prompt: topics/v2-architecture-patterns/prompts/00-landscape.prompt.md
---

# Shared install state, locking, and coexistence patterns for Skillsmith v2

## The specific knowledge

### Co-owned install locations

- **Root sets, not counters (Nix GC roots).** Each (agent, placement) binding in the ledger is a
  root. Delete a placement's files only when its root set is empty. A root set can be recomputed
  from the ledger after a crash; a counter cannot.
- **Scan everything, then write (GNU stow ≥ 2.0).** Compute all collisions for the operation;
  abort before the first write if any is unresolved.
- **Automatic vs sticky manual choice (update-alternatives).** Declared precedence picks the
  winner automatically. An explicit user choice marks that location manual; Skillsmith never
  silently reverts it.
- **Foreign files are recorded, not overwritten (contrast dpkg `--force-overwrite`, Homebrew
  `brew link --overwrite`).** A pre-existing file Skillsmith did not place is reported as foreign.

### Plan, apply, journal

1. Plan is pure and records each resource's expected preimage (presence, content hash, symlink
   target). v1's `journal@1` already carries these fields
   (`packages/core/src/artifacts/journal-types.ts`).
2. Take locks. Re-read preimages; any mismatch means the plan is stale (Terraform's
   lineage/serial check).
3. Append journal intent. Apply each step as write-temp, fsync, rename, fsync the directory.
4. Record completion; release locks.
5. On the next start, an unfinished journal is rolled forward or back (dpkg `--configure -a`).
   Either implementation may find the other's journal.

### v1-compatible locking (read from v1 source, 2026-09-25)

v1 uses `proper-lockfile@4.1.2` (exact pin) with **three lock families**:

| Lock | Lock directory | `stale` ms | `update` ms | v1 retry delays ms |
|---|---|---|---|---|
| Ledger | `<data>/placements.json.lock` | 30000 | 5000 | 100, 200, 400, 800, 1600 |
| Artifact central | `~/.skillsmith/coordination/artifacts-v1/global.lock` (mode 0700) | 2000 | 1000 | 0, 100, 200, 400, 800, 800 |
| Artifact per-target | `<target>.lock`, taken in sorted order, plus a member-marker JSON | 30000 | 5000 | same |

All pass `realpath: false`, `retries: 0`. Sources: v1 `ports/default.ts:49,408-412`,
`artifacts/coordinator-types.ts:7-11`, `artifacts/coordinator.ts:413`,
`artifacts/node-coordinator.ts:562-600,663`. ADR 0001 D5 names only the ledger lock; v2's writes
must also take the artifact locks v1 takes for the same files.

**proper-lockfile 4.1.2 algorithm, which v2 must reproduce:**

- Lock path = `path.resolve(target) + ".lock"`. No symlink resolution. Target need not exist.
- Acquire = `mkdir`. On `EEXIST`: `stat`; if `mtime < now − stale`, `rmdir` and retry once
  without the staleness check; otherwise `ELOCKED`.
- Effective values: `stale = max(stale, 2000)`; `update = max(min(update, stale/2), 1000)`.
- Holder refreshes every `update` ms: `stat`; if mtime ≠ the value last written →
  compromised; else `utimes(now)` (ceil to whole seconds on second-precision filesystems).
  Failed refresh retries after 1000 ms unless `ENOENT` or `lastUpdate + stale < now` →
  compromised.
- Release = `rmdir`. Signal exit also `rmdir`s. `SIGKILL` leaves the dir until it goes stale.

## Minimal example

Rust sketch of the acquire half, refresh described in comments (compiled and interop-tested against proper-lockfile 4.1.2 on 2026-09-25):

```rust
use std::{fs, io, path::{Path, PathBuf}, time::{Duration, SystemTime}};

pub struct MkdirLock { dir: PathBuf, stale: Duration, last_mtime: SystemTime }

fn lock_dir(target: &Path) -> io::Result<PathBuf> {
    // path.resolve semantics: absolutize, do NOT canonicalize (symlinks not followed).
    let abs = std::path::absolute(target)?;
    let mut s = abs.into_os_string(); s.push(".lock"); Ok(s.into())
}

pub fn try_acquire(target: &Path, stale_ms: u64) -> io::Result<Option<MkdirLock>> {
    let stale = Duration::from_millis(stale_ms.max(2000));
    let dir = lock_dir(target)?;
    for check_stale in [true, false] {
        match fs::create_dir(&dir) {
            Ok(()) => {
                let now = SystemTime::now();
                fs::File::open(&dir)?.set_modified(now)?;
                let last_mtime = fs::metadata(&dir)?.modified()?; // read back: FS precision
                return Ok(Some(MkdirLock { dir, stale, last_mtime }));
            }
            Err(e) if e.kind() == io::ErrorKind::AlreadyExists && check_stale => {
                match fs::metadata(&dir) {
                    Err(e) if e.kind() == io::ErrorKind::NotFound => continue,
                    Err(e) => return Err(e),
                    Ok(m) if m.modified()? + stale < SystemTime::now() => {
                        match fs::remove_dir(&dir) {           // rmdir, never remove_dir_all
                            Err(e) if e.kind() != io::ErrorKind::NotFound => return Err(e),
                            _ => continue,
                        }
                    }
                    Ok(_) => return Ok(None),                  // ELOCKED
                }
            }
            Err(e) if e.kind() == io::ErrorKind::AlreadyExists => return Ok(None),
            Err(e) => return Err(e),
        }
    }
    Ok(None)
}
// Refresh thread: every `update` ms, stat; if modified() != last_mtime => set a
// `compromised` flag; else set_modified(now) and store the read-back mtime.
// Commit step checks `compromised` immediately before its final rename.
```

Round-trip codec check, parameterized over fixtures:

```rust
#[test]
fn v1_fixtures_round_trip() {
    insta::glob!("fixtures/ledger@2/*.json", |p| {
        let bytes = std::fs::read(p).unwrap();
        let model: LedgerV2 = formats::decode(&bytes).unwrap(); // deny_unknown_fields everywhere
        assert_eq!(formats::encode(&model), bytes);             // byte-exact
    });
}
```

## Gotchas

- **Never `canonicalize()` the lock target.** v1 does not follow symlinks; a canonical path gives
  a different lock dir. v1 already had this class of split-brain: locking
  `placements.json.lock` as a target created `placements.json.lock.lock` (`place/ledger.ts:420-426`).
- **Keep the lock dir empty.** The peer reclaims stale locks with `rmdir`, which fails
  `ENOTEMPTY` on a non-empty dir (verified). Metadata goes in a separate file, as v1's member
  marker does (its on-disk location was not read).
- **README vs source.** The README says `stale` minimum is 5000 ms; 4.1.2 clamps at 2000 ms.
  Trust the source; v1's central lock uses exactly 2000.
- **Staleness uses wall clocks across processes.** Clock jumps or long sleeps can make a live
  lock look stale. The holder detects this only at its next refresh, so the commit step must
  check the compromised flag.
- **v1's ledger lock uses the default `onCompromised`**, which throws inside a timer and exits
  Node. v2 should abort cleanly instead.
- **`File::lock` (Rust 1.89, flock) does not exclude v1.** It shares no identity with the mkdir
  dir. On Linux NFS, flock and fcntl locks interact; locally they do not (`flock(2)`).
- **serde strictness is per container.** Every nested struct needs `deny_unknown_fields`; it is
  unsupported with `flatten`; any `serde_json::Value` field accepts anything.
- **Byte exactness.** Match key order, integer vs float printing (serde_json prints `f64` 1 as
  `1.0`), string escaping, indentation, and the terminal newline (v1 codec descriptor
  `terminalLf`).
- **Parallel run only for reads.** Scientist "is only safe for wrapping methods that aren't
  changing data". D8 output schemas differ, so compare decoded models, not bytes.

## Currency notes

- `std::fs::File::lock`/`try_lock` stabilised in Rust 1.89; `fs4`/`fd-lock` are no longer needed
  for plain flock.
- proper-lockfile: last push 2023-10-25, 21 open issues (GitHub API, 2026-09-25). Forks exist
  (`@alcalzone/proper-lockfile`, `proper-lockfile2`). A v1 switch to a fork or new options is a
  coexistence-breaking change; pin compatibility tests to v1's shipped version.
- No Rust crate implementing the proper-lockfile convention was found on 2026-09-25.
- MCP's official conformance repo added per-SDK expected-failure baselines and a composite
  GitHub Action; LSP still has no official conformance suite (microsoft/language-server-protocol
  issue #353).

## Recommendation

**Adopt (ranked first):** ledger root sets with reachability GC; scan-then-write collision
planning; sticky manual collision overrides; foreign-file records; preimage compare-and-swap with
a shared `journal@1`; an in-house proper-lockfile-compatible mkdir lock covering all three v1
lock families with v1's exact parameters; strict serde codecs proven by byte-exact `insta::glob!`
fixtures; one conformance suite over the adapter registry, checks tagged by D1 capability
(test262 `features`) with per-adapter expected-failure baselines (MCP conformance); read-only
parallel run of v1 and v2 before any v2 writes.

**Runner-up:** OS locks (`File::lock`) for v2-only files, plus the mkdir lock only around shared
writes. Rejected during the overlap because it adds a second lock ordering and still needs the
mkdir code. **Condition to switch:** once v1 is retired, move to `File::lock` for kernel release
on crash.

**Why not:** stow-style inferred ownership (cannot express co-owners); protobuf-style
unknown-field preservation (two writers diverge silently); RFC 8785 JCS (breaks v1 byte
compatibility); dual execution of writes (unsafe per Scientist); `force-unlock` without a holder
nonce (Terraform warns it creates multiple writers).

## Errata (2026-09-26, from PR review)

- **Lock path derivation.** The sketch's `lock_dir` must match Node's `path.resolve` exactly:
  absolutize, then resolve `.` and `..` lexically, without following symlinks. On POSIX
  `std::path::absolute` keeps `..`, so the sketch as written can pick a different lock directory
  than v1 for paths containing `..` (verified: proper-lockfile 4.1.2 `lib/lockfile.js:17` calls
  `path.resolve` when `realpath` is false). Adopted in ADR 0001 D5.
- **Fencing.** The compromise check (mtime comparison on refresh) does not stop a holder that
  pauses after its last check, loses the lock to a stale reclaim, then resumes and renames over
  newer state. This is an open decision in ADR 0001; the recommendation above is incomplete
  until it is resolved.
