<!-- GSD:project-start source:PROJECT.md -->
## Project

**GQHSM** (klhurley/GQHSM)

A revived ~2005–2010 Google Code project: visual editing of Quantum Hierarchical State Machines. A WinForms-era editor (StateProto/MurphyPA, including the H2D hierarchical 2-D drawing editor) on top of an XML-driven hierarchical state machine runtime (qf4net — the C# port of Miro Samek's Quantum Framework — plus the GQHSM glue layer). Phase 2 (2026-11-24) converted the tree to .NET 9; the editor UI pivots to Uno + SkiaSharp in later phases.

**Core Value:** existing GQHSM XML state machines still load and run unchanged, and any state machine can round-trip losslessly between XML and JSON.

### Constraints

- **Runtime:** .NET 9 (`net9`), pinned by the repo-root `global.json` (.NET SDK 9.0.321, `rollForward: latestPatch`). The 2026-11-23 pivot to Uno + SkiaSharp superseded the original Mono/.NET Framework plan — no Mono components are needed or useful anymore.
- **Platforms:** macOS, Linux, Windows (desktop only).
- **Editor UI (planned):** Uno + SkiaSharp via pinned NuGet (Uno.Sdk 6.7.49, SkiaSharp 3.119.2).
- **Build:** CMake through the `build/` (cmaketemplate) submodule driving `dotnet` over the two solutions. Configure fetches nothing — zero network downloads, ever.
- **Compatibility:** the existing `.xml` corpus is the source of truth and must keep loading unchanged (enforced by the `gqhsm_corpus_*` ctest rows). JSON deliberately mirrors the XML 1:1.
- **Repo layout:** `qf4net/` and `StateProto/` are submodules — code changes commit inside the owning sub-repo, then bump the root gitlink.
<!-- GSD:project-end -->

## Build Rules (non-negotiable)

Full prerequisites, per-OS runbooks, and offline proofs live in `docs/BUILD.md` — these are the rules that break builds when ignored:

- **Never run cmake from the repo root** — the root CMakeLists.txt is not a build entry and fails loudly. Configure only via `scripts/configure.sh osx|linux [Debug|Release]` / `scripts\configure.cmd`.
- **dotnet is not assumed on PATH** — configure discovers it through `DOTNET_ROOT`, `~/.dotnet`, and system prefixes; when missing, configure exits with the per-OS install hint.
- **Never configure without the scripts' `-D` flags** (`ZKLLVM_BUILD_DIR`, `THIRDPARTY_DIR`): those plus the committed `thirdparty/` stub dirs are what guarantee zero configure-time downloads.
- **Build:** `cmake --build build/<Platform>/<Cfg>` runs the two solutions in order (`gqhsm_qf4net`, then `gqhsm_h2d`).
- **Test:** `ctest --test-dir build/<Platform>/<Cfg> --output-on-failure --no-tests=error` — eight rows (`gqhsm_artifacts`, six `gqhsm_corpus_*`, `gqhsm_samplelighter`). Both Debug and Release must be green.
- **A stale binary cannot fake a green build** — `gqhsm_artifacts` asserts the assemblies are newer than the CMake cache; after a reconfigure, make sure the build actually rebuilt.
- **Offline proof on demand:** `scripts/run-corpus.sh` (dead proxy + fresh `NUGET_PACKAGES` scratch that must stay empty).
- **Windows and Linux lanes are PENDING-OWNER** — a lane counts as done only when its full console log exists as evidence, never by inference.

## Architecture Constraints (always follow)

- **Separation of concerns:** engine runtime (`qf4net/`) · editor (`StateProto/`) · build wiring (`cmake/`, `build/` submodule) · planning (`.planning/`). Name the one concern of the file you edit.
- **Encapsulation:** public contracts only; reach another module's internals through its functions.
- **Cohesion / coupling:** one rule change touches one module.
- **DRY:** port and reuse the existing implementation; do not write a second copy beside it.
- **KISS / YAGNI:** simplest working shape; no speculative hooks, flags, or frameworks.
- **Fail fast (validate at edges, never swallow errors) · optimize for deletion · boring tech.**

The hard invariants (never violate):

1. **One owning sub-repo per concern** — state machine semantics live in `qf4net/`; editor and drawing code in `StateProto/`; the root repo carries build wiring and gitlinks only. A change inside a submodule commits to that submodule first, pushes, then bumps the parent gitlink.
2. **Never duplicate logic** — when converting or porting, carry the existing implementation over; a parallel "modern" twin beside the old code is a bug.
3. **No domain logic in adapters or presentation** — editor UI displays model state; state machine semantics are computed in the engine.
4. **The XML corpus is the compatibility contract** — tracked `.xml` files load unchanged under the current build; any format work must preserve that.

## Conventions

- **PRs target `master`**; `develop` is the integration branch. Docs-only changes may push straight to develop.
- **Submodules first, then the gitlink:** `qf4net/`, `StateProto/`, and `build/` are submodules — commit inside, push, then bump and commit the parent's reference. Every shell starts fresh at the repo root: always `git -C <path>` or `cd <path> && ...` in a single command.
- **PR summaries are plain English** — lead with the human consequence; no tracking codes or internal jargon.
- **Planning artifacts live in `.planning/`** (gitignored).

## Project Skills

- **Spike findings for GQHSM** (implementation patterns, constraints, gotchas) → `Skill("spike-findings-gqhsm")`

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.

## Build-System Pre-PR Checklist

Run `/gsd-code-review` with this checklist before any PR that touches builds (CMakeLists, *.cmake, build scripts, codegen wiring). These are the cross-platform / cross-tree failure modes that pass on the dev machine and only surface in external review:

- **Artifact routing:** a SHARED-library target needs BOTH `RUNTIME_OUTPUT_DIRECTORY` (Windows DLL) and `LIBRARY_OUTPUT_DIRECTORY` (macOS dylib / Linux .so), PLUS per-config `<PROP>_<CONFIG>` entries — multi-config generators (Xcode, Ninja Multi-Config, MSVC) append `$<CONFIG>` to the plain directories, landing artifacts in `plugins/Debug/`-style subdirs that loaders never scan.
- **Interpreter discovery:** NEVER `find_program(<var> python3)` — native Windows exposes `python.exe` / `py.exe`, and the lookup fails there even when Python is installed. Use `find_package(Python3 COMPONENTS Interpreter REQUIRED)` and alias to `Python3_EXECUTABLE`.
- **No POSIX shell in build commands:** `sh -c` breaks native Windows. Drive multi-command logic with `${CMAKE_COMMAND} -DPYTHON=... -P driver.cmake` script mode.
- **Absent submodules:** a non-recursive clone must still configure. Gate every `REQUIRED` lookup and `add_subdirectory` on an existence check — nothing tool-`REQUIRED` may run before the submodule guard.
- **Default-off opt-ins:** wiring behind an `option(... OFF)` must execute nothing in a default build — audit `ALL` custom targets chained on generation / dependency resolution.
- **Standalone subdir configures:** any `add_subdirectory`-able directory may be configured alone. `if(TARGET ...)`-guard every cross-directory link dependency; unguarded bare names degrade to `-l` linker flags.
- **Generated sources:** a target compiling generated files registers only when its producer wiring exists (flag or `TARGET` check); otherwise skip with a `message(STATUS ...)` — never fail configure on a file only another wiring produces.
- **Spec-driven regeneration:** generated code must regenerate when its spec changes (`CMAKE_CONFIGURE_DEPENDS` on the spec + newer-than stamp), including fresh trees (configure-time pre-generation before the globs).

## CI Policy: No Blind Reruns (MANDATORY)

**CI runners are limited, costly resources. Never waste runner time on builds that have no verified reason to succeed.**

This policy applies to all GitHub Actions workflows, including self-hosted runners.

### 1. Never rerun CI to diagnose a failure

- **NEVER** use `gh run rerun --failed`, `gh run rerun --all`, or GitHub's "Re-run all jobs" as a diagnostic tool.
- **NEVER** rerun a failed build just to see whether it passes this time.
- **NEVER** assume a merged fix will appear in a rerun of an existing workflow.
- **NEVER** trigger expensive builds before checking the failure and the source being built.
- Do not run multiple CI attempts while guessing at fixes.

### 2. Diagnose failures before taking action

When a CI job fails:

1. Retrieve the failed job's logs using `gh run view <run-id> --log-failed`.
2. Identify the exact compiler error, test failure, missing dependency, or infrastructure problem.
3. Locate the relevant code and determine the cause.
4. Identify the specific fix and its commit SHA.
5. Verify that the fix addresses the observed failure.
6. Confirm that the fix exists in the **exact commit or merge ref** that the next build will check out.

Use local builds, targeted tests, static analysis, or code inspection whenever practical. Prefer these over consuming remote runner time.

### 3. Verify the actual build ref

**GitHub Actions reruns use the original run's commit and ref.**

For `pull_request` workflows:

- GitHub commonly builds a synthetic merge commit combining the PR branch and its base branch.
- A rerun uses the original workflow's SHA and ref, not a newly updated merge.
- Merging a fix into the base branch does not make that fix available to an earlier run's rerun.
- When the original merge ref lacks a fix, update the PR branch and push it to trigger a fresh workflow run.

Before rerunning, inspect the workflow's checkout behavior and determine the actual commit it will build. Do not rely solely on the current state of the base branch or the PR.

Where commit ancestry applies, verify inclusion with:

`git merge-base --is-ancestor <fix-sha> <build-sha>`

An exit code of `0` confirms that the fix commit is an ancestor of the build commit. If the fix was cherry-picked or applied differently, inspect the actual source contents instead.

**If the fix cannot be confirmed in the build ref, STOP. Do not rerun.**

### 4. Use the smallest possible CI workload

Once the fix and build ref have been verified:

- Prefer local validation before invoking CI.
- Run only the affected tests or jobs when possible.
- For source-code fixes, trigger a fresh run containing the fix rather than rerunning an obsolete workflow.
- For confirmed transient infrastructure failures, retry only the affected job, and only when evidence supports a retry.
- Avoid restarting successful jobs or unrelated matrix entries.
- Do not launch overlapping builds for the same change without a clear reason.

A transient failure is not permission to retry blindly. If its cause remains unknown, ask before spending more runner time.

### 5. Report evidence before triggering a build

Before starting another CI attempt, establish:

- **Failure:** The specific error from the failed job.
- **Cause:** Why the previous attempt failed.
- **Fix:** What changed and which commit contains it.
- **Build ref:** The commit or merge SHA the next workflow will use.
- **Verification:** Evidence that the fix is present in that ref.
- **Scope:** The minimum jobs required to validate the change.

If any critical detail is missing, investigate further rather than starting a build.

### 6. Resource conservation is a priority

Self-hosted CI runners consume real hardware, compute time, electricity, and developer capacity.

Treat runner time as a limited resource:

- Minimize redundant compilation.
- Avoid duplicate or speculative workflows.
- Reuse valid test results where appropriate.
- Prefer targeted tests over full matrix builds during development.
- Cancel obsolete runs when safe and appropriate.
- Never use remote CI as a substitute for basic local troubleshooting.

### Absolute rule

**NO VERIFIED FIX IN THE BUILD REF = NO CI RERUN.**

For a proven infrastructure failure, a single targeted retry may be justified. Otherwise, diagnose locally, correct the source, verify the exact build ref, and only then trigger the smallest necessary CI workload.

When uncertain, stop and report the findings instead of consuming runner resources.

## Sources

- `docs/BUILD.md` — authoritative for build/test commands, toolchain prerequisites, runbooks, and offline proofs.
- `.planning/PROJECT.md` — GSD source of truth for the project definition and key decisions.
- `.planning/ROADMAP.md`, `.planning/STATE.md` — phase plan and progress.
