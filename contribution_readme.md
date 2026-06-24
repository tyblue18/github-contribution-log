# Contribution [#1]: uv errors if the `pyproject.toml` is a directory

**Contribution Number:** 1  
**Student:** [Tanishq Somani
**Issue:** (https://github.com/astral-sh/uv/issues/14584)
**Status:** Phase I

---

## Why I Chose This Issue

Since uv is one of the most wiedely adopted Python tooling projexts right now, so contributing here means working in a fast-moving, high standar codebase that thousands of developers depend on daily. I picked this issue because the scope is well-bounded and a core maintainer has already stated the intended fix direction in the thread, which lets me focus on learning the codebase rather than guessing at design. It also sits in the error-handling and filesystem layer, which is a clean entry point into a large project without needing to understand the resolver or type system first.
My background is mostly in JavaScript, SQL, TypeScript, and Python, so this is a deliberate stretch into Rust. My goal is to learn how a production Rust CLI structures error handling, how it distinguishes fatal from non-fatal conditions, and how its testing harness works, while shipping a real, user-facing improvement. The fact that the bug currently blocks unrelated commands (like shell autocompletion) makes the impact concrete and motivating.

---

## Understanding the Issue

### Problem Description

When uv resolves a configuration file, it expects pyproject.toml to be a regular file. If a directory named pyproject.toml exists in the current working directory or a parent directory, uv fails to read it and aborts with a low-level OS error. This blocks essentially every uv command, even ones that do not need the config file at all.

### Expected Behavior

The maintainers have specified the intended behavior in the issue thread:

If a pyproject.toml directory is found in the current working directory, uv should error with a clear, explanatory message (not a raw OS error).
If it is found in a parent directory, erroring is too aggressive; uv should treat it as non-fatal and continue (for example, autocompletion should ignore it since it does not need the config).

### Current Behavior

Any uv command ran with a directory named pyproject.toml (or uv.toml) anywhere between the current directory and the filesystem root crashed with a raw OS error:
```
error: failed to read from file `/pyproject.toml`: Is a directory (os error 21)
```
### Affected Components

uv aborts on any command with:
error: failed to read from file `/pyproject.toml`: Is a directory (os error 21)
Adding -v does not surface any additional helpful context. Because the failure is fatal, downstream actions such as autocompletion cannot run.
Affected Components

The configuration-file discovery/loading logic (the code that searches the working directory and parent directories for pyproject.toml / uv.toml).
The error types and messaging for config reads.
Likely the settings/workspace resolution path that calls into config loading.
[Confirm exact crates/modules after cloning — note them here, e.g. crates/uv-settings, crates/uv-workspace.]

---

## Reproduction Process

### Environment Setup

My machine is Windows, but the issue was reported on Linux (Pop!_OS) and the error is a Linux OS error (os error 21), so I set up a Linux development environment using WSL2 to match the reporter's conditions. Challenges I hit and how I solved them:

-Rust installer failed in PowerShell. The standard rustup command (curl ... | sh) is a Linux/macOS shell script; PowerShell has no sh. Fix: installed WSL2 instead of using native Windows Rust, since the bug is Linux-flavored.
-wsl --install didn't install a distribution. After rebooting, wsl reported "no installed distributions." Fix: explicitly installed Ubuntu with wsl --install -d Ubuntu, then created my UNIX user.
-Ran Linux commands in PowerShell by mistake. Commands like sudo apt update && ... failed with The token '&&' is not a valid statement separator. Fix: learned to enter the Ubuntu shell first with wsl — the prompt changes from PS C:\...> to user@machine:~$.
-npm install -g permission error (EACCES) when installing tooling. Ubuntu's apt-packaged npm tries to write to system directories. Fix: installed Node via nvm (Node Version Manager), which keeps global packages in the home directory — no sudo needed.
-Git identity not set inside WSL. First commit failed with "Author identity unknown" — I had configured git in a different (Windows) terminal, but WSL is a separate environment. Fix: ran git config --global user.name/user.email inside WSL, using the email linked to my GitHub account.
-GitHub password authentication rejected on push. GitHub no longer accepts account passwords for git operations. Fix: installed GitHub CLI (sudo apt install gh), ran gh auth login with browser-based authentication.
-Build tooling: installed build-essential, ripgrep (code search), cargo-insta (snapshot test management). First cargo build of uv took ~10–15 minutes (large Rust workspace) — subsequent builds are incremental and fast. I kept the repository in the Linux filesystem (~/uv) rather than /mnt/c/... because builds on the Windows-mounted filesystem are dramatically slower.

### Steps to Reproduce


---

**1. Clone and build `uv` from source**

```bash
git clone https://github.com/<your-username>/uv.git
cd uv
cargo build
```

**2. Create the trap — a directory named `pyproject.toml` with a project beneath it**

```bash
mkdir -p /tmp/repro/pyproject.toml
mkdir -p /tmp/repro/myproject
cd /tmp/repro/myproject
```

**3. Run any `uv` project command from the subdirectory**

```bash
~/uv/target/debug/uv init --name demo .
~/uv/target/debug/uv lock
```

**Observed (bug):** Both commands fail immediately with:
```
error: failed to read from file `/tmp/repro/pyproject.toml`: Is a directory (os error 21)
```

**Expected:** `uv` should ignore a directory named `pyproject.toml` in a parent directory and continue normally.

---

**4. Second scenario — directory is the current working directory itself**

```bash
mkdir -p /tmp/repro2/pyproject.toml
cd /tmp/repro2
~/uv/target/debug/uv lock
```

**Observed:** Same raw OS error.

**Expected (per maintainer):** Still an error, but a clear diagnostic explaining that `pyproject.toml` is a directory rather than a file.

### Reproduction Evidence
**Bug summary:** A directory (not a file) named `pyproject.toml` anywhere between the current directory and the filesystem root crashes every `uv` command. `uv`'s discovery walks parent directories and tries to read it as a file.

---

**5. Confirmed reproducible** across multiple runs and commands (`uv lock`, `uv init`) on current `main` (`uv 0.11.x`), verifying the issue originally reported in `uv 0.7.20` persists.

branch link: https://github.com/tyblue18/uv
---

### Solution Approach 
---

**Understand**

`uv` discovers project/configuration files by checking the current directory for `pyproject.toml`, then each parent directory up to `/`. The discovery code assumes anything named `pyproject.toml` is a readable file; a directory with that name causes a raw filesystem error that aborts every command.

The maintainer (`zanieb`) specified the desired behavior in the issue thread:
1. A directory-`pyproject.toml` in a **parent directory** should be silently skipped (non-fatal).
2. In the **current working directory** it should still error, but with a clear explanation instead of `os error 21`.

---

**Match**

The codebase already contains the correct pattern: workspace discovery in `crates/uv-workspace/src/workspace.rs` uses `.ancestors().find(|p| p.join("pyproject.toml").is_file())` — `is_file()` returns `false` for directories, so that walk never crashes.

The crash comes from a different walk: settings discovery in `crates/uv-settings/src/lib.rs` (`FilesystemOptions::from_directory`), which reads the file directly and only special-cases the `NotFound` error. This was confirmed with a discriminating test: a valid `pyproject.toml` file in cwd + a directory-`pyproject.toml` in a parent still crashed — workspace discovery would have stopped at the valid file, so the crash had to be settings, which walks independently and runs first during CLI config resolution.

For the error-message half, the pattern to match is the `WorkspaceErrorKind` enum (`thiserror`-based, paths rendered with `simplified_display()`).

---

**Plan**

**`crates/uv-settings/src/lib.rs`** — add one match arm in `from_directory`'s `pyproject.toml` read:
```rust
Err(_) if path.is_dir() => {}
```
Treats a directory like a missing file (skip, continue the walk). This alone fixes the crash and delivers behavior (1). Using `path.is_dir()` rather than matching `ErrorKind::IsADirectory` because that error kind is platform-inconsistent and unused elsewhere in the repo.

**`crates/uv-workspace/src/workspace.rs`** — deliver behavior (2):
- A new error variant `PyprojectTomlIsDirectory(PathBuf)` styled like its neighbors.
- A small private helper `check_pyproject_is_not_directory(path)` called at the top of the three discovery entry points (`Workspace::discover`, `ProjectWorkspace::discover`, `VirtualProject::discover`).

The guard runs on the starting path **before** the ancestor walk, because the walk's `is_file()` filter silently discards the directory and would otherwise produce a misleading "No `pyproject.toml` found" message. Guarding all three entry points keeps the error consistent across commands (`uv init`, `uv lock`, `uv run`, `uv sync`). A helper is used because the three functions don't share a common pre-walk code path.

**Deliberately out of scope:**
- The identical bug pattern for a directory named `uv.toml` (same function) — flagged in the PR as a possible follow-up to keep the change minimal.
- `uv init`'s pre-existing "already initialized" check, which is a different, non-crashing code path.

**Files touched:** `crates/uv-settings/src/lib.rs` (+2 lines), `crates/uv-workspace/src/workspace.rs` (~+18 lines), `crates/uv/tests/it/lock.rs` (new tests).

---

**Implement**

See linked branch (same as Reproduction Evidence).

---

**Review**

Per `CONTRIBUTING.md`: `cargo fmt --all`, `cargo clippy` with warnings as errors, tests via the standard harness.

Self-review checklist:
- Minimal additive diff only — no refactoring or drive-by changes.
- Error message wording/format copied from neighboring `WorkspaceErrorKind` variants.
- Helper function placed at the bottom of the file where other free functions live.
- Test doc comments link the issue using the file's existing `/// See: <url>` convention.
- PR description quotes the maintainer's comment as the design rationale and includes before/after output and `Closes #14584`.

---

**Evaluate**

Two integration snapshot tests (`insta` framework) in `crates/uv/tests/it/lock.rs`:
- Directory-`pyproject.toml` in a parent of a valid project → `uv lock` succeeds.
- Directory-`pyproject.toml` in cwd → fails with the new clear error.

Snapshots use the harness's path filters (`[TEMP_DIR]`) so they're machine, and OS-independent.

Additional verification:
- **Mutation check:** temporarily disable the fix, confirm the test fails, restore and confirm green.
- **Manual repro** of both scenarios with the freshly built binary, plus a symlink sanity check (`pyproject.toml` symlinked to a real file must keep working, since `is_file()`/`is_dir()` follow symlinks).
- **Regression slice:** existing workspace-discovery integration tests (~40) and unit tests on the touched crates.
- **Quality gates:** `cargo fmt --all --check` and `cargo clippy -p uv-workspace -p uv-settings --all-targets -- -D warnings`, both clean.


## Testing Strategy

### Unit Tests

uv's discovery logic is primarily covered by integration tests rather than crate-level unit tests, so no new unit tests were added. The existing unit tests on the two modified crates (`uv-settings`, `uv-workspace`) were run to confirm no regressions:

- **Test case 1:** Existing `uv-workspace` library unit tests — confirm the discovery functions and error types still compile and behave correctly after adding the new `PyprojectTomlIsDirectory` variant and the `check_pyproject_is_not_directory` helper. All pass.
- **Test case 2:** Existing `uv-settings` library unit tests — confirm config resolution still works after adding the new directory-skip match arm in `FilesystemOptions::from_directory`. All pass.
- **Test case 3:** Existing ~40 workspace-discovery integration tests (the suite that exercises the exact `discover` functions I modified) — run as a regression slice to confirm normal project/workspace discovery is unaffected. All pass.

### Integration Tests

Two new snapshot-based integration tests were added in `crates/uv/tests/it/lock.rs` (uv uses the `insta` framework: each test runs a real uv command in a sandboxed temp directory and asserts the exact output against a recorded snapshot, with machine-specific values like temp paths normalized to placeholders such as `[TEMP_DIR]` so the tests are OS-independent).

- **Integration scenario 1 — `lock_pyproject_toml_directory_in_parent`:** Creates a *directory* named `pyproject.toml` in a parent directory, plus a valid project in a subdirectory, then runs `uv lock` from the subdirectory. Asserts **success** (exit code 0, normal lock output) — i.e. the directory in the parent is silently skipped. Verifies behavior #1 (non-fatal).
- **Integration scenario 2 — `lock_pyproject_toml_directory_in_cwd`:** Creates a *directory* named `pyproject.toml` in the current directory and runs `uv lock`. Asserts **failure** (exit code 2) with the new clear error: `error: ` `pyproject.toml` ` at [TEMP_DIR]/pyproject.toml is a directory, expected a file`. Verifies behavior #2 (clear error).

### Manual Testing

Built the binary and verified each scenario by hand against the original bug:

- Parent directory-`pyproject.toml` + valid project in a subdirectory → `Resolved 1 package in 51ms` (succeeds; **previously crashed** with os error 21).
- Parent directory-`pyproject.toml` with no project anywhere → normal `No pyproject.toml found in current directory or any parent directory` (graceful; **previously crashed**).
- Current-directory directory-`pyproject.toml` → the new clear error message (**previously** the raw os error 21).
- **Symlink sanity check:** a `pyproject.toml` symlink pointing at a real TOML file → still locks correctly (confirms `is_file()`/`is_dir()` follow symlinks and the fix doesn't break valid setups).
- **Mutation check (proving the tests have teeth):** temporarily disabled the fix (stubbed an early `return Ok(())` into the helper) and re-ran the tests — the cwd test failed with exactly the misleading "not found" output the fix prevents, while the parent test still passed (protected by the independent settings fix). Restoring the fix returned everything to green.

**Quality gates:** `cargo fmt --all --check` and `cargo clippy -p uv-workspace -p uv-settings --all-targets -- -D warnings` both clean. All CI checks (Linux, macOS, Windows) passed on the submitted PR.

---

## Implementation Notes

### Week 1 Progress

Completed the full contribution end-to-end in the first week (the program encourages working ahead).

- **Environment setup:** Set up a WSL2 Ubuntu environment on Windows to match the issue's Linux conditions, working through a chain of toolchain issues (PowerShell vs. shell scripts, WSL distro install, npm permissions → nvm, git identity inside WSL, GitHub CLI auth). All documented under Environment Setup above.
- **Reproduction:** Reproduced the crash consistently on current `main` (uv 0.11.x), confirming the originally reported bug (uv 0.7.20) still exists.
- **Root-cause investigation (key decision point):** My first assumption was that the bug lived in workspace discovery, but that code already used `is_file()` and handled directories correctly. I designed a *discriminating test* — a valid `pyproject.toml` file in the cwd plus a directory-`pyproject.toml` in a parent — and it still crashed, which proved the crash was in a different, earlier layer: **settings discovery** (`FilesystemOptions::from_directory`), which runs first during CLI config resolution. That explained why *every* command crashed, even ones that don't need a project.
- **Implementation:** Two changes — a directory-skip match arm in settings (fixes the crash, makes the parent case non-fatal) and a new clear error message in workspace discovery for the cwd case, applied via a shared helper across all three discovery entry points for consistency.
- **Testing & submission:** Added the two integration tests, ran the mutation check, verified manually, passed the quality gates, and opened the PR. All CI passed; a reviewer self-requested within 24 hours.

### Week 2 Progress

_Awaiting maintainer review. Will document any requested changes and how I addressed them here._

### Code Changes

- **Files modified:**
  - `crates/uv-settings/src/lib.rs` (+2 lines) — directory-skip match arm in `FilesystemOptions::from_directory`.
  - `crates/uv-workspace/src/workspace.rs` (+18 lines) — new `PyprojectTomlIsDirectory` error variant, the `check_pyproject_is_not_directory` helper, and three call sites in `Workspace::discover`, `ProjectWorkspace::discover`, and `VirtualProject::discover`.
  - `crates/uv/tests/it/lock.rs` (+61 lines) — two integration tests.
  - Total: 3 files, +81 / −0 (pure additions, no deletions, nothing touched outside the fix and its tests).
- **Key commits:** [`d99dc92`](https://github.com/astral-sh/uv/pull/19762/commits/d99dc9282b6d2cb074946dac86949f2cd98c2d98) — Ignore directories named pyproject.toml during settings and workspace discovery
- **Approach decisions:**
  - Used `path.is_dir()` rather than matching `ErrorKind::IsADirectory`, because that error kind is platform-inconsistent (Windows reports directory-read failures differently) and unused elsewhere in the repo.
  - Put the guard *before* the ancestor walk, because the walk's `is_file()` filter silently discards the directory — by the time it runs, the information needed for a clear error is gone.
  - Applied the guard to all three discovery entry points via a shared helper (rather than just the path the repro hit), so the error is consistent across `uv lock`, `uv init`, `uv run`, `uv sync`, etc.
  - Deliberately scoped out the identical bug for a directory named `uv.toml` (same function) to keep the PR minimal, flagging it as a possible follow-up.

---

## Pull Request

**PR Link:** https://github.com/astral-sh/uv/pull/19762

**PR Description (final, as submitted):**

> **Summary** — A directory named `pyproject.toml` anywhere on the path from the current directory up to `/` caused every uv command to crash with `error: failed to read from file ...: Is a directory (os error 21)`. Per @zanieb's spec on the issue, the fix makes two cases work: a directory-`pyproject.toml` in a **parent** directory is silently skipped (non-fatal), and one in the **current directory** produces a clear error instead of a raw OS error.
>
> **Root cause** — The crash was in settings discovery (`FilesystemOptions::from_directory` in `uv-settings`), which only special-cased `NotFound`; workspace discovery already used `is_file()`. Settings discovery runs before everything else, so every command crashed.
>
> **The two changes** — `uv-settings`: a match arm that skips the path when it's a directory. `uv-workspace`: a new `PyprojectTomlIsDirectory` error plus a shared helper called at the top of all three discover functions, so the cwd case errors clearly and consistently.
>
> Includes before/after output, two integration tests, and `Closes #14584`.

**Maintainer Feedback:**

- _<date>_: Reviewer EliteTK self-requested a review within 24 hours of submission. Awaiting feedback.
- _<date>_: _How I addressed it — to be filled in._

**Status:** Awaiting review (all CI checks passing; reviewer self-requested)
---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
