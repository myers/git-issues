# Design: jj-cleanup sweep + CLI output lean pass

Date: 2026-07-02
Status: approved design, pre-implementation

## Summary

A live walkthrough of `docs/quickstart.md` against the real `iss`
binary (in a scratch repo) surfaced seven findings, falling into two
independent tracks:

- **Track A — stale jj references.** The 2026-06-30 jj-removal effort
  (`2026-06-30-git-issues-rename-jj-removal-design.md`) missed a few
  human-surface spots: quickstart.md still tells readers to run
  `jj git init` / `jj git clone`, two `--help` strings still describe
  jj-wrapper behavior that no longer exists, and one internal function
  name (`preflight::jj_repo`) still says jj despite having been pure
  `git rev-parse` for a while. No behavior changes — text and
  identifier renames only.

- **Track B — CLI output leanness.** Three places where `iss` output
  costs more tokens than the information warrants: `iss show` always
  prints four `(none)` lines even when nothing is set, `iss memories`
  spends 4 lines of scaffolding per entry, and the memory
  auto-slugger turns contractions into noise (`can't` → `can-t-`
  fragments instead of `cant`). Real behavior changes, need tests.

Both tracks are small and independent of each other; they're captured
in one spec because they came out of the same walkthrough and share a
"leave things better than the last sweep found them" rationale. Either
could ship alone.

## Track A — stale jj references

### A1. `docs/quickstart.md`

- **Prerequisites**: drop the `jj (Jujutsu) 0.40 or newer on PATH`
  bullet and the jj-identity-config bullet. Replace with nothing (a
  git repo + the `iss` binary is the full prerequisite list now).
- **Step 1 "Create the repo"**: replace `jj git init` with `git init`.
  Drop the "colocated jj+git repo... `.git/` and `.jj/` side-by-side"
  explanation — it's just a git repo now.
- **Step 9 "Joining an existing project"**: replace
  `jj git clone <url> <dir>` with `git clone <url> <dir>`.
- **Step 5 sample output**: the doc's `iss ready` sample block shows
  4 columns (`id status priority-as-"1L" title"`) with no `type`
  column. The real binary emits 5 tab-separated columns
  (`id  status  priority  type  title`, priority rendered as `-` when
  unset). Update both sample blocks in step 5 (before-close and
  after-close) to match real output.
