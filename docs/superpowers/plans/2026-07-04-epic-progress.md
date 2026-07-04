# Epic Progress Bar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a generated completion bar for epics — derived from their `parent-child` leaf descendants' statuses — surfaced both as a standalone `iss dep progress <epic>` verb and embedded automatically in `iss show <epic>`.

**Architecture:** A new `Storage::epic_progress` method in `crates/iss-storage` reuses `Storage::dep_tree`'s exact traversal pattern (one `snapshot()` call, a `children_of` parent-child index, a cycle-safe DFS) but accumulates five counters instead of building a tree. The CLI layer adds one new `DepAction::Progress` subcommand and threads the same computation into `print_issue_plain` and `run_show`'s JSON path.

**Tech Stack:** Rust (`crates/iss-storage` lib, `crates/iss` binary), existing hermetic-scratch integration test harness (`crates/iss/tests/common/mod.rs`).

## Global Constraints

- Counting model: walk `parent-child` edges (same direction/reachability as `iss dep tree`). Only **leaf** nodes (no parent-child children of their own) count toward the ratio; intermediate nodes contribute nothing directly.
- Status mapping: `closed` → `done`; in-progress (claimed) → counts toward `in_progress` AND toward not-done; `open`/`blocked`/`abandoned` → not-done (abandoned is NOT excluded from the denominator).
- A leaf carrying the label `optional` is excluded from `done`/`total`, tracked separately as `optional_done`/`optional_total`.
- Cycle safety: reuse `dep_tree`'s `visited: HashSet<IssueId>` guard — a re-visited node stops recursion, no double-count, no infinite loop.
- Zero-leaf epics render `0/0 (0%)` — not an error, not a hidden line.
- Plain-text bar: fixed 20-cell width, no color, no `--width`/`--color` flags anywhere.
- `iss dep progress <epic>` `--json`: bare object `{"done", "total", "in_progress", "optional_total", "optional_done"}`, no envelope.
- `iss show --json` on a non-epic issue must be byte-for-byte unchanged (no `"progress": null` clutter).
- No changes to `iss dep tree`'s existing `DepTreeNode`/`DepTree` output shape — this is a separate, parallel computation.

---

## File Structure

- `crates/iss-storage/src/lib.rs` — new `EpicProgress` struct + `Storage::epic_progress` method (Task 1). New `progress: Option<EpicProgress>` field on the existing `Issue` struct (Task 3), which requires updating every `Issue { ... }` struct-literal construction site in this file (the `mk_issue` test helper at ~line 4541, plus test fixtures in `cache.rs`).
- `crates/iss-storage/src/read.rs` — one `Issue { ... }` construction site needs the new field (Task 3).
- `crates/iss-storage/src/cache.rs` — five `Issue { ... }` construction sites in tests need the new field (Task 3).
- `crates/iss/src/main.rs` — new `DepAction::Progress` variant + `run_dep_progress` function + rendering helper (Task 2); `run_show` populates `issue.progress` before JSON serialization and passes it to `print_issue_plain` for the plain-text embed (Task 4).
- `crates/iss/tests/dep.rs` — new tests for `iss dep progress` (Task 2).
- `crates/iss/tests/show.rs` — new tests for the embedded progress line/field (Task 4).

## Task Right-Sizing

Three tasks: (1) the storage-layer computation, fully unit-testable in isolation; (2) the standalone CLI verb, which depends on Task 1's method and is independently testable via the compiled binary; (3) combined — adding the `progress` field to `Issue` and wiring it into both `run_show`'s JSON path and `print_issue_plain`'s text path, since these two surfaces share the single struct-literal change and are easiest to review together.

---

### Task 1: `Storage::epic_progress` — the counting engine

**Files:**
- Modify: `crates/iss-storage/src/lib.rs` (add `EpicProgress` struct near `DepTreeNode`/`DepTree`, around line 1387; add `epic_progress` method near `dep_tree`, around line 2874; add unit tests in the existing `#[cfg(test)] mod tests` block)

**Interfaces:**
- Produces: `pub struct EpicProgress { pub done: usize, pub total: usize, pub in_progress: usize, pub optional_total: usize, pub optional_done: usize }` (derives `Debug, Clone, Copy, PartialEq, Eq, serde::Serialize` — no `Deserialize`, this is never read from disk). A free function `fn epic_progress_from_issues(root_id: &IssueId, all: &[Issue]) -> EpicProgress` (private to the crate, unit-tested directly against a `Vec<Issue>` — same pattern as the existing `compute_blocked_set(all: &[Issue])` at `crates/iss-storage/src/lib.rs:674`, tested via plain `Vec<Issue>` literals with no `Storage`/tempdir involved). `pub fn epic_progress(&self, root_id: &IssueId) -> Result<EpicProgress>` on `Storage`, a thin wrapper that calls `self.snapshot()` then delegates to the free function.
- Consumes: nothing from other tasks — this task is self-contained, built entirely on existing `Storage`/`Issue`/`Status`/`IssueType` types.

