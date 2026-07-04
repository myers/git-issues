# Design: epic progress bar

Date: 2026-07-04
Status: approved design, pre-implementation

## Summary

Add a generated, never-hand-typed completion bar for epics, derived
from their `parent-child` tree of leaf descendants. Ported in concept
from `epic-progress.py` (an asterinas-workspace script the operator
dropped at the repo root) — but reimplemented against git-issues'
actual data model (issue records + parent-child dep edges) rather
than that script's markdown-table/README parsing, which doesn't apply
here.

Two surfaces:

- **`iss dep progress <epic>`** — new verb under the existing `dep`
  subcommand group, standalone and scriptable.
- **`iss show <epic>`** — the same computation embedded as one line,
  automatically, only when the shown issue's `type` is `epic`.

## Counting model

Walk the `parent-child` tree rooted at the epic (same reachability
rule as `iss dep tree`: X is a child of Y iff X carries a
`parent-child` edge targeting Y). A **leaf** is any reachable node
with no parent-child children of its own. Only leaves are counted;
an intermediate node (an epic-shaped child with its own children) is
never itself counted — its leaves are.

Per leaf, by status:

- `closed` → counts toward `done`.
- claimed (in-progress) → counts toward `in_progress`, and also
  toward the not-yet-done side of the ratio (not `done`).
- `open`, `blocked`, `abandoned` → not-done. `abandoned` is not
  excluded from the denominator — it still counts as an uncompleted
  leaf (simplest mapping; matches the source script's three-state
  model most closely).

A leaf carrying the label `optional` is excluded from `done`/`total`
entirely and tracked separately as `optional_done`/`optional_total`.

Cycle safety: reuse the same guard `Storage::dep_tree` already
implements (a `HashSet<IssueId>` of visited nodes during the DFS; a
re-visited node stops recursion without double-counting or infinite
looping).

Zero-leaf epics (freshly created, no children yet) render `0/0 (0%)`
— not an error, not a hidden line. Honest signal that no work is
scoped yet.

## Storage layer

New method on `Storage`, next to `dep_tree` in
`crates/iss-storage/src/lib.rs`:

```rust
pub struct EpicProgress {
    pub done: usize,
    pub total: usize,
    pub in_progress: usize,
    pub optional_total: usize,
    pub optional_done: usize,
}

pub fn epic_progress(&self, root_id: &IssueId) -> Result<EpicProgress>
```

Implementation reuses `dep_tree`'s pattern: one `snapshot()` call,
one `children_of: BTreeMap<IssueId, Vec<IssueId>>` index built from
every issue's `parent-child` edges, then a DFS from `root_id` with a
`visited: HashSet<IssueId>` cycle guard. Where `dep_tree`'s `walk`
builds a `DepTreeNode` tree, this walk instead accumulates directly
into the five `EpicProgress` counters — no tree is built, so no new
field is added to the existing public `DepTreeNode`/`DepTree` types
(keeps `iss dep tree`'s output shape untouched).

A node with children recurses into them and contributes nothing
itself. A node with no children (a leaf, including the root itself
if the epic has no parent-child children) contributes to the
counters per the status/label rules above.

## CLI: `iss dep progress <epic>`

New subcommand under `DepAction` (`crates/iss/src/main.rs`, alongside
`Add`/`Rm`/`Tree`). Takes one
positional `<epic>` (7-char hex id or slug, same handle resolution
as every other verb).

Plain-text output, fixed 20-cell width, no color, no `--width`/
`--color` flags:

```
progress: [████████░░░░░░░░░░░░] 3/8 (38%)  2 in-progress · +1 optional
```

Rendering rules:
- Bar: `round(done / total * 20)` filled cells (`█`), rest empty
  (`░`). `total == 0` renders a fully-empty bar.
- Percentage: `round(100 * done / total)`, or `0` when `total == 0`.
- Trailing extras, space-separated by ` · `, only present when
  nonzero: `"<in_progress> in-progress"` and
  `"+<optional_total - optional_done> optional"` (or
  `"<optional_total> optional done"` when every optional leaf is
  closed).

`--json` emits the bare `EpicProgress` struct, no envelope (matches
`show`/`ls`/`dep tree`'s "read verb, no ok-wrapper" convention):

```json
{"done": 3, "total": 8, "in_progress": 2, "optional_total": 1, "optional_done": 0}
```

Calling this on a non-epic-typed issue is not an error — the
parent-child walk works the same regardless of the root's own
`type`. (An epic is just an issue with `type: epic`; nothing about
the tree walk depends on that type. Restricting the standalone verb
to epics-only would be an arbitrary extra check with no storage-layer
backing.)

## CLI: embedded in `iss show <epic>`

In `print_issue_plain` (`crates/iss/src/main.rs`), after rendering
the existing header block (id/status/title/type/slug/labels/
metadata/priority/assignee/dependencies), when `issue.type_ ==
IssueType::Epic`: call `Storage::epic_progress`, render the same
one-line format as the standalone verb, insert it as its own line.
Exact insertion point: immediately after the `dependencies:` block
(or its omission, per the existing omit-when-empty behavior), before
the `created:`/`updated:` timestamp line.

`iss show --json`: add a `"progress"` key holding the same bare
`EpicProgress`-shaped object, present in the JSON output only when
`type == "epic"`. Every other issue type's JSON is byte-for-byte
unchanged (no `"progress": null` clutter on non-epics).

## Testing

- `crates/iss-storage`: unit tests for `epic_progress` mirroring
  `dep_tree`'s existing test style — leaf-only counting (a 2-level
  nested tree where only leaves count), the `optional` label
  exclusion, claimed-status counting toward `in_progress`,
  abandoned-status counting as not-done, cycle safety (a
  parent-child cycle doesn't infinite-loop or double-count), and the
  zero-leaf case.
- `crates/iss/tests`: new test file (or extend `dep.rs`) covering
  `iss dep progress <epic>` plain-text and `--json` output against a
  real fixture tree built through the CLI (`iss new --parent`, `iss
  label add optional`, `iss close`/`iss update --claim`). Extend
  `show.rs` with cases for the embedded line/field: present and
  correct for an epic with children, `0/0 (0%)` for a childless
  epic, entirely absent (no line, no JSON key) for a non-epic issue.

## Out of scope

- No `--width`/`--color` flags (fixed 20-cell width, no ANSI,
  matches this session's earlier CLI-lean-pass philosophy of not
  adding configurability nothing asked for).
- No changes to `iss dep tree`'s existing `DepTreeNode`/`DepTree`
  output shape — the progress computation is a separate, parallel
  walk, not a field bolted onto the tree types.
- No new label/metadata convention beyond the single `optional`
  label already decided.
