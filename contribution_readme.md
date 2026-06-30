# Contribution #1: uv errors if the `pyproject.toml` is a directory

**Contribution Number:** 1
**Student:** Tanishq Somani
**Issue:** [astral-sh/uv#14584](https://github.com/astral-sh/uv/issues/14584)
**Pull Request:** [astral-sh/uv#19762](https://github.com/astral-sh/uv/pull/19762)
**Status:** Iterating (reworked per maintainer review; awaiting response)

---

## Why I Chose This Issue

uv is one of the most widely adopted Python tooling projects right now, so contributing here means working in a fast-moving, high-standard codebase that thousands of developers depend on daily. I picked this issue because the scope is well-bounded and a core maintainer had already stated the intended fix direction in the thread, which let me focus on learning the codebase rather than guessing at design. It also sits in the error-handling and filesystem layer, which is a clean entry point into a large project without needing to understand the resolver or type system first.

My background is mostly in JavaScript, SQL, TypeScript, and Python, so this was a deliberate stretch into Rust. My goal was to learn how a production Rust CLI structures error handling, how it distinguishes fatal from non-fatal conditions, and how its testing harness works, while shipping a real, user-facing improvement. The fact that the bug currently blocks unrelated commands (like shell autocompletion) made the impact concrete and motivating.

---

## Understanding the Issue

### Problem Description

When uv resolves a configuration file, it expects `pyproject.toml` to be a regular file. If a directory named `pyproject.toml` exists in the current working directory or a parent directory, uv fails to read it and aborts with a low-level OS error. This blocks essentially every uv command, even ones that do not need the config file at all.

### Expected Behavior

The maintainers specified the intended behavior in the issue thread:

> If a `pyproject.toml` directory is found in the current working directory, uv should error with a clear, explanatory message (not a raw OS error). If it is found in a parent directory, erroring is too aggressive; uv should treat it as non-fatal and continue (for example, autocompletion should ignore it since it does not need the config).

### Current Behavior (before the fix)

Any uv command run with a directory named `pyproject.toml` (or `uv.toml`) anywhere between the current directory and the filesystem root crashed with a raw OS error:

```
error: failed to read from file `/pyproject.toml`: Is a directory (os error 21)
```

Adding `-v` did not surface any additional helpful context. Because the failure is fatal, downstream actions such as autocompletion could not run.

### Affected Components

- The configuration-file discovery/loading logic (the code that searches the working directory and parent directories for `pyproject.toml` / `uv.toml`).
- The error types and messaging for config reads.
- The settings resolution path that calls into config loading (`crates/uv-settings`).

---

## Reproduction Process

### Environment Setup

My machine is Windows, but the issue was reported on Linux (Pop!_OS) and the error is a Linux OS error (`os error 21`), so I set up a Linux development environment using WSL2 to match the reporter's conditions. Challenges I hit and how I solved them:

1. **Rust installer failed in PowerShell.** The standard rustup command (`curl ... | sh`) is a Linux/macOS shell script; PowerShell has no `sh`. **Fix:** installed WSL2 instead of using native Windows Rust, since the bug is Linux-flavored.
2. **`wsl --install` didn't install a distribution.** After rebooting, `wsl` reported "no installed distributions." **Fix:** explicitly installed Ubuntu with `wsl --install -d Ubuntu`, then created my UNIX user.
3. **Ran Linux commands in PowerShell by mistake.** Commands like `sudo apt update && ...` failed with `The token '&&' is not a valid statement separator`. **Fix:** learned to enter the Ubuntu shell first with `wsl` — the prompt changes from `PS C:\...>` to `user@machine:~$`.
4. **`npm install -g` permission error (EACCES) when installing tooling.** Ubuntu's apt-packaged npm tries to write to system directories. **Fix:** installed Node via `nvm` (Node Version Manager), which keeps global packages in the home directory — no `sudo` needed.
5. **Git identity not set inside WSL.** First commit failed with "Author identity unknown" — I had configured git in a different (Windows) terminal, but WSL is a separate environment. **Fix:** ran `git config --global user.name/user.email` inside WSL, using the email linked to my GitHub account.
6. **GitHub password authentication rejected on push.** GitHub no longer accepts account passwords for git operations. **Fix:** installed GitHub CLI (`sudo apt install gh`), ran `gh auth login` with browser-based authentication.
7. **Build tooling:** installed `build-essential`, `ripgrep` (code search), `cargo-insta` (snapshot test management). First `cargo build` of uv took ~10–15 minutes (large Rust workspace) — subsequent builds are incremental and fast. I kept the repository in the Linux filesystem (`~/uv`) rather than `/mnt/c/...` because builds on the Windows-mounted filesystem are dramatically slower.

### Steps to Reproduce

**1. Clone and build uv from source**

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

**3. Run any uv project command from the subdirectory**

```bash
~/uv/target/debug/uv init --name demo .
~/uv/target/debug/uv lock
```

**Observed (bug):** both commands fail immediately with:

```
error: failed to read from file `/tmp/repro/pyproject.toml`: Is a directory (os error 21)
```

**Expected:** uv should ignore a directory named `pyproject.toml` in a parent directory and continue normally.

**4. Second scenario — directory is the current working directory itself**

```bash
mkdir -p /tmp/repro2/pyproject.toml
cd /tmp/repro2
~/uv/target/debug/uv lock
```

**Observed:** same raw OS error.
**Expected (per maintainer):** still an error, but a clear diagnostic explaining that `pyproject.toml` is a directory rather than a file.

### Reproduction Evidence

**Bug summary:** A directory (not a file) named `pyproject.toml` anywhere between the current directory and the filesystem root crashes every uv command. uv's discovery walks parent directories and tries to read it as a file.

Confirmed reproducible across multiple runs and commands (`uv lock`, `uv init`) on current `main` (uv 0.11.x), verifying the issue originally reported in uv 0.7.20 persists.

**Branch link:** https://github.com/tyblue18/uv/tree/fix-pyproject-dir-discovery

---

## Solution Approach

> _Note: this section records my initial plan, written during Phase II before maintainer review. The approach changed substantially after review — see Week 2 Progress below for the final design._

### Understand

uv discovers project/configuration files by checking the current directory for `pyproject.toml`, then each parent directory up to `/`. The discovery code assumed anything named `pyproject.toml` is a readable file; a directory with that name caused a raw filesystem error that aborted every command.

The maintainer (zanieb) specified the desired behavior in the issue thread:

- A directory-`pyproject.toml` in a **parent** directory should be silently skipped (non-fatal).
- In the **current working directory** it should still error, but with a clear explanation instead of `os error 21`.

### Match

The codebase already contained the correct pattern: workspace discovery in `crates/uv-workspace/src/workspace.rs` uses `.ancestors().find(|p| p.join("pyproject.toml").is_file())` — `is_file()` returns false for directories, so that walk never crashes.

The crash came from a different walk: settings discovery in `crates/uv-settings/src/lib.rs` (`FilesystemOptions::from_directory`), which reads the file directly and only special-cased the `NotFound` error. I confirmed this with a discriminating test: a valid `pyproject.toml` file in cwd + a directory-`pyproject.toml` in a parent still crashed — workspace discovery would have stopped at the valid file, so the crash had to be settings, which walks independently and runs first during CLI config resolution.

### Plan (initial)

1. `crates/uv-settings/src/lib.rs` — add a match arm in `from_directory`'s `pyproject.toml` read to treat a directory like a missing file (skip, continue the walk). Using `path.is_dir()` rather than matching `ErrorKind::IsADirectory` because that error kind is platform-inconsistent and unused elsewhere in the repo.
2. `crates/uv-workspace/src/workspace.rs` — a new error variant plus a small private helper called at the top of the three discovery entry points to deliver the clear cwd error.

**Deliberately out of scope (initial plan):** the identical bug pattern for a directory named `uv.toml` (flagged as a possible follow-up), and `uv init`'s pre-existing "already initialized" check.

### Review

Per `CONTRIBUTING.md`: `cargo fmt --all`, `cargo clippy` with warnings as errors, tests via the standard harness. Self-review checklist: minimal additive diff, error-message wording matching neighboring variants, test doc comments linking the issue, and a PR description quoting the maintainer's comment with before/after output and `Closes #14584`.

### Evaluate

Integration snapshot tests (insta framework), a mutation check (disable the fix, confirm the test fails, restore), manual repro of both scenarios, a symlink sanity check, a regression slice of existing discovery tests, and the `cargo fmt` / `cargo clippy` quality gates.

---

## Testing Strategy

### Unit Tests

uv's discovery logic is primarily covered by integration tests rather than crate-level unit tests, so no new unit tests were added. The existing unit tests on the modified crate (`uv-settings`) and the related `uv-workspace` crate were run to confirm no regressions:

- **Test case 1:** Existing `uv-settings` library unit tests — confirm config resolution still works after adding the new `Error::Directory` handling in `FilesystemOptions::from_directory` / `find`. All pass.
- **Test case 2:** Existing `uv-workspace` library unit tests — confirm workspace discovery is unaffected (the original `uv-workspace` change was reverted during review). All pass.
- **Test case 3:** Existing workspace-discovery integration tests, run as a regression slice to confirm normal project/workspace discovery is unaffected. All pass.

### Integration Tests

Five snapshot-based integration tests using the insta framework (each runs a real uv command in a sandboxed temp directory and asserts the exact output against a recorded snapshot, with machine-specific values like temp paths normalized to placeholders such as `[TEMP_DIR]` so the tests are OS-independent):

- **`lock_pyproject_toml_directory_in_parent` / `lock_uv_toml_directory_in_parent`:** directory named `pyproject.toml` / `uv.toml` in a parent of a valid project → `uv lock` succeeds (parent case is non-fatal).
- **`lock_pyproject_toml_directory_in_cwd`:** directory named `pyproject.toml` in the cwd → clear error.
- **`lock_uv_toml_directory_in_cwd`:** directory named `uv.toml` in the cwd of an otherwise valid project → clear error.
- **`create_venv_pyproject_toml_directory_in_cwd`** (in `venv.rs`): confirms the error also fires for commands that don't require a project.

### Manual Testing

Built the binary and verified each scenario by hand against the original bug:

- Parent directory-`pyproject.toml` (or `uv.toml`) + valid project in a subdirectory → `Resolved 1 package` (succeeds; **previously crashed** with os error 21). With `-v`, a `WARN Ignoring ... during settings discovery: expected a file, found a directory` is emitted.
- Current-directory directory-`pyproject.toml` → the new clear error (**previously** the raw os error 21).
- Current-directory directory-`uv.toml` → the new clear error (the adjacent bug, now also fixed).
- **Symlink sanity check:** a `pyproject.toml` symlink pointing at a real TOML file → still locks correctly (confirms `is_dir()` follows symlinks and the fix doesn't break valid setups).
- `uv venv --no-config` with a `pyproject.toml` directory in the cwd → still succeeds, as on `main`.
- **Mutation check (proving the tests have teeth):** temporarily disabled the fix and re-ran the tests — the affected test failed with the pre-fix behavior, then restoring the fix returned everything to green, confirming each test guards its corresponding behavior.

**Quality gates:** `cargo fmt --all --check` and `cargo clippy -p uv-workspace -p uv-settings --all-targets -- -D warnings` both clean. All CI checks (Linux, macOS, Windows) passed on the submitted PR.

---

## Implementation Notes

### Week 1 Progress

Completed the initial contribution end-to-end in the first week (the program encourages working ahead).

- **Environment setup:** Set up a WSL2 Ubuntu environment on Windows to match the issue's Linux conditions, working through a chain of toolchain issues (PowerShell vs. shell scripts, WSL distro install, npm permissions → nvm, git identity inside WSL, GitHub CLI auth). All documented under Environment Setup above.
- **Reproduction:** Reproduced the crash consistently on current `main` (uv 0.11.x), confirming the originally reported bug (uv 0.7.20) still exists.
- **Root-cause investigation (key decision point):** My first assumption was that the bug lived in workspace discovery, but that code already used `is_file()` and handled directories correctly. I designed a *discriminating test* — a valid `pyproject.toml` file in the cwd plus a directory-`pyproject.toml` in a parent — and it still crashed, which proved the crash was in a different, earlier layer: settings discovery (`FilesystemOptions::from_directory`), which runs first during CLI config resolution. That explained why *every* command crashed, even ones that don't need a project.
- **Initial implementation:** Two changes — a directory-skip arm in `uv-settings` and a new clear error in `uv-workspace` discovery for the cwd case, applied via a shared helper. Added integration tests, ran the mutation check, verified manually, passed the quality gates, and opened the PR. All CI passed; a reviewer self-requested within 24 hours.

### Week 2 Progress — Maintainer Review & Redesign

Maintainer review (EliteTK) led to a significant redesign — the fix moved out of `uv-workspace` and entirely into `uv-settings`, which is the correct layer.

- **Review feedback:** EliteTK raised that the discovery functions in `uv-workspace` were the wrong place for the check — they aren't guaranteed to be called only on the CWD, their errors are suppressed in several cases, and the check was technically a breaking change. He also noted the original approach lost error coverage for the adjacent `uv.toml`-in-cwd case, and pointed to `FilesystemOptions::find` as the correct location: propagate the directory error *if and only if* it occurs at the top level of the search, and warn (silent by default) otherwise. Smaller comments: an unnecessary code comment should be removed, and a warning should be added when skipping in parent directories (torn between `warn_user_once!` and `warn!`, he suggested `warn!` since the directory may be in a location the user can't manage).
- **How I addressed it:** Reverted all `uv-workspace` changes. `from_directory` now returns a dedicated `Error::Directory` for both `uv.toml` and `pyproject.toml` instead of failing with the raw IO error. `FilesystemOptions::find` propagates that error only when it occurs at the top level of the search; for parent directories it emits a `warn!` (silent by default) and continues walking up. Because this runs during config resolution, the top-level error now surfaces for *all* commands — including ones that don't do project discovery, like `uv venv` and `uv pip` — and the same handling covers `uv.toml`. Added a `uv venv` integration test to prove the error fires for non-project commands. Force-pushed as `f99814d`.
- **Open question I raised with the maintainer:** a `pyproject.toml` directory in the CWD when a valid project exists in a *parent* is still silent — workspace discovery succeeds and the settings search starts at the workspace root, so the CWD is never visited. This seemed consistent with "error only at the top level of the search," but I flagged that I'm happy to add a warning there if preferred.
- **Outcome:** A simpler, more correct fix living entirely in one crate, with no breaking change and broader command coverage than my original approach.

### Code Changes

- **Files modified (final, post-review):**
  - `crates/uv-settings/src/lib.rs` — new `Error::Directory` variant returned by `from_directory` for a directory named `uv.toml` or `pyproject.toml`; `FilesystemOptions::find` propagates it only at the top level of the search and emits a `warn!` for parent directories.
  - `crates/uv/tests/it/lock.rs` — four integration tests.
  - `crates/uv/tests/it/venv.rs` — one integration test (non-project command coverage).
  - The original `crates/uv-workspace/src/workspace.rs` changes were **reverted** per review.
- **Key commits:**
  - [`d99dc92`](https://github.com/astral-sh/uv/pull/19762/commits/d99dc9282b6d2cb074946dac86949f2cd98c2d98) — initial approach (uv-settings skip arm + uv-workspace guard)
  - [`f99814d`](https://github.com/astral-sh/uv/pull/19762/commits/f99814d) — reworked per review: directory handling moved entirely into `FilesystemOptions::find`, with top-level-only propagation and `warn!` for parents, plus `uv.toml` coverage and a `uv venv` test
- **Approach decisions (final):**
  - Placed the fix in `FilesystemOptions::find` rather than the discovery functions, because `find` is the single point that runs during config resolution before any command logic — so the error covers every command, including non-project ones — and it avoids the breaking-change and error-suppression problems of the discovery path.
  - Propagate the directory error **only at the top level** of the search; for parent directories, `warn!` (silent by default) and continue, matching the maintainer's spec that erroring on a parent is too aggressive.
  - Extended the same handling to `uv.toml`, fixing the adjacent bug in the same change.
  - Used `is_dir()` (which follows symlinks) so a `pyproject.toml` symlink to a real file still works.

---

## Pull Request

**PR Link:** [astral-sh/uv#19762](https://github.com/astral-sh/uv/pull/19762)

**PR Description (final, as submitted after rework):**

> **Summary** — Any uv command run with a directory named `pyproject.toml` (or `uv.toml`) anywhere between the current directory and the filesystem root crashed with a raw OS error. The crash site is settings discovery: `FilesystemOptions::find` walks up the directory tree during CLI config resolution, before any command logic runs, which is why every command was affected; `from_directory` only tolerated `NotFound` when reading the config.
>
> Following review, the fix lives entirely in `uv-settings`: `from_directory` returns a new `Error::Directory` when `uv.toml` or `pyproject.toml` is a directory, and `FilesystemOptions::find` propagates that error if and only if it occurs at the top level of the search; for parent directories it emits a `warn!` (silent by default) and continues. Because this runs during config resolution, the top-level error surfaces for all commands, including ones that don't do project discovery (`uv venv`, `uv pip`), and the same handling covers `uv.toml`.
>
> Includes before/after output, five integration tests, and `Closes #14584`.

**Maintainer Feedback:**

- _<date>_: EliteTK self-requested a review within 24 hours of submission. Requested: remove an unnecessary comment, add a warning when skipping directories in parent locations (`warn!`, silent by default), extend the same handling to `uv.toml`, and — in a follow-up comment — move the fix out of the discovery functions into `FilesystemOptions::find`, propagating directory errors only at the top level of the search.
- _<date>_: Addressed all feedback. Reverted the `uv-workspace` changes, moved directory handling into `FilesystemOptions::find` with top-level-only propagation and `warn!` for parents, added `uv.toml` coverage, and added a `uv venv` integration test. Force-pushed as `f99814d`. Flagged one edge case for the maintainer's input.
- _<date>_: Awaiting maintainer response on the reworked approach.

**Status:** Iterating (reworked per review; awaiting maintainer response)

---

## Learnings & Reflections

### Technical Skills Gained

- Rust error handling with `thiserror`, the difference between fatal and non-fatal conditions in a CLI, how `Result/?` propgation works, snapshot resting with insta, navigating a large multi-crate Rust workspace with ripgrep, the WSL2/Linux dev workflow. 

### Challenges Overcome

- Toolchain setup chain on Windows/WSL; tracing the root cause to the correct layer with a discriminating test; the redesign, by taking architectural feedback and moving the fix to `FilesystemOptions:find` withou breaking exisiting behavior. 

### What I'd Do Differently Next Time

- I should have considered where in the call graph an error should live first, check if the function is only ever called on the CWD, before placing a check there; check for adjacent cases lik `uv.toml` from the start rather than scoping them out. 

### Resources Used

- [uv CONTRIBUTING.md](https://github.com/astral-sh/uv/blob/main/CONTRIBUTING.md)
- [The Rust Book](https://doc.rust-lang.org/book/) — `Result`, error handling, pattern matching
- [insta snapshot testing docs](https://insta.rs/)
- The issue thread and PR review discussion on [astral-sh/uv#14584](https://github.com/astral-sh/uv/issues/14584) / [#19762](https://github.com/astral-sh/uv/pull/19762)