- [ ] **Step 1: Write the failing tests**

In `crates/iss-storage/src/lib.rs`, find the `#[cfg(test)] mod tests` block (search for `fn mk_issue` to locate it — the tests live in the same module, alongside `compute_blocked_set`'s tests). Add these test functions near the existing `compute_blocked_set` tests (the ones starting around line 4565, e.g. `blocks_edge_to_open_target_blocks_owner`):

```rust
    fn mk_labeled_issue(
        id: &str,
        status: Status,
        deps: Vec<DepEdge>,
        labels: Vec<&str>,
    ) -> Issue {
        let mut issue = mk_issue(id, status, deps);
        issue.labels = labels.into_iter().map(String::from).collect();
        issue
    }

    fn parent_child_edge(target: &str) -> DepEdge {
        DepEdge {
            kind: DepKind::ParentChild,
            target: iid(target),
        }
    }

    #[test]
    fn epic_progress_counts_only_leaves() {
        // epic -> mid -> leaf1, leaf2. epic and mid must NOT be
        // counted themselves; only leaf1/leaf2 count.
        let epic = mk_issue("0000001", Status::Open, vec![]);
        let mid = mk_issue("0000002", Status::Open, vec![parent_child_edge("0000001")]);
        let leaf1 = mk_issue("0000003", Status::Closed, vec![parent_child_edge("0000002")]);
        let leaf2 = mk_issue("0000004", Status::Open, vec![parent_child_edge("0000002")]);
        let all = vec![epic, mid, leaf1, leaf2];

        let progress = epic_progress_from_issues(&iid("0000001"), &all);
        assert_eq!(progress.total, 2, "only leaf1/leaf2 count, not mid or epic itself");
        assert_eq!(progress.done, 1, "leaf1 is closed");
    }

    #[test]
    fn epic_progress_excludes_optional_labeled_leaves() {
        let epic = mk_issue("0000001", Status::Open, vec![]);
        let required = mk_issue("0000002", Status::Closed, vec![parent_child_edge("0000001")]);
        let optional = mk_labeled_issue(
            "0000003",
            Status::Open,
            vec![parent_child_edge("0000001")],
            vec!["optional"],
        );
        let all = vec![epic, required, optional];

        let progress = epic_progress_from_issues(&iid("0000001"), &all);
        assert_eq!(progress.total, 1, "optional leaf excluded from total");
        assert_eq!(progress.done, 1);
        assert_eq!(progress.optional_total, 1);
        assert_eq!(progress.optional_done, 0, "optional leaf is Open, not done");
    }

    #[test]
    fn epic_progress_counts_in_progress_leaves() {
        let epic = mk_issue("0000001", Status::Open, vec![]);
        let claimed = mk_issue(
            "0000002",
            Status::InProgress,
            vec![parent_child_edge("0000001")],
        );
        let all = vec![epic, claimed];

        let progress = epic_progress_from_issues(&iid("0000001"), &all);
        assert_eq!(progress.total, 1);
        assert_eq!(progress.done, 0, "in-progress is not done");
        assert_eq!(progress.in_progress, 1);
    }

    #[test]
    fn epic_progress_counts_abandoned_as_not_done() {
        let epic = mk_issue("0000001", Status::Open, vec![]);
        let abandoned = mk_issue(
            "0000002",
            Status::Abandoned,
            vec![parent_child_edge("0000001")],
        );
        let all = vec![epic, abandoned];

        let progress = epic_progress_from_issues(&iid("0000001"), &all);
        assert_eq!(progress.total, 1, "abandoned still counts toward the denominator");
        assert_eq!(progress.done, 0);
    }

    #[test]
    fn epic_progress_handles_cycles_without_double_counting() {
        // a -> b -> a (parent-child cycle). Root is "0000001".
        let a = mk_issue("0000001", Status::Open, vec![parent_child_edge("0000002")]);
        let b = mk_issue("0000002", Status::Closed, vec![parent_child_edge("0000001")]);
        let all = vec![a, b];

        let progress = epic_progress_from_issues(&iid("0000001"), &all);
        // Must terminate (no infinite loop) and not double-count.
        assert!(progress.total <= 2);
    }

    #[test]
    fn epic_progress_zero_leaves_is_zero_over_zero() {
        let epic = mk_issue("0000001", Status::Open, vec![]);
        let all = vec![epic];

        let progress = epic_progress_from_issues(&iid("0000001"), &all);
        assert_eq!(progress.total, 0);
        assert_eq!(progress.done, 0);
    }
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cargo test -p iss-storage epic_progress 2>&1 | tail -40`
Expected: FAIL — `epic_progress_from_issues` doesn't exist yet (compile error: function not found).

- [ ] **Step 3: Implement `EpicProgress`, `epic_progress_from_issues`, and `Storage::epic_progress`**

In `crates/iss-storage/src/lib.rs`, near the `DepTreeNode`/`DepTree` struct definitions (around line 1387), add:

```rust
/// Completion summary for an epic's `parent-child` leaf descendants
/// (spec: epic-progress design, 2026-07-04). Only LEAVES (nodes with
/// no parent-child children of their own) count; an intermediate
/// node contributes nothing directly — its own leaves do. A leaf
/// carrying the label `optional` is excluded from `done`/`total` and
/// tracked separately in `optional_total`/`optional_done`.
#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize)]
pub struct EpicProgress {
    pub done: usize,
    pub total: usize,
    /// Leaves in `Status::InProgress` — a subset of "not yet done",
    /// broken out so callers can render "N in-progress" separately.
    pub in_progress: usize,
    pub optional_total: usize,
    pub optional_done: usize,
}
```

Then, as a free function near `compute_blocked_set` (around line 674 — same "pure function over a slice of `Issue`" shape, easiest to unit test):

```rust
/// Compute the leaf-only completion summary for the `parent-child`
/// tree rooted at `root_id`, given every issue in the snapshot. See
/// [`EpicProgress`] for the counting rules. The ROOT itself is never
/// counted as a leaf — only issues reachable via at least one
/// parent-child edge FROM this function's perspective (i.e. `root`'s
/// descendants) are eligible; if `root` has zero parent-child
/// children, the result is `0/0` (an epic with no children yet has
/// no scoped work, not one unit of "root as its own leaf").
fn epic_progress_from_issues(root_id: &IssueId, all: &[Issue]) -> EpicProgress {
    let mut children_of: std::collections::BTreeMap<IssueId, Vec<IssueId>> =
        std::collections::BTreeMap::new();
    for issue in all {
        for edge in &issue.dependencies {
            if edge.kind == DepKind::ParentChild {
                children_of
                    .entry(edge.target.clone())
                    .or_default()
                    .push(issue.id.clone());
            }
        }
    }

    let issue_by_id: std::collections::HashMap<IssueId, &Issue> =
        all.iter().map(|i| (i.id.clone(), i)).collect();

    let mut progress = EpicProgress {
        done: 0,
        total: 0,
        in_progress: 0,
        optional_total: 0,
        optional_done: 0,
    };

    fn count_leaf(issue: &Issue, progress: &mut EpicProgress) {
        let is_optional = issue.labels.iter().any(|l| l == "optional");
        let is_done = issue.status == Status::Closed;
        let is_in_progress = issue.status == Status::InProgress;
        if is_optional {
            progress.optional_total += 1;
            if is_done {
                progress.optional_done += 1;
            }
        } else {
            progress.total += 1;
            if is_done {
                progress.done += 1;
            }
            if is_in_progress {
                progress.in_progress += 1;
            }
        }
    }

    fn walk(
        node: &IssueId,
        issue_by_id: &std::collections::HashMap<IssueId, &Issue>,
        children_of: &std::collections::BTreeMap<IssueId, Vec<IssueId>>,
        visited: &mut std::collections::HashSet<IssueId>,
        progress: &mut EpicProgress,
    ) {
        if !visited.insert(node.clone()) {
            return; // Cycle — already counted (or being counted) upstream.
        }
        match children_of.get(node) {
            Some(child_ids) if !child_ids.is_empty() => {
                for cid in child_ids {
                    walk(cid, issue_by_id, children_of, visited, progress);
                }
            }
            _ => {
                if let Some(issue) = issue_by_id.get(node) {
                    count_leaf(issue, progress);
                }
                // Dangling id (no matching issue record) — silently
                // skip rather than panic; shouldn't happen in
                // practice since `children_of` is built from real
                // issues' own dependency edges.
            }
        }
    }

    let mut visited = std::collections::HashSet::new();
    // Seed `visited` with the root BEFORE walking its children, so
    // the root itself is never counted as a leaf even if it has zero
    // children (the zero-leaf case: children_of.get(root_id) is
    // None/empty, but we must NOT fall into the `_ => count_leaf(...)`
    // arm for the root — we only want to count DESCENDANTS).
    visited.insert(root_id.clone());
    if let Some(child_ids) = children_of.get(root_id) {
        for cid in child_ids {
            walk(cid, &issue_by_id, &children_of, &mut visited, &mut progress);
        }
    }
    progress
}
```

Then, near `dep_tree` (around line 2874, right after its closing brace), add the thin `Storage` wrapper:

```rust
    /// Completion summary for the `parent-child` tree rooted at
    /// `root_id`. See [`EpicProgress`] for the counting rules.
    /// Delegates to [`epic_progress_from_issues`] after loading the
    /// snapshot — same "one `snapshot()` call" shape as `dep_tree`.
    pub fn epic_progress(&self, root_id: &IssueId) -> Result<EpicProgress> {
        let snapshot = self.snapshot()?;
        let all: Vec<Issue> = snapshot.issues.values().cloned().collect();
        Ok(epic_progress_from_issues(root_id, &all))
    }
```

This shape means Step 1's tests call `epic_progress_from_issues(&iid("0000001"), &all)` directly — no `Storage`/tempdir needed, matching `compute_blocked_set`'s existing test pattern exactly. The zero-leaf test (`epic_progress_zero_leaves_is_zero_over_zero`) passes because the root is seeded into `visited` up front and only its children (of which there are none) get walked — the root is never itself passed to `count_leaf`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `cargo test -p iss-storage epic_progress 2>&1 | tail -60`
Expected: all 6 tests PASS.

- [ ] **Step 5: Run the full `iss-storage` test suite**

Run: `cargo test -p iss-storage 2>&1 | tail -60`
Expected: all pass, no regressions.

- [ ] **Step 6: Commit**

```bash
git add crates/iss-storage/src/lib.rs
git commit -m "feat(storage): add EpicProgress + Storage::epic_progress"
```

---

### Task 2: `iss dep progress <epic>` — standalone verb

**Files:**
- Modify: `crates/iss/src/main.rs` (add `DepAction::Progress` variant near line 1136; add dispatch arm near line 2050; add `run_dep_progress` + a `render_epic_progress_line` helper near `run_dep_tree` at line 3152)
- Test: `crates/iss/tests/dep.rs`

**Interfaces:**
- Consumes: `Storage::epic_progress(&self, root_id: &IssueId) -> Result<EpicProgress>` (Task 1). `EpicProgress { done, total, in_progress, optional_total, optional_done }` (all `usize`, Task 1).
- Produces: `fn render_epic_progress_line(progress: &EpicProgress) -> String` — returns the one-line plain-text rendering (e.g. `"progress: [████░░░░░░░░░░░░░░░░] 3/8 (38%)  2 in-progress · +1 optional"`). This is reused by Task 4 for the embedded `iss show` line, so the exact function name and signature matter — Task 4's implementer reads this task's code to call it.

- [ ] **Step 1: Write the failing tests**

In `crates/iss/tests/dep.rs`, add (check the top of the file for existing helper imports — this file already has `make_initialized_repo`/`run_iss`-style helpers per the existing test convention; use whatever this file already uses for creating issues via the CLI, matching its existing style rather than reinventing):

```rust
#[test]
fn dep_progress_counts_leaves_and_renders_bar() {
    let repo = make_initialized_repo("dep_progress_basic");
    let epic_id = create_issue(&repo, "Epic: test", b"", &["--type", "epic"]);
    let child1 = create_issue(&repo, "child one", b"", &[]);
    let child2 = create_issue(&repo, "child two", b"", &[]);

    run_jjf(&repo, &["dep", "add", &child1, &epic_id, "--kind", "parent-child"]);
    run_jjf(&repo, &["dep", "add", &child2, &epic_id, "--kind", "parent-child"]);
    run_jjf(&repo, &["close", &child1]);

    let out = run_jjf(&repo, &["dep", "progress", &epic_id]);
    assert!(out.status.success(), "{}", String::from_utf8_lossy(&out.stderr));
    let stdout = String::from_utf8_lossy(&out.stdout);
    assert!(stdout.contains("1/2"), "expected 1/2 in output, got: {stdout}");
    assert!(stdout.contains("50%"), "expected 50%, got: {stdout}");
}

#[test]
fn dep_progress_json_shape() {
    let repo = make_initialized_repo("dep_progress_json");
    let epic_id = create_issue(&repo, "Epic: test", b"", &["--type", "epic"]);
    let child1 = create_issue(&repo, "child one", b"", &[]);
    run_jjf(&repo, &["dep", "add", &child1, &epic_id, "--kind", "parent-child"]);
    run_jjf(&repo, &["close", &child1]);

    let out = run_jjf(&repo, &["dep", "progress", "--json", &epic_id]);
    assert!(out.status.success());
    let v: serde_json::Value =
        serde_json::from_str(&String::from_utf8_lossy(&out.stdout)).expect("valid JSON");
    assert_eq!(v["done"], 1);
    assert_eq!(v["total"], 1);
    assert_eq!(v["in_progress"], 0);
    assert_eq!(v["optional_total"], 0);
    assert_eq!(v["optional_done"], 0);
}

#[test]
fn dep_progress_zero_children_is_zero_over_zero() {
    let repo = make_initialized_repo("dep_progress_empty");
    let epic_id = create_issue(&repo, "Epic: empty", b"", &["--type", "epic"]);

    let out = run_jjf(&repo, &["dep", "progress", &epic_id]);
    assert!(out.status.success());
    let stdout = String::from_utf8_lossy(&out.stdout);
    assert!(stdout.contains("0/0"), "expected 0/0, got: {stdout}");
}
```

If this test file's actual helper names differ from `make_initialized_repo`/`create_issue`/`run_jjf` (check — an earlier session-wide rename pass may have renamed these to `make_git_repo`/`run_iss`; grep `crates/iss/tests/dep.rs` and `crates/iss/tests/common/mod.rs` for the CURRENT helper names before writing this step for real, and use whatever's actually there), substitute the real names. The test logic (create epic + 2 children, wire parent-child, close one, assert on the rendered ratio) is what matters.

