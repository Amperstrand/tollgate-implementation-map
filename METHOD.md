# Method, and a correction log

This repo makes claims about six codebases. This file records **how** each was measured, and
**what I got wrong first** — because the wrong version was more plausible than the right one and
someone will otherwise repeat it.

---

## Method

**Date of evidence:** 2026-10-18 (repo heads as recorded in [README.md](README.md#5-capability-matrix-measured)).

**Rules applied:**

1. **Read the code, not the self-description.** Every capability claim cites a file path in the
   matrix. A README saying "implements TIP-01" is not evidence.
2. **Count with a real walker.** Term-frequency counts were produced with Python
   `os.walk` + `re` over absolute paths — see the failure below for why `grep` was not trusted.
3. **Separate "exists" from "works".** Nothing here was built or run except where explicitly noted.
   Test counts are **static counts of test declarations**, not executed suites.
4. **Attribute forks by commit ancestry**, not by name. Two repos share the name `tollgate-rs`;
   `git merge-base` and `rev-list --left-right --count` settle which is which.
5. **State the unverified.** Each claim that rests on a document rather than a measurement is
   marked as such.

### What was NOT verified

- No implementation was built, run, or benchmarked.
- `v1_compat/` was characterised by reading its module list, its kind table
  (`v1_compat/nostr.rs:12-15`) and its `price_per_step` parser (`:62-73`) — **it was not built, and
  not run against the corpus.** Whether it actually passes the conformance corpus is **UNVERIFIED**
  and is the single most valuable next measurement.
- The corpus' green/red state is quoted from `corpus/manifest.json`, not re-run.
- Test counts are `grep`/`re` counts of declarations (`#[test]`, `func Test`), not passing suites.
- Package sizes (flash/RSS) are quoted from the repos' own measurement docs.

---

## Correction log

### 2026-10-18 — "tollgate-rs implements a different protocol with zero TIP references"

**The claim I first published:** `OpenTollGate/tollgate-rs` implements a different protocol
(vouchers + Spilman channels, CBOR) with **zero** references to TIP-01/TIP-02/HTTP-01 or event kinds
10021/1022/21023, and therefore the fleet is "two incompatible designs" with no bridge.

**What is actually true:**

- That statement is correct **for `OpenTollGate/tollgate-rs` only.** I verified it three ways and a
  second reviewer independently reproduced it.
- But it was **the wrong repo to generalise from.** There are **three** Rust repos, and the one the
  test framework calls `rust` is `Amperstrand/tollgate-rs-ai-research-and-experiments` — which
  contains a **5,594-line `v1_compat/` module** implementing exactly the TIP surface I said was absent.
- So the conclusion "no bridge exists" was **false**. A bridge exists; it is just on a fork.

**Why the error was easy to make (three compounding causes):**

| Cause | Detail |
|---|---|
| Wrong repo | PRTA's `--backend rust` → `Amperstrand/…`, not `OpenTollGate/tollgate-rs`. Two repos, same project name. |
| Missed a repo | Three Rust repos, not two. I never enumerated them; I assumed the org repo was canonical. |
| Corrupted measurements | The shell working directory drifted mid-session (`pwd` returned `.../cache`), so several `grep`/`find` calls returned **empty output** — which I read as "zero hits" rather than "command failed". |

**The lesson that generalises:** *empty output is not evidence of absence.* A `grep` that returns
nothing because it ran in the wrong directory is indistinguishable, in the transcript, from a `grep`
that genuinely found nothing. **Assert the working directory, or use absolute paths, before drawing
a conclusion from a zero.**

**How the correction was made:** re-measured with an absolute-path search tool, enumerated all repos
via the GitHub API, then proved ancestry:

```bash
git merge-base HEAD org/master                          # be5400f
git cat-file -t be5400f:crates/tollgate-net/src/v1_compat   # fatal: not in be5400f
git rev-list --left-right --count HEAD...org/master         # 94  117
```

Three independent confirmations that `v1_compat/` is post-fork.

---

## Repo-by-repo: what each claim rests on

| Claim | Evidence type |
|---|---|
| `tollgate-rs` (org) has no TIP surface | `re` count = 0 across all text files; no HTTP server dependency; own design doc `tollgate-protocol.md:203` records HTTP as *"The shape, if it is added"* |
| `v1_compat/` exists and is post-fork | file listing (16 modules, 5,594 LOC) + `git cat-file` at merge-base + commit list |
| `v1_compat/` implements the kinds | source: `v1_compat/nostr.rs:12-15` kind table, `:62-73` `price_per_step` parser |
| `basic-rust` is a V1 clone and non-conformant on 1022/21023 | source: `src/http/routes/pay.rs:180-184`, `:51-53,90,117,135`; `device-identifier` count = 0 |
| `basic-go` implements V1 fully | source: `src/main.go:426` (`:2121`), `merchant/merchant.go:1991-2009` (10021), `:2133-2139` (1022), `:2272` (21023) |
| `gonuts` is archived | GitHub API `repos/elnosh/gonuts` → `archived: true`, `pushed_at: 2025-08-16` |
| `cdk` is active | GitHub API → 300 commits since 2026-07-18, 68 releases, last `v0.18.2` 2026-10-14 |
| Corpus certifies only V1 consumers | `corpus/manifest.json` → `consumer_ids`, `kg_state.status` |
| `NOSTR-01`/`WIFI-01` are stubs | both files are 5 lines, body is "RESERVED" |

---

## Contributing

Corrections welcome, and they should arrive as **evidence**: a command plus its output, or a
`file:line`. This repo exists because a plausible wrong answer was already published once; the
correction log is the mechanism that keeps that from being invisible.
