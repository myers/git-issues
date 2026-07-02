# jj-cleanup sweep + CLI output lean pass Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove stale jj-era text/identifiers that survived the 2026-06-30 rename, and trim three sources of unnecessary CLI-output tokens (`iss show`'s always-on empty fields, `iss memories`' 4-lines-per-entry format, and `slugify`'s apostrophe-splitting bug).

**Architecture:** Two independent tracks in one plan. Track A is rename-only (doc comments, `--help` text, Rust identifiers, `docs/quickstart.md`) — zero behavior change, verified by `cargo build` catching every call site plus one test-comment update. Track B changes real `println!` output and one pure-function algorithm — each change gets a new or updated unit/integration test asserting the new shape.

**Tech Stack:** Rust (clap-derive CLI, `crates/iss` binary + `crates/iss-storage` lib), existing hermetic-scratch integration test harness (`crates/iss/tests/common/mod.rs`), Markdown docs.

## Global Constraints

- Per the 2026-06-30 rename design: `refs/jjf/*` ref namespace and `Jjf-*` commit-trailer keys are intentionally left as vestigial wire-format tokens — do NOT touch any mention of `refs/jjf/*` or `Jjf-*` trailers anywhere in this plan's tasks.
- Track B changes are **plain-text-output only** — every `--json` code path is untouched (all three affected functions already branch on `json: bool` / have a separate `--json` path before reaching the plain-text code).
- No new crates, no new files unless a task explicitly says so (none do — this plan only modifies existing files).
- Rust edition/toolchain: whatever `crates/iss/Cargo.toml` already pins — no version changes.

---

## File Structure

No new files. Files modified:

- `docs/quickstart.md` — jj → plain-git command/prose fixes, sample-output correction (Task 1).
- `crates/iss/src/main.rs` — `--help` doc comments (Remote/Init/RemoteAction), `CliError` variant rename (`JjGitRemote` → `GitRemoteError`), `print_issue_plain` (omit empty fields), memories-list rendering (Tasks 2, 3, 6, 7).
- `crates/iss/src/preflight.rs` — `jj_repo` → `git_repo` rename, doc-comment cleanup (Task 4).
- `crates/iss-storage/src/lib.rs` — `Error::NotAJjRepo` → `Error::NotAGitRepo` rename, doc-comment cleanup (Task 4).
- `crates/iss-storage/src/memory.rs` — `slugify` quote-stripping fix + doc comment + tests (Task 8).
- `crates/iss/tests/init.rs` — update comment referencing old variant name (Task 4).
- `crates/iss-storage/tests/integration.rs` — update comments referencing old variant name (Task 4).
- `crates/iss/tests/show.rs` — invert `show_plain_text_renders_none_for_unset_optionals`, add new omit-when-empty / include-when-set tests (Task 7).
- `crates/iss/tests/memory.rs` — add new one-line-per-entry format test (Task 6).

## Task Right-Sizing

Track A is split into 4 tasks by file-cluster (quickstart doc; remote/init help text; the JjGitRemote rename; the jj_repo/NotAJjRepo rename) because each is independently reviewable and touches a distinct file or a distinct identifier chain. Track B is split into 3 tasks (show, memories, slugify) — one per independently-testable behavior change. Each task ends with a passing test run.

---

### Task 1: Fix stale jj references in `docs/quickstart.md`

**Files:**
- Modify: `docs/quickstart.md`

**Interfaces:** None (pure doc edit, no code).

- [ ] **Step 1: Edit Prerequisites section**

Open `docs/quickstart.md`. Find the `## Prerequisites` section (starts line 7):

```markdown
## Prerequisites

- `jj` (Jujutsu) 0.40 or newer on `PATH`.
- The `iss` binary on `PATH`. From this repo:
  `cargo install --path crates/iss` — or symlink `bin/iss`
  (which prefers `target/release/iss`, falls back to debug,
  builds release on demand) somewhere on `PATH`.
- A jj identity configured (`jj config set --user user.name ...`,
  `user.email ...`). Without one, commit authorship is empty
  and `iss` will refuse to write.
```

Replace with:

```markdown
## Prerequisites

- The `iss` binary on `PATH`. From this repo:
  `cargo install --path crates/iss` — or symlink `bin/iss`
  (which prefers `target/release/iss`, falls back to debug,
  builds release on demand) somewhere on `PATH`.
- A git identity configured (`git config user.name ...`,
  `git config user.email ...`). Without one, commit authorship is
  empty and `iss` will refuse to write.
```

- [ ] **Step 2: Edit "1. Create the repo" section**

Find (starts around line 18):