- [ ] **Step 2: Run tests to verify they fail**

Run: `cargo test -p iss --test dep dep_progress 2>&1 | tail -40`
Expected: FAIL — `dep progress` isn't a recognized subcommand yet (clap parse error / exit code mismatch).

- [ ] **Step 3: Add the `DepAction::Progress` variant**

In `crates/iss/src/main.rs`, find the `DepAction` enum (around line 1100) and add a new variant after `Tree`:

```rust
    /// Print a one-line completion bar for the `parent-child` tree
    /// rooted at `<id>`. Only LEAF descendants count (an
    /// intermediate node with its own children contributes nothing
    /// directly). A leaf labeled `optional` is excluded from the
    /// ratio and reported separately. `--json` emits the bare
    /// `EpicProgress` object, no envelope.
    Progress {
        /// Root issue (id-or-slug).
        id: String,
    },
```

- [ ] **Step 4: Wire the dispatch arm**

Find the match on `DepAction` (search for `DepAction::Tree { id } => run_dep_tree`, around line 2050) and add:

```rust
            DepAction::Progress { id } => run_dep_progress(cli.json, id),
```

- [ ] **Step 5: Implement `run_dep_progress` and `render_epic_progress_line`**

In `crates/iss/src/main.rs`, right after `run_dep_tree` (which ends around line 3168, before `render_dep_tree_text`), add:

