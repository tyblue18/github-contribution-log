# Contribution #2: Blueprint for Porting Apache Airflow Breeze to Apache Sedona

**Contribution Number:** 2
**Student:** Tanishq Somani
**Issue:** [apache/sedona#2993](https://github.com/apache/sedona/issues/2993)
**Status:** Phase II

---

## Why I Chose This Issue

I picked this issue because the pain it describes is the exact pain I hit trying to make my first change to Sedona. Sedona is a dual-language project: the geospatial engine is Scala/Java built with Maven, the user-facing package is Python built on PySpark, and the two are glued together by shaded JARs that have to be compiled before the Python tests can pass. To run the test suite the way CI runs it, a contributor needs a specific JDK, Maven, a specific Scala version, a specific Spark version, and a specific Python version — and CI tests four different combinations of those. Setting that up by hand on a laptop is most of a day of work, and getting it slightly wrong produces failures that look like code bugs but are really environment bugs. That is a real barrier to entry for a project that wants new contributors, and it is measurable: it is the difference between a first-time contributor landing a PR and giving up.

It also fits where I am as a developer. The issue is unusually well scoped for a feature request — the author laid out the four pieces (architecture deconstruction, a `click` CLI with `shell`/`test`/`build-image`, a dual-language Dockerfile, and a suggested directory layout) and even named the trap to avoid, which is over-complicating the backend into a full Spark cluster when local mode covers almost every workflow. It is tagged `help wanted` and `improvement`, and there is a proven reference implementation to learn from in Apache Airflow's Breeze. I use Docker, Spark and Python packaging constantly as a consumer, but I have never built the tooling side of it — the part that decides what a *contributor's* environment looks like. Porting Breeze meant reading Airflow's CLI structure, reading Sedona's CI workflows closely enough to reproduce their build matrix exactly, and then deciding what to deliberately leave out. And because Sedona is an ASF project, everything I write has to satisfy real review standards: license headers on every file, no shell injection, documentation that a stranger can follow.

---

## Understanding the Issue

### Problem Description

Apache Sedona has no supported local development environment. A contributor who clones the repo and wants to run the tests has to assemble the toolchain themselves: a JDK (11 for Spark 3.x, 17 for Spark 4.x), Maven, the right Scala version, the right PySpark version, and a matching Python interpreter. The repo does ship Docker assets under [docker/](docker/), but those build the *demo/notebook* image that is published to Docker Hub for end users — they are not a development environment and they do not mirror the CI matrix.

The result is that the only place the project's supported version combinations are actually described is [.github/workflows/python.yml](.github/workflows/python.yml), and the only place they are actually exercised is GitHub Actions. A contributor's laptop is a fifth, undocumented, unreproducible environment. This is what issue #2993 asks to fix, by porting the Airflow Breeze concept: a thin CLI that wraps Docker so the local environment *is* the CI environment.

Note that this is a feature request ("blueprint"), not a bug report. There is no crash to reproduce — the thing being reproduced is the setup cost itself, which is why the reproduction section below measures friction rather than capturing a stack trace.

### Expected Behavior

A contributor with only Docker installed should be able to clone Sedona and, in one command, get an interactive shell that already has the JDK, Maven, PySpark, and an editable install of `apache-sedona`, pinned to a chosen cell of the CI matrix:

```bash
./breeze shell --python 3.11 --spark 4.0.0 --scala 2.13.8
./breeze test -- tests/core
```

Code edited on the host should take effect inside the container immediately, without an image rebuild. Version combinations that the project does not support should be rejected up front with an explanatory message rather than failing deep inside a Spark stack trace.

### Current Behavior

There is no `breeze`. A contributor either installs the full toolchain on the host — accepting that it will only ever match one cell of the CI matrix and will conflict with whatever Java/Spark versions their other projects need — or repurposes the end-user Docker image, which does not have Maven, the test dependencies, or a live mount of their working tree. Version incompatibilities are discovered at runtime: asking for Spark 4.x with Scala 2.12 fails during dependency resolution, and Spark 4.x on Python 3.9 fails at import time, neither with a message that points at the actual cause.

### Affected Components

- **New:** [dev/sedona_breeze/](dev/sedona_breeze/) — the entire contribution, self-contained. Nothing in Sedona's build, packaging, or CI is modified.
- **Read as the source of truth:** [.github/workflows/python.yml](.github/workflows/python.yml) — the CI build matrix that the version constants must mirror.
- **Referenced for parity:** [docker/sedona-docker.dockerfile](docker/sedona-docker.dockerfile) — the existing end-user image, used to match the JDK choice.
- **Reference implementation studied:** Apache Airflow's `dev/breeze`.

---

## Reproduction Process

### Environment Setup

The development host is Windows 11 with Docker Desktop, which shaped several decisions. Notes on what went wrong and how I handled it:

- **`./breeze` is a bash script and Windows has no bash on PATH by default.** Rather than write and maintain a parallel `breeze.ps1`, I documented the Windows entry point as `uv run --project dev/sedona_breeze breeze shell`, which is what the shell script does anyway. The script stays the POSIX entry point; `uv` is the cross-platform one.
- **Line endings.** Committing `entrypoint.sh` with CRLF makes the container fail with an unhelpful `exec format`-style error. Sedona's `.gitattributes`/pre-commit setup keeps shell scripts LF, and I verified the file stayed LF after commit.
- **Bind-mount performance.** The repository lives under OneDrive on the host; live-mounting it into the container is noticeably slower than a native Linux bind mount, and the first `pip install -e` of the Python package (which compiles a native extension) is slow. This is why the uv cache is a named Docker volume rather than a host mount — it keeps repeated installs off the slow path.
- **First-run cost.** Building the CI image pulls Ubuntu, a JDK, Maven and PySpark. I sized the Dockerfile so the expensive layers (apt, JDK, PySpark) sit above anything that changes often, so rebuilds after editing the entrypoint are cheap.

### Steps to Reproduce

The friction the issue describes, reproduced on a clean machine:

1. Clone `apache/sedona` and read [.github/workflows/python.yml](.github/workflows/python.yml) to find out what the project actually tests against. Four combinations are listed: Spark 4.1.1/Scala 2.13.8/Java 17/Python 3.11, Spark 4.0.0/Scala 2.13.8/Java 17/Python 3.10, Spark 3.5.0/Scala 2.12.8/Java 11/Python 3.9, and Spark 3.4.0/Scala 2.12.8/Java 11/Python 3.8 (with `shapely<2`).
2. Try to run the Python tests without building the Scala side first. They fail, because the Python package needs the shaded JARs that `mvn clean install -DskipTests -Dspark=... -Dscala=...` produces — a step that is only discoverable by reading the CI workflow.
3. Try to test a second cell of the matrix on the same machine. This requires a second JDK, a second Python interpreter and a different PySpark, side by side with the first.

**Observed result:** reproducing what CI does takes reading a 200-line workflow file and installing four separate toolchains, and testing more than one matrix cell means managing conflicting global installs. Nothing in the repo documents this; the CI workflow is the documentation.

### Reproduction Evidence

- **Commit showing reproduction and fix:** `48e8e77b` on branch `feature/sedona-breeze` — "[SEDONA-XXXX] Add Sedona Breeze containerized development environment" (20 files, 1237 insertions).
- **Screenshots/logs:** every Breeze command echoes the exact `docker` / `docker compose` invocation it runs (`utils/docker_command_utils.py:53`), so the terminal output doubles as a reproduction log — a contributor can copy any line and run it by hand.
- **My findings:**
  1. The version matrix is the whole problem. Everything else follows from pinning it.
  2. Spark's own constraints are not encoded anywhere a contributor can see: Spark 4.x is only published for Scala 2.13, and the CI workflow's own comment confirms Spark 4.x requires Python 3.10+. These deserved to be validated in the CLI, not discovered at runtime.
  3. CI installs PySpark from PyPI at an exact pinned version (`uv add pyspark==${SPARK_VERSION}`) rather than downloading a Spark distribution. Mirroring that exactly in the image is what makes the local environment genuinely equal to CI.
  4. The JARs must exist before Python tests are meaningful — the same warning the issue author gave.

---

## Solution Approach

### Analysis

The root cause is that the project's supported environments are defined only inside CI, and CI is not runnable locally. Any fix has to make that matrix a first-class, executable artifact in the repo. Two constraints shape the design:

- The environment cannot be baked into an image, because a developer's whole workflow is editing code and re-running tests. If the sources were copied into the image, every edit would mean a rebuild. So the repository must be **live-mounted**, and the Python package installed in **editable mode at container start** rather than at build time.
- The matrix is multiplicative, so there is no single image. There must be one image **per cell**, cached by tag, built on demand.

The issue's own warning — do not over-complicate the backend — is the other half of the analysis. A Spark master/worker/history-server compose stack looks impressive and helps almost nobody: Sedona development is overwhelmingly done against Spark in local mode.

### Proposed Solution

A `click`-based CLI in `dev/sedona_breeze` that wraps Docker and Docker Compose:

- A `BreezeCiParams` dataclass represents one (Python, Spark, Scala) cell, validates it against Spark's real constraints, and derives both the Maven profiles (`-Dspark=3.5 -Dscala=2.12`) and the image tag from it.
- A single parameterized `Dockerfile.ci` builds a per-cell image tagged `sedona-breeze-ci:python3.11-spark3.5.0-scala2.12.8`, containing the JDK, Maven, the exact PySpark version and the Python toolchain — but no Sedona sources.
- A deliberately lean `docker-compose.yaml`: one container, repository live-mounted at `/opt/sedona`, host `~/.m2` shared so Maven does not re-download the world, uv cache persisted in a named volume, ports published only when a command asks for them.
- Four commands: `shell`, `test`, `jupyter`, `build-image`. `shell`/`test`/`jupyter` build the image automatically when it is missing.

### Implementation Plan (UMPIRE)

**Understand.** Contributors cannot reproduce CI locally because Sedona's supported Python × Spark × Scala combinations exist only inside a GitHub Actions workflow. Give them a CLI that stands up any one of those combinations in a container, with their working tree live-mounted.

**Match.** Patterns I borrowed rather than invented:
- *Airflow Breeze* — the overall shape: a `click` command group split into developer commands and CI/image commands, a `global_constants` module holding the matrix, per-matrix-cell image tags, and a shell script entry point that bootstraps the CLI's own dependencies.
- *Sedona's own CI* — the version matrix, the `mvn -Dspark=<major.minor> -Dscala=<major.minor>` profile convention, and PySpark installation from PyPI at a pinned version.
- *Sedona's existing Docker assets* — [docker/sedona-docker.dockerfile](docker/sedona-docker.dockerfile) for the JDK choice, so Breeze does not introduce a third opinion about Java.
- *ASF conventions* — Apache license header on all 20 files, and `# nosec` annotations with justifying comments where Bandit flags `subprocess`.

**Plan.**
1. Add `dev/sedona_breeze/pyproject.toml` declaring `click` + `rich` and a `breeze` console script.
2. Add `global_constants.py` transcribing the CI matrix (allowed/default Python, Spark, Scala; JDK; image repo; container root; Jupyter port).
3. Add `params.py` with `BreezeCiParams`: `validate()`, `spark_profile`, `scala_profile`, `image_tag`.
4. Add `utils/path_utils.py` to find the repository root by marker files (`pom.xml`, `.asf.yaml`), from the module location first and the CWD second.
5. Add `utils/docker_command_utils.py`: `check_docker_is_available`, `ci_image_exists`, `build_ci_image`, `ensure_ci_image`, `run_in_container` — all list-argument, never `shell=True`.
6. Add `utils/common_options.py` so `--python/--spark/--scala/--force-build` are declared once, with `click.Choice` and env-var fallbacks.
7. Add `docker/Dockerfile.ci` parameterized by four build args; `docker/docker-compose.yaml`; `docker/entrypoint.sh` doing the editable install.
8. Add `commands/developer_commands.py` (`shell`, `test`, `jupyter`) and `commands/ci_commands.py` (`build-image`); wire them in `main.py`.
9. Add the `breeze` shell entry point that prefers `uv` and falls back to system Python.
10. Add `README.md` covering prerequisites, quick start, the matrix, the manual JAR build, layout, and troubleshooting.
11. **Add unit tests** — see Testing Strategy. *Not yet done; this is the top of the remaining work.*

**Implement.** Branch `feature/sedona-breeze`, commit `48e8e77b`. All 20 files are new and live under `dev/sedona_breeze/`; no existing file is modified.

**Review.** Self-review checklist against Sedona's contribution guidelines:
- [x] Apache license header on every new file (including YAML, Dockerfile and shell scripts)
- [x] No changes to build, packaging or CI — the contribution is additive and cannot break existing workflows
- [x] `subprocess` used with argument lists only, never `shell=True`; `# nosec` annotations carry a justification comment
- [x] Docstrings explain *why*, and the constants file states explicitly that it mirrors `python.yml`
- [x] Documentation ships with the code, including a troubleshooting section
- [x] Matches the structure the issue author proposed
- [ ] Automated tests — **not yet written**
- [ ] Pre-commit run clean on a POSIX host (verified on Windows only so far)
- [ ] Commit message uses a real issue key rather than the `SEDONA-XXXX` placeholder

**Evaluate.** Verification is three-layered: unit tests for the pure logic (validation and tag derivation, no Docker needed), an integration run of `build-image` + `test` for at least two matrix cells, and a manual walkthrough of the README from a clean clone.

---

## Testing Strategy

### Unit Tests

*Status: not yet implemented. `pytest` is declared but no test module exists in commit `48e8e77b`.*
Still working on getting proper test implemented, ran into a lot of issues

- [ ] **Test case 1 — matrix validation rejects Spark 4.x on Scala 2.12.** `BreezeCiParams("3.11", "4.0.0", "2.12.8").validate()` raises `click.UsageError` naming both the constraint and the fix.
- [ ] **Test case 2 — matrix validation rejects Spark 4.x on Python < 3.10.** `("3.9", "4.0.0", "2.13.8")` raises; `("3.10", "4.0.0", "2.13.8")` does not.
- [ ] **Test case 3 — profile and tag derivation.** `spark_profile` and `scala_profile` truncate to major.minor (`4.0.0` → `4.0`, `2.13.8` → `2.13`) since that is what the Maven profiles expect, and `image_tag` round-trips the full triple.
- [ ] **Test case 4 — repository root discovery.** `find_sedona_root()` finds the root from a nested CWD, and raises a `ClickException` (not an `IndexError`) from a directory that is not a Sedona clone.
- [ ] **Test case 5 — Docker preflight.** With `docker` absent from PATH, `check_docker_is_available()` raises the install-Docker message; with `docker info` returning non-zero, it raises the daemon-not-running message.
- [ ] **Test case 6 — command construction.** With `subprocess.run` patched, `breeze test --module scala` builds `mvn -Dspark=… -Dscala=… test` at `/opt/sedona`, and `--module python` builds `pytest` at `/opt/sedona/python`; `extra_args` are forwarded verbatim in both.
- [ ] **Test case 7 — image reuse.** `ensure_ci_image` skips the build when the image exists and rebuilds when `force_build=True`.

### Integration Tests

- [ ] **Scenario 1 — full matrix-cell round trip.** On a Docker-enabled runner, `breeze build-image --python 3.9 --spark 3.5.0 --scala 2.12.8`, then `mvn clean install -DskipTests` inside `breeze shell`, then `breeze test -- tests/core`. Confirms the image, the mount, the editable install and the JAR handoff all line up.
- [ ] **Scenario 2 — live mount is genuinely live.** Edit a Python source file on the host while a `breeze shell` session is open and confirm the change is visible inside the container without any rebuild.
- [ ] **Scenario 3 — CI parity.** Run the same test selection under Breeze and under the GitHub Actions job for the same matrix cell and confirm the results agree. This is the claim the whole contribution rests on.
- [ ] **Scenario 4 — two cells side by side.** Build Spark 3.5/Scala 2.12 and Spark 4.0/Scala 2.13 on one machine and confirm the tags do not collide and neither image disturbs the other.

### Manual Testing

Performed on Windows 11 + Docker Desktop:

- `breeze --help` and each subcommand's `--help` render, and the defaults shown match `global_constants.py`.
- Invalid combinations are rejected before Docker is touched: `--spark 4.0.0 --scala 2.12.8` and `--spark 4.0.0 --python 3.9` both fail immediately with a message that names the fix. `click.Choice` rejects versions outside the matrix entirely.
- With Docker stopped, commands fail with "Docker is installed but the daemon does not respond" instead of a raw traceback.
- `build-image` produces a correctly tagged image; a second `shell` invocation reuses it instead of rebuilding.
- Inside `breeze shell`: `java -version` and `mvn -v` resolve, `python -c "import pyspark; print(pyspark.__version__)"` reports the requested version, and the repository is present at `/opt/sedona`.
- `breeze jupyter` publishes Lab on `http://localhost:8888` and `--port` moves it.
- Running from a subdirectory works — repository-root discovery is not CWD-dependent.

**Not yet verified:** scenario 3 (CI parity) and a clean-clone run on Linux/macOS. Those are Phase II.

---

## Implementation Notes

### Week 1 Progress (through Jul 13, 2026)

Read the issue's blueprint alongside Airflow's `dev/breeze` to decide how much to port. Airflow's Breeze is enormous — dozens of commands, a self-upgrade mechanism, a CI image registry. Porting it wholesale would produce something nobody would review. I took the shape (click groups, `global_constants`, per-cell image tags, a bootstrap shell script) and cut everything else, landing on the three commands the issue actually names plus `jupyter`.

Then read [.github/workflows/python.yml](.github/workflows/python.yml) line by line, since it is the real specification. Two details changed the design: CI installs PySpark from PyPI at a pinned version rather than downloading a Spark tarball (so the Dockerfile does the same, and there is no `SPARK_HOME` to manage), and Spark 4.x needs both Scala 2.13 and Python 3.10+ (so those became validation rules rather than README warnings).

Built the whole thing in one commit, `48e8e77b`: 20 files, 1237 lines, entirely additive.

Decisions worth recording:

- **One lean container, not a cluster.** Directly following the issue's warning. Spark local mode covers nearly all Sedona development, and a master/worker/history-server stack is more to maintain and slower to start for something most contributors would never use.
- **No sources in the image; editable install at container start.** The image holds only the toolchain. `entrypoint.sh` installs `apache-sedona` from the mounted `/opt/sedona/python` in editable mode on start. This is what makes edit-and-rerun instant.
- **Validate in the CLI, not in Docker.** Impossible combinations fail in milliseconds with an actionable message, instead of after a multi-minute image build.
- **Shared caches.** Host `~/.m2` is bind-mounted so Maven reuses whatever the developer already downloaded; the uv cache is a *named volume* rather than a host mount, because on Windows a bind-mounted cache is slow enough to undo the benefit.
- **`uv` first with a real fallback.** The `breeze` script prefers `uv run --project`, and falls back to system Python with an explicit `click`/`rich` check and a pip command to fix it — never a bare `ImportError`.
- **Security posture up front.** Every Docker invocation is a list of arguments with `shell=True` nowhere, and the `# nosec` markers carry the reasoning inline so a reviewer does not have to reconstruct it.

### Week 2 Progress (through Jul 28, 2026)

Manual verification pass (results above) and self-review against the checklist. What that surfaced, honestly:

1. **Java version divergence — the most important known gap.** `DEFAULT_JDK_VERSION` is hardcoded to `17`, but CI uses **Java 11** for Spark 3.4.0 and 3.5.0 and Java 17 only for Spark 4.x. So the "what passes locally passes in CI" promise is not yet exact for Spark 3.x cells. Fix: derive the JDK from the Spark version in `BreezeCiParams` and pass it through as a build arg — the plumbing already exists, only the value is wrong.
2. **Breeze allows a superset of CI's matrix.** CI pins four *pairings*; Breeze allows any Python × Spark × Scala combination that passes validation, and its default (3.11 / 3.5.0 / 2.12.8) is not itself a CI cell. That is deliberate — a contributor should be able to test 3.5.0 on 3.11 — but it needs to be stated in the README so nobody reads "mirrors CI" as "restricted to CI".
3. **JAR building is still manual.** `breeze test --module python` will fail on anything requiring the shaded JARs until the user runs Maven inside `breeze shell`. The README documents this, but the issue author flagged it as the main trap, so a `breeze build-jars` command (or having `test` detect missing JARs and say so) belongs in Phase II.
4. **The editable-install guard is coarse.** `entrypoint.sh` skips reinstalling when `import sedona` succeeds, which can leave a stale install in place after the package metadata changes. Worth tightening.
5. **No automated tests yet.** The validation and tag-derivation logic is pure and trivially testable with no Docker required; that gap is the first thing to close.

**Next up (Phase II):** map the JDK to the Spark version, add the unit tests above, run pre-commit on a POSIX host, verify CI parity on one matrix cell end to end, and replace the `SEDONA-XXXX` placeholder in the commit message before opening the PR.

### Code Changes

**Files modified:** none. All 20 files are new under `dev/sedona_breeze/`:

| File | Purpose |
| --- | --- |
| `breeze` | Shell entry point; prefers `uv run`, falls back to system Python |
| `pyproject.toml` | CLI package metadata, `click` + `rich`, `breeze` console script |
| `README.md` | Prerequisites, quick start, matrix, JAR build, layout, troubleshooting |
| `.gitignore` | Keeps Breeze's own venv/caches out of the repo |
| `docker/Dockerfile.ci` | Per-cell image: JDK, Maven, build tools, uv-provisioned Python, pinned PySpark |
| `docker/docker-compose.yaml` | Single service, live mount, `~/.m2` + uv cache, ports on demand |
| `docker/entrypoint.sh` | Editable install of `apache-sedona` from the mount, then `exec "$@"` |
| `src/sedona_breeze/main.py` | Root `click` group wiring all commands |
| `src/sedona_breeze/__main__.py` | `python -m sedona_breeze` support for the no-uv fallback |
| `src/sedona_breeze/global_constants.py` | The CI matrix, defaults, image repo, container root, Jupyter port |
| `src/sedona_breeze/params.py` | `BreezeCiParams`: validation, Maven profiles, image tag |
| `src/sedona_breeze/commands/developer_commands.py` | `shell`, `test`, `jupyter` |
| `src/sedona_breeze/commands/ci_commands.py` | `build-image` |
| `src/sedona_breeze/utils/common_options.py` | Shared `--python/--spark/--scala/--force-build` with env-var fallbacks |
| `src/sedona_breeze/utils/docker_command_utils.py` | Docker preflight, image build/reuse, `docker compose run` |
| `src/sedona_breeze/utils/path_utils.py` | Repository-root discovery via marker files |
| `src/sedona_breeze/utils/console.py` | Shared `rich` console |
| `__init__.py` × 3 | Package markers |

**Key commits:** `48e8e77b` — "[SEDONA-XXXX] Add Sedona Breeze containerized development environment" (branch `feature/sedona-breeze`).

**Approach decisions:**

- *Ported the shape of Airflow Breeze, not its scale.* Four commands, matching the issue's own list, instead of Airflow's dozens. A reviewable diff beats a complete one.
- *Treated `python.yml` as the specification.* `global_constants.py` says so in its module docstring, so the next person to touch it knows where the numbers come from and what to do when CI changes.
- *Made the CI parity claim falsifiable.* Every command echoes the exact Docker invocation it runs, so a reviewer can check the claim instead of trusting it.
- *Chose additive over invasive.* Nothing outside `dev/sedona_breeze/` changes, so the contribution cannot break an existing workflow — which for a first substantial PR to an ASF project is worth more than a slightly cleaner integration.


## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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