```markdown
## 1. Create the repo

`iss` writes to a `refs/jjf/*` namespace on an existing jj repo;
it does not create a repo for you. `jj git init` produces a
colocated jj+git repo — you get `.git/` and `.jj/` side-by-side,
and `git push` / `pull` and `iss push` share the same remote:

```bash
mkdir my-project && cd my-project
jj git init
```
```

Replace with:

```markdown
## 1. Create the repo

`iss` writes to a `refs/jjf/*` namespace on an existing git repo;
it does not create a repo for you.

```bash
mkdir my-project && cd my-project
git init
```
```

- [ ] **Step 3: Fix step 5's sample `iss ready` output blocks**

Find (around lines 105-108 and 117-120):

```
3764c4b  open  1L  Backend handler crashes on empty input
6f227f7  open  1L  Epic: ship v1
```

and

```
6dbb571  open  1L  Write the README
6f227f7  open  1L  Epic: ship v1
```

Replace with the real 5-column tab-separated shape (`id status priority type title`, priority `-` when unset):

```
3764c4b	open	-	bug	Backend handler crashes on empty input
6f227f7	open	-	epic	Epic: ship v1
```

and

```
6dbb571	open	-	feature	Write the README
6f227f7	open	-	epic	Epic: ship v1
```

(Use literal tab characters between columns, matching the real `iss ready` output — not spaces.)

- [ ] **Step 4: Edit "9. Joining an existing project" section**

Find (around line 293-305):

```markdown
## 9. Joining an existing project

When you (or a collaborator) clone a git-issues project on a new
machine, the planner refs don't ride along by default — git's
default fetch refspec only covers `refs/heads/*`. The recipe:

```bash
jj git clone <url> <dir>
cd <dir>
iss init               # writes the refs/jjf/* fetch refspec
iss pull origin        # fetches issues, memories, sentinel
iss ls                 # see the project's open issues
```
```

Replace `jj git clone <url> <dir>` with `git clone <url> <dir>` (keep everything else in that block unchanged).

- [ ] **Step 5: Fix the Verified output transcript**

Find (around line 320):

```
$ jj git init
Initialized repo in "."
```

Replace with:

```
$ git init
Initialized empty Git repository in ...
```

- [ ] **Step 6: Verify no `jj` mentions remain outside `refs/jjf`**

Run: `grep -in 'jj[^f]' docs/quickstart.md`
Expected: no output (exit 1 from grep, meaning no matches). Any hit that isn't part of `refs/jjf/*` or `jjf/issues/...` is a miss — fix it before continuing.

- [ ] **Step 7: Commit**

```bash
git add docs/quickstart.md
git commit -m "docs(quickstart): drop stale jj commands, fix ready sample output"
```

---

### Task 2: Fix stale jj wording in `iss remote` / `iss init` `--help` text

**Files:**
- Modify: `crates/iss/src/main.rs:759-772` (Remote command doc comment)
- Modify: `crates/iss/src/main.rs:1149-1178` (RemoteAction enum doc comments)
- Modify: `crates/iss/src/main.rs:202-203` (Init command doc comment)
- Modify: `crates/iss/src/main.rs:3200-3216` (`run_remote_add` doc comment)
- Modify: `crates/iss/src/main.rs:3325-3337` (`run_remote_ls` doc comment)

**Interfaces:** None — doc comments only, no signature or behavior changes.

- [ ] **Step 1: Fix the `Remote` command doc comment**

In `crates/iss/src/main.rs`, find (around line 759):

```rust
    /// Manage git remotes on the underlying jj repo. Thin wrapper over
    /// `jj git remote add|list|remove` — jj already supports git
    /// transport for bookmarks (and bookmarks ARE the unit `issues`
    /// travels as), so this verb does NOT need to write per-bookmark
    /// refspec config. Verified in `experiments/sync-remote/`.
    ///
    /// Preflight is jj-repo-only (no `issues` bookmark required) —
    /// adding a remote is meaningful BEFORE `iss init` runs, and the
    /// soon-to-come `iss push` will be how the bookmark first reaches
    /// a remote.
    Remote {
```

Replace with:

```rust
    /// Manage git remotes on the underlying git repo. Thin wrapper
    /// over plain `git remote add|list|remove`.
    ///
    /// Preflight is repo-existence-only (no `issues` bookmark
    /// required) — adding a remote is meaningful BEFORE `iss init`
    /// runs, since `iss push` needs a remote to already exist.
    Remote {
```

- [ ] **Step 2: Fix the `RemoteAction::Add` doc comment**

Find (around line 1150):

```rust
    /// Add a git remote to the underlying jj repo. Wraps `jj git
    /// remote add <name> <url>`. URL is whatever git accepts; jj
    /// validates it and we surface its error verbatim. Adding a name
    /// that already exists is exit 2 (`remote_already_exists`).
    Add {
        /// Remote name (e.g. `origin`, `upstream`). Free-form string,
        /// jj decides what's legal.
        name: String,

        /// Remote URL or local path. Local paths are resolved to
        /// absolute form by jj.
        url: String,
    },
```

Replace with:

```rust
    /// Add a git remote to the underlying git repo. Wraps `git
    /// remote add <name> <url>`. URL is whatever git accepts; git
    /// validates it and we surface its error verbatim. Adding a name
    /// that already exists is exit 2 (`remote_already_exists`).
    Add {
        /// Remote name (e.g. `origin`, `upstream`). Free-form string,
        /// git decides what's legal.
        name: String,

        /// Remote URL or local path. Local paths are resolved to
        /// absolute form by git.
        url: String,
    },
```

- [ ] **Step 3: Fix the `RemoteAction::Ls` doc comment**

Find (around line 1164):

```rust
    /// List configured git remotes. Plain-text output is one
    /// `<name>\t<url>` per line (tab-separated, no header — matches
    /// the `ls`-style convention every other read verb in jjforge
    /// uses). `--json` emits a JSON array of `{name, url}` objects.
    Ls,
```

Replace with:

```rust
    /// List configured git remotes. Plain-text output is one
    /// `<name>\t<url>` per line (tab-separated, no header — matches
    /// the `ls`-style convention every other read verb in git-issues
    /// uses). `--json` emits a JSON array of `{name, url}` objects.
    Ls,
```

- [ ] **Step 4: Fix the `RemoteAction::Rm` doc comment**

Find (around line 1170):

```rust
    /// Remove a git remote from the underlying jj repo. Wraps `jj git
    /// remote remove <name>` — note that jj also forgets bookmarks
    /// tracked from that remote (its own behavior, not jjforge's).
    /// Removing a name that doesn't exist is exit 2 (`remote_not_found`).
    Rm {
```

Replace with:

```rust
    /// Remove a git remote from the underlying git repo. Wraps `git
    /// remote remove <name>`. Removing a name that doesn't exist is
    /// exit 2 (`remote_not_found`).
    Rm {
```

- [ ] **Step 5: Fix the `Init` command doc comment**

Find (around line 202):

```rust
    /// Initialize the `issues` bookmark on the current jj repo.
    /// Idempotent — running twice in the same repo is a no-op.
    Init,
```

Replace with:

```rust
    /// Initialize git-issues in the current git repo. Idempotent —
    /// running twice in the same repo is a no-op.
    Init,
```

- [ ] **Step 6: Fix `run_remote_add`'s doc comment**

Find (around line 3200, the doc comment directly above `fn run_remote_add`):

```rust
/// against the cwd's git repo.
///
/// git does the actual remote-add work; we translate the two specific
/// error stderrs we recognize (`already exists`, anything else) into
/// typed `CliError` variants so `kind()` stays stable. URL syntax
/// validation is git's responsibility — we accept what it accepts and
/// surface its rejection unchanged.
///
/// Preflight is the repo-existence check only (no `issues` bookmark
/// required), because adding a remote is meaningful before `iss init`
/// runs.
///
/// After git registers the remote we also add the v3 fetch refspec
/// (`+refs/jjf/*:refs/remotes/<name>/jjf/*`) to `.git/config`. Without
/// this, a plain `git fetch <name>` carries refs under `refs/heads/*`
/// only and leaves the jjforge namespace empty on the new clone (see
/// ticket `eaf0674`).
fn run_remote_add(json: bool, name: String, url: String) -> Result<(), CliError> {
```

Replace the last paragraph's `jjforge namespace` with `git-issues namespace` (the rest of this comment is already jj-free — leave it):

```rust
/// After git registers the remote we also add the v3 fetch refspec
/// (`+refs/jjf/*:refs/remotes/<name>/jjf/*`) to `.git/config`. Without
/// this, a plain `git fetch <name>` carries refs under `refs/heads/*`
/// only and leaves the git-issues namespace empty on the new clone
/// (see ticket `eaf0674`).
fn run_remote_add(json: bool, name: String, url: String) -> Result<(), CliError> {
```

- [ ] **Step 7: Fix `run_remote_ls`'s doc comment**

Find (around line 3328):

```rust
/// `git remote -v` prints two lines per remote (fetch + push); we
/// dedupe to fetch-only and re-render because every other `ls`-style
/// verb in jjforge emits tab-separated columns, and a stable separator
/// means downstream `cut -f1` / `awk -F'\t'` pipelines don't have to
/// guess at column widths.
```

Replace `jjforge` with `git-issues`:

```rust
/// `git remote -v` prints two lines per remote (fetch + push); we
/// dedupe to fetch-only and re-render because every other `ls`-style
/// verb in git-issues emits tab-separated columns, and a stable
/// separator means downstream `cut -f1` / `awk -F'\t'` pipelines
/// don't have to guess at column widths.
```

- [ ] **Step 8: Build and eyeball the real `--help` output**

Run: `cargo build -p iss 2>&1 | tail -20`
Expected: clean build, no errors.

Run: `./target/debug/iss remote --help && ./target/debug/iss remote add --help && ./target/debug/iss init --help`
Expected: no "jj" mentions anywhere in the output except `refs/jjf/*` (if it appears).

- [ ] **Step 9: Commit**

```bash
git add crates/iss/src/main.rs
git commit -m "docs(help): drop stale jj wrapper language from remote/init --help"
```

---

### Task 3: Rename `CliError::JjGitRemote` → `GitRemoteError`

**Files:**
- Modify: `crates/iss/src/main.rs:1306-1312` (variant definition + doc comment)
- Modify: `crates/iss/src/main.rs:1649` (exit-code match arm)
- Modify: `crates/iss/src/main.rs:1725` (`kind()` match arm)
- Modify: `crates/iss/src/main.rs:3234` (`run_remote_add` error construction)
- Modify: `crates/iss/src/main.rs:3353` (`run_remote_ls` error construction)
- Modify: `crates/iss/src/main.rs:3411` (doc comment mentioning `JjGitRemote`)
- Modify: `crates/iss/src/main.rs:3430` (`run_remote_rm` error construction)

**Interfaces:**
- Produces: `CliError::GitRemoteError(String)` (renamed from `JjGitRemote`). `kind()` for this variant now returns `"git_remote_error"` (renamed from `"jj_git_remote_error"`). This is a machine-readable `--json` error-envelope change — any external script matching on the old kind string breaks (acceptable per design: git-issues has no external consumers yet).

- [ ] **Step 1: Rename the variant definition**

In `crates/iss/src/main.rs`, find (around line 1306):

```rust
    /// `iss remote *` shelled out to `jj git remote ...` and got a
    /// non-zero exit that wasn't one of the two typed cases above.
    /// Runtime failure (exit 1) — surfaces git's stderr verbatim so
    /// the operator can see what git said. URL syntax errors, network-
    /// adjacent failures, and anything else git rejects land here.
    #[error("git remote failed: {0}")]
    JjGitRemote(String),
```

Replace with:

```rust
    /// `iss remote *` shelled out to `git remote ...` and got a
    /// non-zero exit that wasn't one of the two typed cases above.
    /// Runtime failure (exit 1) — surfaces git's stderr verbatim so
    /// the operator can see what git said. URL syntax errors, network-
    /// adjacent failures, and anything else git rejects land here.
    #[error("git remote failed: {0}")]
    GitRemoteError(String),
```

- [ ] **Step 2: Rename the exit-code match arm**

Find (around line 1649): `CliError::JjGitRemote(_) => 1,`
Replace with: `CliError::GitRemoteError(_) => 1,`

- [ ] **Step 3: Rename the `kind()` match arm**

Find (around line 1725): `CliError::JjGitRemote(_) => "jj_git_remote_error",`
Replace with: `CliError::GitRemoteError(_) => "git_remote_error",`

- [ ] **Step 4: Rename the three construction sites**

In `run_remote_add` (around line 3234):
Find: `return Err(CliError::JjGitRemote(stderr.trim().to_owned()));`
Replace: `return Err(CliError::GitRemoteError(stderr.trim().to_owned()));`

In `run_remote_ls` (around line 3353):
Find:
```rust
        return Err(CliError::JjGitRemote(
            String::from_utf8_lossy(&out.stderr).trim().to_owned(),
        ));
```
Replace:
```rust
        return Err(CliError::GitRemoteError(
            String::from_utf8_lossy(&out.stderr).trim().to_owned(),
        ));
```

In `run_remote_rm` (around line 3430):
Find: `return Err(CliError::JjGitRemote(stderr.trim().to_owned()));`
Replace: `return Err(CliError::GitRemoteError(stderr.trim().to_owned()));`

- [ ] **Step 5: Fix the doc comment referencing the old name**

Find (around line 3411, the comment above `run_remote_rm`):
```rust
/// (exit 2); anything else falls through to `JjGitRemote` (exit 1).
```
Replace:
```rust
/// (exit 2); anything else falls through to `GitRemoteError` (exit 1).
```

- [ ] **Step 6: Search for any remaining `JjGitRemote` references**

Run: `grep -rn "JjGitRemote\|jj_git_remote_error" crates/`
Expected: no output.

- [ ] **Step 7: Build**

Run: `cargo build -p iss 2>&1 | tail -30`
Expected: clean build. If `cargo build` reports unmatched patterns or unresolved names, it means a reference was missed — grep again and fix.

- [ ] **Step 8: Run the existing remote-related tests**

Run: `cargo test -p iss --test remote 2>&1 | tail -40` (if no `tests/remote.rs` exists, run `cargo test -p iss 2>&1 | tail -60` instead and check for remote-related test names).
Expected: all pass (the Display message `"git remote failed: {0}"` is unchanged, so any test asserting on that string still passes; only the identifier changed).

- [ ] **Step 9: Commit**

```bash
git add crates/iss/src/main.rs
git commit -m "refactor: rename CliError::JjGitRemote -> GitRemoteError"
```

---

### Task 4: Rename `preflight::jj_repo` → `git_repo` and `StorageError::NotAJjRepo` → `NotAGitRepo`

**Files:**
- Modify: `crates/iss/src/preflight.rs:28-50`
- Modify: `crates/iss-storage/src/lib.rs:196-204`
- Modify: `crates/iss/src/main.rs:1185, 1600, 1677, 1756, 2167` (references)
- Modify: `crates/iss/tests/init.rs:93` (comment only)
- Modify: `crates/iss-storage/tests/integration.rs:1102, 1104` (comments only)

**Interfaces:**
- Produces: `preflight::git_repo(cwd: &Path) -> Result<(), CliError>` (renamed from `jj_repo`, same signature/body). `iss_storage::Error::NotAGitRepo(PathBuf)` (renamed from `NotAJjRepo`, same Display message `"not a git repo: {0}"`, unchanged). `kind()` for this variant now returns `"not_a_git_repo"` (renamed from `"not_a_jj_repo"`) — another `--json` error-envelope machine-readable change, same acceptable-breakage rationale as Task 3.
- Consumes: nothing new from other tasks.

- [ ] **Step 1: Rename the `Error` variant in `iss-storage`**

In `crates/iss-storage/src/lib.rs`, find (around line 196):

```rust
    /// The repo root passed to `Storage::init` / `Storage::open` does not
    /// hold a jj repo. Distinct from `Jj` so callers can tell "not a
    /// repo at all" from "jj broke for some other reason" without
    /// string-matching stderr.
    #[error("not a git repo: {0}")]
    NotAJjRepo(PathBuf),
```

Replace with:

```rust
    /// The repo root passed to `Storage::init` / `Storage::open` does
    /// not hold a git repo.
    #[error("not a git repo: {0}")]
    NotAGitRepo(PathBuf),
```

(Dropped the "Distinct from `Jj`" sentence — that sibling variant no longer exists.)

- [ ] **Step 2: Rename the preflight function and its internal use**

In `crates/iss/src/preflight.rs`, find (around line 28):

```rust
/// Probe that `cwd` is inside a git repo. Shells out to `git rev-parse
/// --git-dir` and translates a non-zero exit into `NotAJjRepo`;
/// everything else becomes a generic `Probe` failure.
///
/// Callers that ALSO need the `issues` bookmark (every read/write
/// verb that touches an existing issue) should use
/// [`issues_bookmark`] instead — it composes this probe with the
/// bookmark check. Callers that meaningfully run before `iss init`
/// (today: `iss remote add|ls|rm`) call this one directly.
pub(crate) fn jj_repo(cwd: &Path) -> Result<(), CliError> {
    let out = Command::new("git")
        .arg("-C")
        .arg(cwd)
        .args(["rev-parse", "--git-dir"])
        .output()
        .map_err(CliError::Probe)?;
    if !out.status.success() {
        return Err(CliError::Storage(StorageError::NotAJjRepo(
            PathBuf::from(cwd),
        )));
    }
    Ok(())
}
```

Replace with:

```rust
/// Probe that `cwd` is inside a git repo. Shells out to `git rev-parse
/// --git-dir` and translates a non-zero exit into `NotAGitRepo`;
/// everything else becomes a generic `Probe` failure.
///
/// Callers that ALSO need the `issues` bookmark (every read/write
/// verb that touches an existing issue) should use
/// [`issues_bookmark`] instead — it composes this probe with the
/// bookmark check. Callers that meaningfully run before `iss init`
/// (today: `iss remote add|ls|rm`) call this one directly.
pub(crate) fn git_repo(cwd: &Path) -> Result<(), CliError> {
    let out = Command::new("git")
        .arg("-C")
        .arg(cwd)
        .args(["rev-parse", "--git-dir"])
        .output()
        .map_err(CliError::Probe)?;
    if !out.status.success() {
        return Err(CliError::Storage(StorageError::NotAGitRepo(
            PathBuf::from(cwd),
        )));
    }
    Ok(())
}
```

- [ ] **Step 3: Fix the `issues_bookmark` doc comment referencing `jj_repo`**

In `crates/iss/src/preflight.rs`, find the doc comment above `issues_bookmark` (a few lines below what you just edited):

```rust
/// Most read/write verbs (`iss new`, `iss show`, `iss ls`, etc.) call
/// this. The `iss remote *` verbs use the simpler [`jj_repo`] probe
/// because remote setup is meaningful before `iss init`.
pub(crate) fn issues_bookmark(cwd: &Path) -> Result<(), CliError> {
    // Check 1: is this a git repo at all?
    jj_repo(cwd)?;
```

Replace with:

```rust
/// Most read/write verbs (`iss new`, `iss show`, `iss ls`, etc.) call
/// this. The `iss remote *` verbs use the simpler [`git_repo`] probe
/// because remote setup is meaningful before `iss init`.
pub(crate) fn issues_bookmark(cwd: &Path) -> Result<(), CliError> {
    // Check 1: is this a git repo at all?
    git_repo(cwd)?;
```

- [ ] **Step 4: Find and rename every remaining `jj_repo(` call site**

Run: `grep -rn "jj_repo(" crates/iss/src/main.rs`

For each hit (expect `run_remote_add`, `run_remote_ls`, `run_remote_rm`, and possibly others), change `preflight::jj_repo(` to `preflight::git_repo(`.

- [ ] **Step 5: Find and rename every remaining `NotAJjRepo` reference in `main.rs`**

Run: `grep -n "NotAJjRepo" crates/iss/src/main.rs`

Expect hits around lines 1185, 1600, 1677, 1756, 2167. For each:

Line ~1185 (doc comment on `CliError::Storage` variant):
Find: `/// Bubbled up from \`jjf-storage\`. The \`NotAJjRepo\` variant gets`
Replace: `/// Bubbled up from \`iss-storage\`. The \`NotAGitRepo\` variant gets`

Line ~1600 (exit-code match arm):
Find: `CliError::Storage(StorageError::NotAJjRepo(_)) => 2,`
Replace: `CliError::Storage(StorageError::NotAGitRepo(_)) => 2,`

Line ~1677 (`kind()` match arm):
Find: `CliError::Storage(StorageError::NotAJjRepo(_)) => "not_a_jj_repo",`
Replace: `CliError::Storage(StorageError::NotAGitRepo(_)) => "not_a_git_repo",`

Line ~1756 (`details()` match arm):
Find:
```rust
            CliError::Storage(StorageError::NotAJjRepo(path)) => {
                json!({ "path": path.display().to_string() })
            }
```
Replace:
```rust
            CliError::Storage(StorageError::NotAGitRepo(path)) => {
                json!({ "path": path.display().to_string() })
            }
```

Line ~2167 (a plain comment, not code):
Find: `// surfaces the typed \`NotAJjRepo\` error (exit 2, "run inside a`
Replace: `// surfaces the typed \`NotAGitRepo\` error (exit 2, "run inside a`

- [ ] **Step 6: Update the test comment in `init.rs`**

In `crates/iss/tests/init.rs`, find (around line 93):

```rust
    // The exact phrasing comes from `StorageError::NotAJjRepo`'s
    // Display impl ("not a git repo: <path>"). We assert on both the
    // tag and the path so a future error-message rewording can't
    // silently strip context.
```

Replace with:

```rust
    // The exact phrasing comes from `StorageError::NotAGitRepo`'s
    // Display impl ("not a git repo: <path>"). We assert on both the
    // tag and the path so a future error-message rewording can't
    // silently strip context.
```

(The assertions themselves — `stderr.contains("not a git repo")` and the path check — are unchanged, since the Display message text didn't change.)

- [ ] **Step 7: Update the comments in `iss-storage`'s integration test**

In `crates/iss-storage/tests/integration.rs`, find (around lines 1102-1104):

```rust
    // `Error::Git` rather than the (deleted) jj-probe's `NotAJjRepo`.
```
and
```rust
    // to the operator-facing `NotAJjRepo` (exit 2) — see the `iss init`
```

Replace both `NotAJjRepo` mentions with `NotAGitRepo` (keep the surrounding sentence structure unchanged).

- [ ] **Step 8: Search for any remaining `jj_repo` or `NotAJjRepo` references anywhere in `crates/`**

Run: `grep -rn "jj_repo\b\|NotAJjRepo\|not_a_jj_repo" crates/`
Expected: no output. If anything remains, fix it before continuing.

- [ ] **Step 9: Build**

Run: `cargo build --workspace 2>&1 | tail -40`
Expected: clean build across both `iss` and `iss-storage` crates.

- [ ] **Step 10: Run the full test suite**

Run: `cargo nextest run --workspace 2>&1 | tail -60` (fall back to `cargo test --workspace` if nextest isn't installed).
Expected: all tests pass, including `init_in_non_jj_directory_fails_with_exit_two_and_useful_stderr` in `init.rs` (its assertions are string-content-based and don't reference the Rust identifier, so they should pass unchanged).

- [ ] **Step 11: Commit**

```bash
git add crates/iss/src/preflight.rs crates/iss-storage/src/lib.rs crates/iss/src/main.rs crates/iss/tests/init.rs crates/iss-storage/tests/integration.rs
git commit -m "refactor: rename jj_repo/NotAJjRepo -> git_repo/NotAGitRepo"
```

---

### Task 5: Fix `iss init --help`'s remaining `Storage::init` doc-comment jj mention (sweep check)

**Files:**
- Modify: `crates/iss/src/main.rs` (wherever the `Init` command's *implementation* — not just its `--help` doc comment already fixed in Task 2 — lives, if it has jj-stale comments)

**Interfaces:** None — doc comment only.

- [ ] **Step 1: Grep for any remaining `jj` mentions in `main.rs` outside `refs/jjf`/`Jjf-`**

Run: `grep -n "jj" crates/iss/src/main.rs | grep -v "refs/jjf\|Jjf-\|jjf-storage\|jjf-merge"`

Review each hit. Tasks 2-4 already covered the ones identified during planning (Remote/Init help text, JjGitRemote, jj_repo/NotAJjRepo). If this grep turns up additional stale mentions not yet fixed (e.g. in the `Init` command's execution function, or elsewhere), fix them the same way: replace "jj repo" with "git repo", drop any "wraps jj" language that no longer reflects plain-git implementation, per the same pattern as Task 2.

If the grep shows only `jjf-storage`/`jjf-merge` crate-name references (which may be legitimate Cargo dependency names, not human-surface prose — check `Cargo.toml` before touching), leave those alone; they're wire/build-system tokens, not doc prose.

- [ ] **Step 2: Build**

Run: `cargo build -p iss 2>&1 | tail -20`
Expected: clean build.

- [ ] **Step 3: Commit (only if Step 1 found and fixed something)**

```bash
git add crates/iss/src/main.rs
git commit -m "docs(help): sweep remaining stale jj mentions in main.rs"
```

If Step 1 found nothing beyond what Tasks 2-4 already fixed, skip this commit — there's nothing to commit.

---

### Task 6: `iss memories` — one line per entry, no header

**Files:**
- Modify: `crates/iss/src/main.rs` (the memories-list plain-text branch, around lines 2579-2599)
- Test: `crates/iss/tests/memory.rs`

**Interfaces:**
- Produces: plain-text `iss memories` (and `iss memories <search>`) output is now `N` lines total (one per memory, `{key}\t{value}` tab-separated, value truncated via the existing `truncate_memory(&m.value, 120)`), with zero header line and zero blank lines. The two empty-result messages ("no memories stored...", "no memories matching...") are unchanged. `--json` output is unaffected (separate code path, untouched).

- [ ] **Step 1: Read the current implementation to confirm exact line numbers**

Run: `grep -n "memories ({}" crates/iss/src/main.rs` and `grep -n "memories matching" crates/iss/src/main.rs` to pinpoint the exact current line numbers before editing (they may have shifted slightly from earlier tasks' edits).

- [ ] **Step 2: Write the failing test**

In `crates/iss/tests/memory.rs`, add this test after `memories_filters_by_substring_case_insensitive` (after line 219, before the `memories_json_returns_bare_array` test at line 221):

```rust
#[test]
fn memories_plain_text_is_one_line_per_entry_no_header() {
    let repo = make_initialized("memory_lean_format");
    run_jjf(&repo, &["remember", "first fact", "--key", "alpha"]);
    run_jjf(&repo, &["remember", "second fact", "--key", "beta"]);

    let out = run_jjf(&repo, &["memories"]);
    assert!(out.status.success());
    let stdout = String::from_utf8_lossy(&out.stdout);
    let lines: Vec<&str> = stdout.lines().collect();

    assert_eq!(
        lines.len(),
        2,
        "expected exactly 2 lines (one per memory, no header/blank lines), got: {stdout:?}"
    );
    assert!(
        !stdout.contains("memories ("),
        "header line should be gone, got: {stdout}"
    );
    assert_eq!(lines[0], "alpha\tfirst fact");
    assert_eq!(lines[1], "beta\tsecond fact");
}
```

- [ ] **Step 3: Run it to confirm it fails**

Run: `cargo test -p iss --test memory memories_plain_text_is_one_line_per_entry_no_header 2>&1 | tail -30`
Expected: FAIL — either the line count assertion fails (currently 6 lines: header + blank + 2×(key+value) = 6) or the exact-line-content assertions fail.

- [ ] **Step 4: Implement the leaner format**

In `crates/iss/src/main.rs`, find the memories-list plain-text rendering block:

```rust
    if let Some(s) = &search {
        println!("memories matching {s:?}:");
    } else {
        println!("memories ({}):", memories.len());
    }
    println!();
    for m in &memories {
        println!("{}", m.key);
        println!("  {}", truncate_memory(&m.value, 120));
        println!();
    }
    Ok(())
```

Replace with:

```rust
    for m in &memories {
        println!("{}\t{}", m.key, truncate_memory(&m.value, 120));
    }
    Ok(())
```

(This drops the `search`-vs-not-search header distinction entirely, along with the blank lines. The `search` variable may now be unused in this branch after the header lines are removed — check the surrounding code; if `search` is still used above this block for filtering the `memories` vec, that's fine and this edit doesn't affect it. If Rust reports an unused-variable warning for `search` specifically in this scope, that means it truly has no other use in this function and the warning should be investigated, not silenced — but per the code above this block already filters `memories` by `search`, so it stays in use.)

- [ ] **Step 5: Run the test to confirm it passes**

Run: `cargo test -p iss --test memory memories_plain_text_is_one_line_per_entry_no_header 2>&1 | tail -30`
Expected: PASS.

- [ ] **Step 6: Run the full memory test file to check for regressions**

Run: `cargo test -p iss --test memory 2>&1 | tail -60`
Expected: all pass, including `memories_lists_all_keys_alphabetically` and `memories_filters_by_substring_case_insensitive` (both check substring/ordering, not exact format, so they should be unaffected).

- [ ] **Step 7: Commit**

```bash
git add crates/iss/src/main.rs crates/iss/tests/memory.rs
git commit -m "feat(memories): one line per entry, no header/blank lines"
```

---

### Task 7: `iss show` — omit empty labels/priority/assignee/dependencies lines

**Files:**
- Modify: `crates/iss/src/main.rs:2677-2762` (`print_issue_plain`)
- Test: `crates/iss/tests/show.rs`

**Interfaces:**
- Produces: `print_issue_plain(issue: &Issue)` (unchanged signature) now omits the `labels:`, `priority:`, `assignee:`, and `dependencies: (none)` lines entirely when those fields are empty/unset, matching the existing `metadata:` omit-when-empty pattern in the same function. `slug:` keeps its `(none)` fallback (single always-relevant identity field, not in scope). `--json` output unaffected.

- [ ] **Step 1: Write the failing tests — replace the old "renders none" test**

In `crates/iss/tests/show.rs`, find the existing test (lines 97-121):

```rust
#[test]
fn show_plain_text_renders_none_for_unset_optionals() {
    // No labels, no assignee, empty body. The plain-text renderer
    // should print `(none)` for the optional fields rather than going
    // blank or eliding the line entirely.
    let repo = make_initialized_repo("show_none");
    let id = create_issue(&repo, "bare bug", b"", &[]);

    let out = run_jjf(&repo, &["show", &id]);
    assert!(out.status.success(), "{}", String::from_utf8_lossy(&out.stderr));
    let stdout = String::from_utf8_lossy(&out.stdout);

    assert!(
        stdout.contains("labels: (none)"),
        "labels line missing/wrong: {stdout}"
    );
    assert!(
        stdout.contains("assignee: (none)"),
        "assignee line missing/wrong: {stdout}"
    );
    assert!(
        stdout.contains("dependencies: (none)"),
        "dependencies line missing/wrong: {stdout}"
    );
}
```

Replace with:

```rust
#[test]
fn show_plain_text_omits_empty_optionals() {
    // No labels, no priority, no assignee, no deps, empty body. The
    // plain-text renderer should omit those four lines entirely
    // rather than printing `(none)` placeholders — matches the
    // existing metadata-block omit-when-empty convention.
    let repo = make_initialized_repo("show_omit_empty");
    let id = create_issue(&repo, "bare bug", b"", &[]);

    let out = run_jjf(&repo, &["show", &id]);
    assert!(out.status.success(), "{}", String::from_utf8_lossy(&out.stderr));
    let stdout = String::from_utf8_lossy(&out.stdout);

    assert!(
        !stdout.contains("labels:"),
        "labels line should be omitted when empty: {stdout}"
    );
    assert!(
        !stdout.contains("priority:"),
        "priority line should be omitted when unset: {stdout}"
    );
    assert!(
        !stdout.contains("assignee:"),
        "assignee line should be omitted when unset: {stdout}"
    );
    assert!(
        !stdout.contains("dependencies:"),
        "dependencies line should be omitted when empty: {stdout}"
    );
    // slug: keeps its `(none)` fallback — not in scope for omission.
    assert!(
        stdout.contains("slug: (none)"),
        "slug line should still render with (none) fallback: {stdout}"
    );
}

#[test]
fn show_plain_text_includes_set_optionals() {
    // When labels/priority/assignee/deps ARE set, the lines must
    // still render with their real values — the omission only
    // applies to the empty case.
    let repo = make_initialized_repo("show_include_set");
    let id = create_issue(
        &repo,
        "populated bug",
        b"body",
        &["-l", "bug", "-p", "2", "-a", "carol"],
    );

    let out = run_jjf(&repo, &["show", &id]);
    assert!(out.status.success(), "{}", String::from_utf8_lossy(&out.stderr));
    let stdout = String::from_utf8_lossy(&out.stdout);

    assert!(
        stdout.contains("labels: bug"),
        "labels line missing/wrong: {stdout}"
    );
    assert!(
        stdout.contains("priority: P2"),
        "priority line missing/wrong: {stdout}"
    );
    assert!(
        stdout.contains("assignee: carol"),
        "assignee line missing/wrong: {stdout}"
    );
}
```

- [ ] **Step 2: Run to confirm both new tests fail (or the old one is gone and these are new)**

Run: `cargo test -p iss --test show show_plain_text_omits_empty_optionals show_plain_text_includes_set_optionals 2>&1 | tail -40`
Expected: `show_plain_text_omits_empty_optionals` FAILs (current code still prints `labels: (none)` etc.); `show_plain_text_includes_set_optionals` should already PASS (no behavior change needed for the populated case) — confirm it does, since that locks in the "don't break the populated path" guarantee before you touch the code.

- [ ] **Step 3: Implement the omit-when-empty change**

In `crates/iss/src/main.rs`, find `print_issue_plain` (around line 2677). The current relevant block:

```rust
    let labels = if issue.labels.is_empty() {
        "(none)".to_owned()
    } else {
        issue.labels.join(", ")
    };
    println!("labels: {labels}");
    // Metadata block: render between labels and dependencies,
    // mirroring the JSON field ordering (metadata sits after labels in
    // the IssueRecord struct). Omitted entirely when the map is empty.
    if !issue.metadata.is_empty() {
        println!("metadata:");
        for (k, v) in &issue.metadata {
            // BTreeMap iterates in sorted key order — no re-sort needed.
            println!("  {k}={v}");
        }
    }
    // v2.8: priority renders as `P0`..`P4` when set, `(none)` when
    // null — mirrors slug / assignee's Optional-presentation
    // convention rather than the row-format dash (the show output
    // is human-readable; the row format is the awk-friendly one).
    let priority = match issue.priority {
        Some(n) => format!("P{n}"),
        None => "(none)".to_owned(),
    };
    println!("priority: {priority}");
    let assignee = issue.assignee.as_deref().unwrap_or("(none)");
    println!("assignee: {assignee}");
    // v2.4: the dependency section renders one line per kind so the
    // typed-edge model is visible at a glance. Empty kinds are
    // collapsed; an entirely empty dep set falls back to the v1 shape
    // `dependencies: (none)`.
    if issue.dependencies.is_empty() {
        println!("dependencies: (none)");
    } else {
        println!("dependencies:");
```

Replace with:

```rust
    // v2.11: labels/priority/assignee/dependencies each omit their
    // line entirely when empty/unset, mirroring the metadata block's
    // existing omit-when-empty convention two lines below. A fully
    // bare issue now renders with no empty-field lines at all.
    if !issue.labels.is_empty() {
        println!("labels: {}", issue.labels.join(", "));
    }
    // Metadata block: render between labels and dependencies,
    // mirroring the JSON field ordering (metadata sits after labels in
    // the IssueRecord struct). Omitted entirely when the map is empty.
    if !issue.metadata.is_empty() {
        println!("metadata:");
        for (k, v) in &issue.metadata {
            // BTreeMap iterates in sorted key order — no re-sort needed.
            println!("  {k}={v}");
        }
    }
    // v2.8: priority renders as `P0`..`P4` when set. Omitted entirely
    // when unset (v2.11) — matches the metadata block's convention.
    if let Some(n) = issue.priority {
        println!("priority: P{n}");
    }
    if let Some(assignee) = issue.assignee.as_deref() {
        println!("assignee: {assignee}");
    }
    // v2.4: the dependency section renders one line per kind so the
    // typed-edge model is visible at a glance. Empty kinds are
    // collapsed; an entirely empty dep set omits the whole
    // `dependencies:` line (v2.11 — was `dependencies: (none)`).
    if !issue.dependencies.is_empty() {
        println!("dependencies:");
```

Note: the `else { println!("dependencies: (none)"); }` branch that followed the old `if issue.dependencies.is_empty()` is now gone entirely — the `if !issue.dependencies.is_empty() { println!("dependencies:"); ... }` block simply doesn't execute when empty, with no `else`. Make sure the closing brace structure after the `for kind in [...]` loop still matches (it should — you're only changing the `if`/`else` at the top of that block into a single `if`, the loop body inside is untouched).

- [ ] **Step 4: Run to confirm both tests pass**

Run: `cargo test -p iss --test show show_plain_text_omits_empty_optionals show_plain_text_includes_set_optionals 2>&1 | tail -40`
Expected: both PASS.

- [ ] **Step 5: Run the full show test file to check for regressions**

Run: `cargo test -p iss --test show 2>&1 | tail -80`
Expected: all pass, including `show_plain_text_includes_every_scalar_field`, `show_plain_renders_metadata_block`, `show_plain_omits_metadata_when_empty`. Pay particular attention to `show_plain_text_includes_every_scalar_field` (line 47) — it creates an issue with labels `bug`/`p1` and assignee `alice`, all non-empty, so its assertions (`stdout.contains("bug")`, etc.) should be unaffected by the omit-when-empty change.

- [ ] **Step 6: Run the full workspace test suite**

Run: `cargo nextest run --workspace 2>&1 | tail -80` (fall back to `cargo test --workspace` if nextest isn't installed).
Expected: all pass. Watch specifically for any other test file that might assert on `"labels: (none)"`, `"priority: (none)"`, `"assignee: (none)"`, or `"dependencies: (none)"` substrings outside `show.rs` — if `grep -rn '"(labels\|priority\|assignee\|dependencies): (none)"' crates/iss/tests/` (run before this step, or immediately if a failure surfaces here) finds any, update them the same way as Step 1 of this task.

- [ ] **Step 7: Commit**

```bash
git add crates/iss/src/main.rs crates/iss/tests/show.rs
git commit -m "feat(show): omit empty labels/priority/assignee/dependencies lines"
```

---

### Task 8: `slugify` — strip quote/apostrophe characters instead of splitting on them

**Files:**
- Modify: `crates/iss-storage/src/memory.rs:52-94` (`slugify` function + its doc comment)
- Test: `crates/iss-storage/src/memory.rs` (inline `#[cfg(test)] mod tests`, around lines 160-198)

**Interfaces:**
- Produces: `pub fn slugify(s: &str) -> String` (unchanged signature). Behavior change: ASCII apostrophe (`'`, U+0027), right single quotation mark (`'`, U+2019), ASCII double quote (`"`, U+0022), left double quotation mark (`"`, U+201C), and right double quotation mark (`"`, U+201D) are now stripped (deleted, not converted to a hyphen) before the existing lowercase + non-alphanumeric-collapse pass runs. Everything else in the algorithm (lowercase, other non-alphanumeric runs → single `-`, leading/trailing trim, first-8-tokens cap, 60-char cap) is unchanged.
- Consumes: nothing from other tasks — this is a standalone pure-function fix.

- [ ] **Step 1: Write the failing tests**

In `crates/iss-storage/src/memory.rs`, find the `slugify_basic` test (around line 162) and add new test functions immediately after it (before `slugify_caps_eight_tokens`):

```rust
    #[test]
    fn slugify_strips_apostrophes_instead_of_splitting() {
        assert_eq!(slugify("can't"), "cant");
        assert_eq!(
            slugify("Backend's /submit handler can't take empty bodies."),
            "backends-submit-handler-cant-take-empty-bodies"
        );
    }

    #[test]
    fn slugify_strips_curly_apostrophe() {
        assert_eq!(slugify("don\u{2019}t stop"), "dont-stop");
    }

    #[test]
    fn slugify_strips_double_quotes() {
        assert_eq!(slugify("a \"quoted\" phrase"), "a-quoted-phrase");
        assert_eq!(
            slugify("a \u{201C}curly quoted\u{201D} phrase"),
            "a-curly-quoted-phrase"
        );
    }
```

- [ ] **Step 2: Run to confirm they fail**

Run: `cargo test -p iss-storage slugify_strips 2>&1 | tail -40`
Expected: FAIL. `slugify("can't")` currently produces `"can-t"` (apostrophe becomes a hyphen-run separator), not `"cant"`.

- [ ] **Step 3: Implement the fix**

In `crates/iss-storage/src/memory.rs`, find `slugify` (around line 67):

```rust
pub fn slugify(s: &str) -> String {
    let lower = s.to_lowercase();
    // Replace non-alphanumeric runs with a single `-`.
    let mut out = String::with_capacity(lower.len());
    let mut in_run = false;
    for c in lower.chars() {
        if c.is_ascii_alphanumeric() {
            out.push(c);
            in_run = false;
        } else if !in_run {
            out.push('-');
            in_run = true;
        }
    }
```

Replace with:

```rust
pub fn slugify(s: &str) -> String {
    let lower = s.to_lowercase();
    // Strip quote/apostrophe characters entirely (not replaced with a
    // hyphen) so contractions and quoted phrases don't get spuriously
    // split: "can't" -> "cant", not "can-t". Covers the ASCII and
    // curly-quote forms most likely to show up in pasted text.
    const STRIP_CHARS: &[char] =
        &['\'', '\u{2019}', '"', '\u{201C}', '\u{201D}'];
    let stripped: String =
        lower.chars().filter(|c| !STRIP_CHARS.contains(c)).collect();
    // Replace remaining non-alphanumeric runs with a single `-`.
    let mut out = String::with_capacity(stripped.len());
    let mut in_run = false;
    for c in stripped.chars() {
        if c.is_ascii_alphanumeric() {
            out.push(c);
            in_run = false;
        } else if !in_run {
            out.push('-');
            in_run = true;
        }
    }
```

- [ ] **Step 4: Update the doc comment**

Immediately above `slugify` (around line 52), find:

```rust
/// Convert a free-text insight to a kebab-case slug suitable for use as
/// a memory key. Port of beads' `slugify()` from
/// `reference/beads/cmd/bd/memory.go:23-44`.
///
/// Rules:
/// - Lowercase.
/// - Non-alphanumeric runs collapse to a single `-`.
/// - Strip leading/trailing `-`.
/// - Take only the first 8 hyphen-separated tokens.
/// - Cap total length at 60 chars; trim any trailing `-` left after
///   truncation.
///
/// Returns an empty string if the input contains no alphanumerics
/// (caller's job to surface "could not auto-slug, pass --key" in that
/// case).
```

Replace with:

```rust
/// Convert a free-text insight to a kebab-case slug suitable for use as
/// a memory key. Port of beads' `slugify()` from
/// `reference/beads/cmd/bd/memory.go:23-44`, with one deviation: quote
/// and apostrophe characters are stripped rather than treated as word
/// separators (see Rules below).
///
/// Rules:
/// - Lowercase.
/// - Quote/apostrophe characters (`'` `'` `"` `"` `"`) are stripped
///   entirely — contractions don't get split (`"can't"` -> `"cant"`,
///   not `"can-t"`).
/// - Remaining non-alphanumeric runs collapse to a single `-`.
/// - Strip leading/trailing `-`.
/// - Take only the first 8 hyphen-separated tokens.
/// - Cap total length at 60 chars; trim any trailing `-` left after
///   truncation.
///
/// Returns an empty string if the input contains no alphanumerics
/// (caller's job to surface "could not auto-slug, pass --key" in that
/// case).
```

- [ ] **Step 5: Run the new tests to confirm they pass**

Run: `cargo test -p iss-storage slugify 2>&1 | tail -60`
Expected: all `slugify_*` tests PASS, including the 3 new ones and the 4 pre-existing ones (`slugify_basic`, `slugify_caps_eight_tokens`, `slugify_caps_60_chars`, `slugify_caps_60_chars_with_hyphens` — none of those use apostrophes or quotes, so they're unaffected by this change).

- [ ] **Step 6: Run the full iss-storage test suite**

Run: `cargo test -p iss-storage 2>&1 | tail -80`
Expected: all pass. Check specifically for any `memory.rs` integration test (in `crates/iss/tests/memory.rs`, e.g. `remember_value_with_no_alphanumerics_errors_with_hint` at line 413) that might depend on old slugify behavior with quote characters — read it if it exists and confirm it doesn't assert on apostrophe/quote splitting.

- [ ] **Step 7: Run the full workspace test suite**

Run: `cargo nextest run --workspace 2>&1 | tail -80` (fall back to `cargo test --workspace`).
Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git add crates/iss-storage/src/memory.rs
git commit -m "fix(slugify): strip quote/apostrophe chars instead of splitting on them"
```

---

### Task 9: Rename `jjf`-named test helper functions to `iss`/`git`-named equivalents

**Files:**
- Modify: `crates/iss/tests/common/mod.rs` (source definitions: `make_jj_repo`, `run_jjf`, `run_jjf_with_stdin`, `scratch_non_git`'s doc comment/path segment, `make_initialized_repo`'s internal messages)
- Modify: `crates/iss/tests/actor.rs` (`make_jj_repo_with_user`, `run_jjf_with_env`, `run_jjf_with_stdin_env`)
- Modify: `crates/iss/tests/claim.rs` (`make_jj_repo_with_user`)
- Modify: `crates/iss/tests/comment.rs` (`make_jj_repo`)
- Modify: `crates/iss/tests/concurrent_write.rs` (`make_jj_repo`)
- Modify: `crates/iss/tests/memory.rs` (`run_jjf_stdin`)
- Modify: `crates/iss/tests/stale.rs` (`run_jjf_with_env`, `run_jjf_with_stdin_and_env`)
- Modify: every other file under `crates/iss/tests/*.rs` that calls `run_jjf`/`make_jj_repo` via `use common::*` (call-site renames only, no new definitions) — found via `grep -rl "run_jjf\|make_jj_repo" crates/iss/tests/*.rs`

**Interfaces:**
- Produces: `run_iss`, `run_iss_with_stdin`, `make_git_repo`, `make_git_repo_with_user`, `run_iss_with_env`, `run_iss_with_stdin_env`, `run_iss_stdin`, `run_iss_with_stdin_and_env` (each the direct rename of its `jjf`/`jj`-named counterpart, same signature, same body — pure identifier rename).
- Consumes: nothing from other tasks. Independent of Tasks 1-8's renames (different files, different identifiers) except Task 4, which already touched `crates/iss/tests/init.rs`'s comment referencing `NotAJjRepo` — if `init.rs` also has a local `make_jj_repo`-style helper, rename it here too, in the same pass (no conflict, Task 4's edit was comment-only and already landed).

This is the same identifier-naming bug class as Tasks 3 and 4 (stale `jj`-era names on functions that have been pure-git implementations for a while), just in test-helper code rather than production code. Found during Task 1's implementation when the user noticed test output referencing "jjf". Confirmed via `grep -rn "fn make_jj_repo\|fn run_jjf" crates/iss/tests/` that all such functions already shell out to plain `git`/the `iss` binary — this is a naming-only fix, zero behavior change.

- [ ] **Step 1: Enumerate every definition and call site**

Run:
```bash
grep -rn "fn make_jj_repo\|fn run_jjf" crates/iss/tests/*.rs crates/iss/tests/common/*.rs
grep -rl "run_jjf\|make_jj_repo" crates/iss/tests/*.rs
```

Record the full list — this is your rename map. Expect roughly: `common/mod.rs` (3 definitions: `make_jj_repo`, `run_jjf`, `run_jjf_with_stdin`), plus per-file local definitions in `actor.rs`, `claim.rs`, `comment.rs`, `concurrent_write.rs`, `memory.rs`, `stale.rs` (7 more definitions), plus call-site-only files that `use common::*` and invoke `run_jjf`/`make_jj_repo` without defining their own copy.

- [ ] **Step 2: Rename definitions and call sites in `common/mod.rs`**

In `crates/iss/tests/common/mod.rs`:

- `pub(crate) fn make_jj_repo(name: &str) -> PathBuf` → rename to `make_git_repo`. Update its two `.expect(...)` messages if they mention "jj" (they currently say `"spawn git init"` etc. — already git-flavored, leave as-is; only the fn name and any doc comment change).
- `pub(crate) fn make_initialized_repo(name: &str) -> PathBuf` — this function's *name* is fine (no "jj"), but its body calls `make_jj_repo(name)` (update to `make_git_repo(name)`) and has `.expect("spawn jjf init")` / the assert message `"jjf init in {} failed: {}"` — update both to say `"spawn iss init"` / `"iss init in {} failed: {}"`.
- `pub(crate) fn run_jjf(cwd: &Path, args: &[&str]) -> Output` → rename to `run_iss`. Update `.expect("spawn jjf")` to `.expect("spawn iss")`.
- `pub(crate) fn run_jjf_with_stdin(cwd: &Path, args: &[&str], stdin_bytes: &[u8]) -> Output` → rename to `run_iss_with_stdin`. Update `.expect("spawn jjf")` → `.expect("spawn iss")` and `.expect("wait for jjf")` → `.expect("wait for iss")`.
- `scratch_non_git`'s doc comment says "cannot be inside the jjforge source tree" — change to "the git-issues source tree". Its path segment `std::env::temp_dir().join("jjf-tests")` — rename the literal string `"jjf-tests"` to `"iss-tests"` (this is a scratch-directory name on disk, not a wire token — safe to rename).

- [ ] **Step 3: Rename in `actor.rs`**

In `crates/iss/tests/actor.rs`:
- `fn make_jj_repo_with_user(name: &str, user: Option<&str>) -> PathBuf` → `make_git_repo_with_user`.
- `fn run_jjf_with_env(cwd: &Path, args: &[&str], env: &[(&str, Option<&str>)]) -> Output` → `run_iss_with_env`.
- `fn run_jjf_with_stdin_env(...)` → `run_iss_with_stdin_env`.
- Update every call site within this file that references the old names.
- Update any `.expect("spawn jjf...")`-style messages in these functions' bodies to say `"iss"` instead.

- [ ] **Step 4: Rename in `claim.rs`**

In `crates/iss/tests/claim.rs`:
- `fn make_jj_repo_with_user(name: &str, user: &str) -> PathBuf` → `make_git_repo_with_user`.
- Update call sites within this file.

- [ ] **Step 5: Rename in `comment.rs`**

In `crates/iss/tests/comment.rs`:
- `fn make_jj_repo(name: &str) -> PathBuf` → `make_git_repo`.
- Update call sites within this file.

- [ ] **Step 6: Rename in `concurrent_write.rs`**

In `crates/iss/tests/concurrent_write.rs`:
- `fn make_jj_repo(name: &str) -> PathBuf` → `make_git_repo`.
- Update call sites within this file.

- [ ] **Step 7: Rename in `memory.rs`**

In `crates/iss/tests/memory.rs`:
- `fn run_jjf_stdin(cwd: &Path, args: &[&str], stdin: &str) -> Output` → `run_iss_stdin`.
- Update call sites within this file. (Note: `memory.rs` also has its own local `make_initialized` helper — that name has no "jj" in it, leave it as-is; only rename the `run_jjf_stdin` identifier.)

- [ ] **Step 8: Rename in `stale.rs`**

In `crates/iss/tests/stale.rs`:
- `fn run_jjf_with_env(cwd: &Path, args: &[&str], clock_secs: u64) -> Output` → `run_iss_with_env`.
- `fn run_jjf_with_stdin_and_env(...)` → `run_iss_with_stdin_and_env`.
- Update call sites within this file.

- [ ] **Step 9: Update call sites in every remaining file**

For every other file `grep -rl "run_jjf\|make_jj_repo" crates/iss/tests/*.rs` found in Step 1 that does NOT define its own copy (i.e. imports via `use common::*`), do a straightforward rename of each call: `run_jjf(` → `run_iss(`, `run_jjf_with_stdin(` → `run_iss_with_stdin(`, `make_jj_repo(` → `make_git_repo(`. Use `grep -n` per file to confirm you've caught every call site before moving to the next file.

- [ ] **Step 10: Full-tree verification grep**

Run: `grep -rn "run_jjf\|make_jj_repo\b" crates/iss/tests/`
Expected: no output. (Note: this grep intentionally does NOT match `make_jj_repo_with_user` variants if you already renamed those in Steps 3-4 — if it does match, you missed one.)

Run: `grep -rn "make_jj_repo_with_user\|run_jjf_with_env\|run_jjf_with_stdin_env\|run_jjf_stdin\|run_jjf_with_stdin_and_env" crates/iss/tests/`
Expected: no output.

- [ ] **Step 11: Build the test binaries**

Run: `cargo build --workspace --tests 2>&1 | tail -60`
Expected: clean build. Any unresolved-name error points at a call site you missed — fix and rebuild.

- [ ] **Step 12: Run the full workspace test suite**

Run: `cargo nextest run --workspace 2>&1 | tail -100` (fall back to `cargo test --workspace` if nextest isn't installed).
Expected: same pass count as before this task's changes (pure rename, zero behavior change — no test should newly pass or newly fail).

- [ ] **Step 13: Commit**

```bash
git add crates/iss/tests/
git commit -m "refactor(tests): rename jjf-named test helpers to iss/git-named equivalents"
```

---

## Final Verification

- [ ] **Step 1: Full workspace build + test**

Run:
```bash
cargo build --workspace 2>&1 | tail -30
cargo nextest run --workspace 2>&1 | tail -100
```
Expected: clean build, all tests pass.

- [ ] **Step 2: Confirm no stale jj references remain in touched files**

Run: `grep -rn "jj" docs/quickstart.md crates/iss/src/main.rs crates/iss/src/preflight.rs crates/iss-storage/src/lib.rs | grep -vi "refs/jjf\|Jjf-\|jjf-storage\|jjf-merge"`
Expected: no output (or only pre-existing, out-of-scope mentions not touched by this plan — review any hit against the spec's Track A scope before deciding it's fine to leave).

- [ ] **Step 3: Live smoke test in a scratch repo**

```bash
mkdir -p /tmp/iss-plan-verify && cd /tmp/iss-plan-verify && rm -rf repo && mkdir repo && cd repo
git init
/Users/myers/p/git-issues/target/debug/iss init
/Users/myers/p/git-issues/target/debug/iss new -t "bare test issue" -F /dev/null
/Users/myers/p/git-issues/target/debug/iss show $(cd /tmp/iss-plan-verify/repo && /Users/myers/p/git-issues/target/debug/iss ls --json | python3 -c "import json,sys; print(json.load(sys.stdin)[0]['id'])")
/Users/myers/p/git-issues/target/debug/iss remember "it's a test, can't skip it"
/Users/myers/p/git-issues/target/debug/iss memories
```

Expected: `iss show`'s output has no `labels:`/`priority:`/`assignee:`/`dependencies:` lines (bare issue, all empty); `iss memories` shows one line, tab-separated, key roughly `its-a-test-cant-skip-it` (no `-s-`/`-t-` fragments from the stripped apostrophes).

Clean up: `rm -rf /tmp/iss-plan-verify`

- [ ] **Step 4: Push**

Per this repo's CLAUDE.md convention (push at the end of every issue/unit of work):

```bash
cd /Users/myers/p/git-issues
git push origin main
```