```rust
fn run_dep_progress(json: bool, id: String) -> Result<(), CliError> {
    let cwd: PathBuf = std::env::current_dir().map_err(CliError::Cwd)?;
    let cwd = std::fs::canonicalize(&cwd).map_err(CliError::Cwd)?;
    preflight::require_initialized(&cwd)?;

    let storage = Storage::open(&cwd)?;
    let root_id = resolve_handle(&storage, &id)?;
    let progress = storage.epic_progress(&root_id)?;
    if json {
        let payload = serde_json::to_string(&progress)
            .expect("EpicProgress serializes — derive contract");
        println!("{payload}");
    } else {
        println!("{}", render_epic_progress_line(&progress));
    }
    Ok(())
}

/// Render one line: `progress: [<bar>] <done>/<total> (<pct>%)  <extras>`.
/// Fixed 20-cell bar width, no color. Used by both `iss dep progress`
/// and the embedded line in `iss show <epic>`.
fn render_epic_progress_line(progress: &EpicProgress) -> String {
    const WIDTH: usize = 20;
    let filled = if progress.total > 0 {
        ((progress.done as f64 / progress.total as f64) * WIDTH as f64).round() as usize
    } else {
        0
    };
    let bar: String = "█".repeat(filled) + &"░".repeat(WIDTH - filled);
    let pct = if progress.total > 0 {
        ((progress.done as f64 / progress.total as f64) * 100.0).round() as usize
    } else {
        0
    };
    let mut line = format!(
        "progress: [{}] {}/{} ({}%)",
        bar, progress.done, progress.total, pct
    );
    let mut extras: Vec<String> = Vec::new();
    if progress.in_progress > 0 {
        extras.push(format!("{} in-progress", progress.in_progress));
    }
    if progress.optional_total > 0 {
        let pending = progress.optional_total - progress.optional_done;
        if pending > 0 {
            extras.push(format!("+{pending} optional"));
        } else {
            extras.push(format!("{} optional done", progress.optional_total));
        }
    }
    if !extras.is_empty() {
        line.push_str("  ");
        line.push_str(&extras.join(" · "));
    }
    line
}
```