- **Verified output section**: the transcript's `$ jj git init` line
  becomes `$ git init`; re-verify the rest of the transcript still
  matches (it should — steps 2-8 don't reference jj).

### A2. `--help` text

- `iss remote --help` (crate-level doc comment on the `Remote`
  subcommand in `crates/iss/src/main.rs`, plus the `add`/`ls`/`rm`
  subcommand docs): strip "Thin wrapper over `jj git remote
  add|list|remove`", "underlying jj repo", "jj already supports git
  transport for bookmarks", "jj-repo-only" language. Describe actual
  behavior — `add`/`rm` shell out to plain `git remote add|remove`,
  `ls` shells out to `git remote -v`.
- `iss init --help`: "Initialize the `issues` bookmark on the current
  jj repo" → "...on the current git repo" (or reword to avoid
  "bookmark", which is jj terminology — "Initialize git-issues in the
  current git repo" reads cleaner).
- Use `iss pull --help`'s existing text as the template — it already
  correctly describes plain-git behavior with no jj mentions.
- Leave `refs/jjf/*` and `Jjf-*` trailer mentions untouched anywhere
  they appear — those are the intentionally-vestigial wire-format
  tokens per the 2026-06-30 design's "Human surface only" rule, not
  stale references.

### A3. `preflight::jj_repo` rename

- `crates/iss/src/preflight.rs`: rename fn `jj_repo` → `git_repo`.
  Implementation body unchanged (already pure
  `git rev-parse --git-dir`).
- Rename `StorageError::NotAJjRepo` variant → `NotAGitRepo` (check
  `crates/iss-storage/src/lib.rs` or wherever `Error` is defined).
  Update its Display message if it mentions "jj repo".
- Update all call sites (`issues_bookmark`, `run_remote_ls`, and
  anywhere else `jj_repo(...)` or `NotAJjRepo` is referenced) —
  mechanical rename, `cargo build` will surface every site.
- Doc comments in `preflight.rs` referencing "jj repo" get the same
  s/jj/git/ treatment as A2.

## Track B — CLI output leanness

### B1. `iss show`: omit empty fields

In `print_issue_plain` (`crates/iss/src/main.rs`), `labels:`,
`priority:`, `assignee:`, and `dependencies:` currently always print,
falling back to `(none)` when unset. Change to: **omit the line
entirely when the field is empty**, matching the pattern already used
for `metadata` two lines above in the same function (`if
!issue.metadata.is_empty() { ... }`).

- `labels:` — omit when `issue.labels.is_empty()`.
- `priority:` — omit when `issue.priority.is_none()`.
- `assignee:` — omit when `issue.assignee.is_none()`.
- `dependencies:` — omit when `issue.dependencies.is_empty()` (the
  existing `dependencies:` / per-kind-line block only fires for
  non-empty deps already; just drop the `else` branch's
  `println!("dependencies: (none)")`).

`slug:` keeps its `(none)` fallback — it's a single always-relevant
identity field, not a collection/optional-attribute field, and isn't
part of this finding.

Scope: **plain-text mode only** (`print_issue_plain`'s existing doc
comment already says "not a contract" — `--json` output is a separate
path and is unaffected).

A fully-empty issue (no labels, no priority, no assignee, no deps, no
metadata) now renders as just: id/status, title, type, slug, created/
updated timestamps, body, comments — no empty-field noise.

### B2. `iss memories`: one line per entry

Current shape (`crates/iss/src/main.rs`, the memories-list branch):

```
memories (N):

<key>
  <truncated value>

<key>
  <truncated value>
```

New shape — one line per memory, tab-separated, no header, no blank
lines, matching `iss remote ls`'s existing convention:

```
<key>	<truncated value>
<key>	<truncated value>
```

- Drop the `memories ({}):` header line and the `println!()` blank
  lines (before the loop and after each entry).
- Same change applies to the `--search` variant's matched-list
  rendering (`memories matching {s:?}:` header + blank lines get the
  same treatment — becomes `key\tvalue` per line, no header).
- The two empty-result messages ("no memories stored...", "no
  memories matching...") are already single-line — unchanged.
- `truncate_memory(&m.value, 120)` truncation logic unchanged.
- `--json` output unaffected (separate path).

### B3. `slugify`: strip quote/apostrophe chars instead of splitting on them

`crates/iss-storage/src/memory.rs`, `slugify()`. Currently every
non-alphanumeric char (including `'`) starts a new hyphen-run, so
`"can't"` → tokens `can`, `t` → slug fragment `can-t`. A real example
hit during the quickstart walkthrough: `"Backend's /submit handler
can't take empty bodies."` auto-slugged to
`backend-s-submit-handler-can-t-take-empty` — two spurious splits
from apostrophes alone.

Fix: before the existing lowercase + non-alphanumeric-collapse pass,
strip (delete, not replace-with-hyphen) these characters entirely:

- ASCII apostrophe `'` (U+0027)
- Right single quotation mark `'` (U+2019, common smart-quote form)
- ASCII double quote `"` (U+0022)
- Left double quotation mark `"` (U+201C)
- Right double quotation mark `"` (U+201D)

So `"Backend's /submit handler can't take..."` → after stripping →
`"Backends /submit handler cant take..."` → slugifies to
`backends-submit-handler-cant-take-empty` (first 8 tokens, ≤60
chars — same downstream rules, just no `-s-`/`-t-` fragments).

Everything else in the algorithm (lowercase, other non-alphanumeric
runs → single `-`, leading/trailing trim, 8-token cap, 60-char cap)
is unchanged. Update the doc comment above `slugify` (lines 52-66) to
mention the strip-not-split rule for quote characters.

## Testing

Both tracks get unit tests; Track B's are new, Track A is a rename
with no new test surface (existing tests should still pass unchanged
except for any that assert on the old `NotAJjRepo` variant name or
`--help` text, which get updated to match).

- **B3 (`slugify`)**: extend `memory.rs`'s existing slugify test
  table with cases: `"can't"` → `"cant"`, `"it's a test"` → roughly
  `"its-a-test"`, a curly-quote case (`"don\u{2019}t"` → `"dont"`),
  and a double-quote case (`"a \"quoted\" phrase"` →
  `"a-quoted-phrase"` — no split around the quote marks). Keep
  existing empty-input and no-alphanumeric-input cases passing.
- **B1 (`print_issue_plain`)**: add/extend a test asserting a
  minimal issue (no labels/priority/assignee/deps/metadata) omits
  all four lines, and a populated issue still prints all of them —
  likely via whatever existing plain-text-output test harness
  `main.rs` already has for `show` (grep for existing `show` output
  tests to match the harness style).
- **B2 (memories list)**: add/extend a test asserting N memories
  render as exactly N lines, tab-separated, no header/blank lines;
  and the `--search` variant likewise.

## Sequencing

Track A first (pure rename/text, zero risk, unblocks nothing but
also blocks nothing), then Track B (behavior change, wants the
tests). Both can land as one PR or two commits in the existing
"open PR, address feedback, merge" flow this repo uses — no epic/
ticket scaffolding needed, this is sweep-class cleanup work, not a
new roadmap item.

## Out of scope

- `refs/jjf/*` ref namespace, `Jjf-*` commit trailers, `jjf-storage`/
  `iss-storage` crate internals that reference the wire format — all
  intentionally vestigial per the 2026-06-30 rename design.
- Any other `--help` strings not already identified as jj-stale (a
  full grep sweep of `main.rs` for "jj" is a reasonable follow-up but
  wasn't exhaustively done here — this spec covers the two verbs
  found during the walkthrough: `remote` and `init`).
- Further CLI-output leanness beyond the three Track B items (e.g.
  `iss ls` row format, `iss dep tree` format) — not examined during
  this walkthrough, out of scope unless a future pass finds them.