You'll need `EpicProgress` in scope — check the `use` statements at the top of `main.rs` for how `DepTreeNode`/`DepTree` are imported (search for `use iss_storage::` near the top of the file) and add `EpicProgress` to that same import list.

- [ ] **Step 6: Run tests to verify they pass**

Run: `cargo test -p iss --test dep dep_progress 2>&1 | tail -60`
Expected: all 3 new tests PASS.

- [ ] **Step 7: Build and run the full `iss` test suite**

Run: `cargo build --workspace 2>&1 | tail -30` — expect clean.
Run: `cargo nextest run --workspace 2>&1 | tail -60` (fall back to `cargo test --workspace`) — expect all pass, same count as before plus these 3 new tests. If you hit `spawn`-style "No such file or directory" errors unrelated to your change, this workspace has a known stale-incremental-build-artifact issue — run `cargo clean -p iss && cargo clean -p iss-storage && cargo build --workspace --tests` first. Also known: `iss-storage::v3_write_path::v3_write_path_source_does_not_invoke_jj` is a confirmed pre-existing flaky test unrelated to any change — rerun in isolation if it fails.

- [ ] **Step 8: Commit**

```bash
git add crates/iss/src/main.rs crates/iss/tests/dep.rs
git commit -m "feat(cli): add iss dep progress <epic> verb"
```

---

### Task 3 + 4: `Issue.progress` field + embed in `iss show`

**Files:**
- Modify: `crates/iss-storage/src/record.rs` (add `progress` field to `Issue` struct, around line 559-593)
- Modify: `crates/iss-storage/src/lib.rs` (update `mk_issue` test helper around line 4541; this file's `epic_progress` doesn't touch `Issue` construction, only reads fields, so no change needed there beyond the test helper)
- Modify: `crates/iss-storage/src/read.rs` (update the `Issue { ... }` construction around line 95)
- Modify: `crates/iss-storage/src/cache.rs` (update 5 `Issue { ... }` construction sites — grep `Issue {` in this file to find exact lines, they shift as the file changes; there were 5 as of this plan's writing, at approximately lines 561, 680, 697, 724, 741)
- Modify: `crates/iss/src/main.rs` (`run_show` populates `issue.progress` before serializing/rendering; `print_issue_plain` gains a parameter and renders the embedded line)
- Test: `crates/iss/tests/show.rs`

**Interfaces:**
- Consumes: `Storage::epic_progress` (Task 1), `render_epic_progress_line` (Task 2) — both called from `run_show`.
- Produces: `Issue.progress: Option<EpicProgress>` (new field, `#[serde(default, skip_serializing_if = "Option::is_none")]`). `print_issue_plain`'s signature changes — later code reading this function must call it the new way (there is only one call site, in `run_show`, updated in this same task).

- [ ] **Step 1: Add the `progress` field to `Issue`**

In `crates/iss-storage/src/record.rs`, find the `Issue` struct (around line 559-593) and add the field at the end, after `updated_at`:

```rust
    pub created_at: String,
    pub updated_at: String,
    /// Completion summary for this issue's `parent-child` leaf
    /// descendants — populated ONLY when `type_ == IssueType::Epic`,
    /// and only by `run_show` at display time (never persisted;
    /// `#[serde(default)]` means old on-disk records and every other
    /// code path that builds an `Issue` without this field still
    /// deserialize/construct fine). `None` for every non-epic issue,
    /// and `None` by default for epics until `run_show` fills it in.
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub progress: Option<EpicProgress>,
```

You'll need to import `EpicProgress` in this file — check the top of `record.rs` for its existing `use` statements and add `EpicProgress` to whatever brings in `DepEdge`/`DepKind` etc. (likely a `crate::` self-import if `EpicProgress` lives in `lib.rs`, or move `EpicProgress`'s definition — check whether `record.rs` and `lib.rs` cross-reference each other's types already, and follow that existing pattern).

- [ ] **Step 2: Fix every `Issue { ... }` construction site (compile-error-driven)**

Run: `cargo build --workspace 2>&1 | tail -80`

Expected: multiple "missing field `progress`" errors. For EACH one reported, add `progress: None,` to that struct literal. Based on this plan's research, expect errors at:
- `crates/iss-storage/src/lib.rs` — the `mk_issue` test helper (~line 4541-4558): add `progress: None,` after `updated_at: "...".into(),`.
- `crates/iss-storage/src/read.rs` — the real construction site (~line 95): add `progress: None,`.
- `crates/iss-storage/src/cache.rs` — 5 sites (~lines 561, 680, 697, 724, 741 as of this plan's writing, but re-find them via the compiler errors since line numbers drift): add `progress: None,` to each.

Keep re-running `cargo build --workspace` and fixing each reported site until it compiles clean. The compiler is your checklist here — don't rely on this plan's line-number estimates, trust the error output.

- [ ] **Step 3: Run the full `iss-storage` test suite to confirm no field-addition regressions**

Run: `cargo test -p iss-storage 2>&1 | tail -60`
Expected: all pass (the new field defaults to `None` everywhere existing tests don't set it, so no existing assertion should be affected).

- [ ] **Step 4: Write the failing tests for `iss show`**

In `crates/iss/tests/show.rs`, add (following this file's existing style — check `show_plain_text_includes_every_scalar_field` near the top for the exact helper names/patterns currently in use, since an earlier session renamed some of these):

```rust
#[test]
fn show_plain_embeds_progress_line_for_epic() {
    let repo = make_initialized_repo("show_progress_epic");
    let epic_id = create_issue(&repo, "Epic: test", b"", &["--type", "epic"]);
    let child_id = create_issue(&repo, "child", b"", &[]);
    run_jjf(&repo, &["dep", "add", &child_id, &epic_id, "--kind", "parent-child"]);
    run_jjf(&repo, &["close", &child_id]);

    let out = run_jjf(&repo, &["show", &epic_id]);
    assert!(out.status.success(), "{}", String::from_utf8_lossy(&out.stderr));
    let stdout = String::from_utf8_lossy(&out.stdout);
    assert!(
        stdout.contains("progress:"),
        "epic show should include a progress line, got: {stdout}"
    );
    assert!(stdout.contains("1/1"), "got: {stdout}");
}

#[test]
fn show_plain_omits_progress_line_for_non_epic() {
    let repo = make_initialized_repo("show_progress_non_epic");
    let bug_id = create_issue(&repo, "a bug", b"", &["--type", "bug"]);

    let out = run_jjf(&repo, &["show", &bug_id]);
    assert!(out.status.success());
    let stdout = String::from_utf8_lossy(&out.stdout);
    assert!(
        !stdout.contains("progress:"),
        "non-epic show should NOT include a progress line, got: {stdout}"
    );
}

#[test]
fn show_json_includes_progress_for_epic() {
    let repo = make_initialized_repo("show_progress_json_epic");
    let epic_id = create_issue(&repo, "Epic: test", b"", &["--type", "epic"]);
    let child_id = create_issue(&repo, "child", b"", &[]);
    run_jjf(&repo, &["dep", "add", &child_id, &epic_id, "--kind", "parent-child"]);

    let out = run_jjf(&repo, &["show", "--json", &epic_id]);
    assert!(out.status.success());
    let v: serde_json::Value =
        serde_json::from_str(&String::from_utf8_lossy(&out.stdout)).expect("valid JSON");
    assert_eq!(v["progress"]["total"], 1);
    assert_eq!(v["progress"]["done"], 0);
}

#[test]
fn show_json_omits_progress_key_for_non_epic() {
    let repo = make_initialized_repo("show_progress_json_non_epic");
    let bug_id = create_issue(&repo, "a bug", b"", &["--type", "bug"]);

    let out = run_jjf(&repo, &["show", "--json", &bug_id]);
    assert!(out.status.success());
    let v: serde_json::Value =
        serde_json::from_str(&String::from_utf8_lossy(&out.stdout)).expect("valid JSON");
    assert!(
        v.get("progress").is_none(),
        "non-epic JSON should have no progress key at all, got: {v}"
    );
}

#[test]
fn show_plain_zero_children_epic_shows_zero_over_zero() {
    let repo = make_initialized_repo("show_progress_empty_epic");
    let epic_id = create_issue(&repo, "Epic: empty", b"", &["--type", "epic"]);

    let out = run_jjf(&repo, &["show", &epic_id]);
    assert!(out.status.success());
    let stdout = String::from_utf8_lossy(&out.stdout);
    assert!(stdout.contains("0/0"), "got: {stdout}");
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `cargo test -p iss --test show show_.*progress 2>&1 | tail -60`
Expected: FAIL — `progress` isn't populated/rendered yet.

- [ ] **Step 6: Wire `run_show` to populate `progress` and pass it to `print_issue_plain`**

In `crates/iss/src/main.rs`, find `run_show` (around line 2415). Modify the section between resolving the issue (`let issue = storage.read(&issue_id)?;`) and the render branch:

```rust
    // 4. Hand off to storage. `IssueNotFound` flows out as a `Storage`
    // variant of `CliError`, which `exit_code` maps to 1.
    let mut issue = storage.read(&issue_id)?;

    // 4b. Epics get a computed progress summary — never persisted,
    // populated here at display time only.
    if issue.type_ == IssueType::Epic {
        issue.progress = Some(storage.epic_progress(&issue.id)?);
    }

    // 5. Render.
    if json {
        // The `Issue` struct IS the structured payload — emit it
        // verbatim, no `{"ok": true, ...}` envelope. (`init` and `new`
        // use the envelope because they have no payload beyond a
        // success signal; `show`'s whole job is to expose the record.)
        // `--include-memories` is plain-text only — JSON consumers
        // call `iss memories --json` for that.
        let s = serde_json::to_string_pretty(&issue)
            .map_err(|e| CliError::Storage(StorageError::Json(e)))?;
        println!("{s}");
    } else {
        print_issue_plain(&issue);
        if include_memories {
            let memories = storage.list_memories()?;
            print_memories_block(&memories);
        }
    }
    Ok(())
```

Note the `let issue = ...` becomes `let mut issue = ...` (needs to be mutable to set `.progress`).

- [ ] **Step 7: Add the embedded line to `print_issue_plain`**

In `crates/iss/src/main.rs`, find `print_issue_plain` (around line 2671). Locate the dependencies block's closing brace (the `if !issue.dependencies.is_empty() { ... }` block, ending right before the `created:`/`updated:` println — search for `println!(\n        "created: {}   updated: {}",` to find the exact spot). Insert the progress line right before that:

```rust
    // Epic progress: one line, only when the issue is an epic AND
    // `progress` was populated by the caller (run_show). Non-epic
    // issues never have this field set, so the line is naturally
    // absent for them.
    if let Some(progress) = &issue.progress {
        println!("{}", render_epic_progress_line(progress));
    }
    println!(
        "created: {}   updated: {}",
        issue.created_at, issue.updated_at
    );
```

`print_issue_plain`'s signature (`fn print_issue_plain(issue: &Issue)`) does NOT need to change — `progress` is already a field on `&Issue`, populated by the caller before this function is invoked. No new parameter needed.

- [ ] **Step 8: Run tests to verify they pass**

Run: `cargo test -p iss --test show show_.*progress 2>&1 | tail -60`
Expected: all 5 new tests PASS.

- [ ] **Step 9: Run the full `show.rs` test file to check for regressions**

Run: `cargo test -p iss --test show 2>&1 | tail -100`
Expected: all pass, including `show_json_emits_bug_record_verbatim` and any other JSON-shape test — these should be unaffected since `progress` is `skip_serializing_if = "Option::is_none"` and defaults to `None` for non-epics.

- [ ] **Step 10: Run the full workspace test suite**

Run: `cargo build --workspace 2>&1 | tail -30` then `cargo nextest run --workspace 2>&1 | tail -100` (fall back to `cargo test --workspace`).
Expected: all pass. If you hit stale-binary-cache errors unrelated to your change, run `cargo clean -p iss && cargo clean -p iss-storage && cargo build --workspace --tests` first. `iss-storage::v3_write_path::v3_write_path_source_does_not_invoke_jj` is a known pre-existing flaky test — rerun in isolation if it fails, unrelated to this work.

- [ ] **Step 11: Commit**

```bash
git add crates/iss-storage/src/record.rs crates/iss-storage/src/lib.rs crates/iss-storage/src/read.rs crates/iss-storage/src/cache.rs crates/iss/src/main.rs crates/iss/tests/show.rs
git commit -m "feat(show): embed epic progress line/field for type=epic issues"
```

---

## Final Verification

- [ ] **Step 1: Full workspace build + test**

```bash
cargo build --workspace 2>&1 | tail -30
cargo nextest run --workspace 2>&1 | tail -100
```
Expected: clean build, all tests pass.

- [ ] **Step 2: Live smoke test in a scratch repo**

```bash
mkdir -p /tmp/epic-progress-verify && cd /tmp/epic-progress-verify && rm -rf repo && mkdir repo && cd repo
git init
/Users/myers/p/git-issues/target/debug/iss init
EPIC=$(/Users/myers/p/git-issues/target/debug/iss new --json -t "Epic: verify" --type epic -F /dev/null | jq -r .id)
C1=$(/Users/myers/p/git-issues/target/debug/iss new --json -t "child 1" --parent "$EPIC" -F /dev/null | jq -r .id)
C2=$(/Users/myers/p/git-issues/target/debug/iss new --json -t "child 2" --parent "$EPIC" -F /dev/null | jq -r .id)
/Users/myers/p/git-issues/target/debug/iss close "$C1"
/Users/myers/p/git-issues/target/debug/iss dep progress "$EPIC"
/Users/myers/p/git-issues/target/debug/iss show "$EPIC"
/Users/myers/p/git-issues/target/debug/iss show --json "$EPIC" | jq .progress
```

Expected: `iss dep progress` prints a bar showing `1/2 (50%)`. `iss show` includes a `progress:` line with the same ratio. `iss show --json | jq .progress` prints `{"done": 1, "total": 2, "in_progress": 0, "optional_total": 0, "optional_done": 0}`.

Clean up: `rm -rf /tmp/epic-progress-verify`

- [ ] **Step 3: Push**

Per this repo's CLAUDE.md convention (push at the end of every issue/unit of work):

```bash
cd /Users/myers/p/git-issues
git push origin main
git push forgejo main
```
