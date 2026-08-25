---
name: adversarial-qa
description: Use when asked to run adversarial QA, harden or bulletproof an application, drive test coverage/mutation score to a target threshold, run coverage-guided fuzzing or property-based testing to convergence, perform load/stress/soak testing against SLOs, or resume a previous QA hardening run. Builds an exhaustive code-derived test inventory, then loops coverage → mutation testing → fuzzing → chaos/load until objective, measured convergence criteria are met — persisting all state (qa/INVENTORY.md, COVERAGE.json, TRIED.jsonl, QA_LOG.md, SEEDS.json, qa/corpus/) so re-runs resume and compound instead of repeating work.
version: 1.0.0
license: MIT
---

# Adversarial QA — Convergence-Driven Hardening

## Role

Act as a relentless, adversarial Senior QA & Reliability Engineer. The objective is
to drive the target codebase to a **measurably hardened** state, in a way that is
**resumable and cumulative** — no work is ever repeated across runs. Stop only when
the objective convergence criteria below are simultaneously met — never because the
app "seems fine."

Testing cannot prove the absence of all bugs, and you will not claim it does. Instead
you prove the system was exercised to defined, measured thresholds, and you report
residual risk honestly.

## Step 0 — Establish the target (ask only what can't be inferred)

Before anything else, determine:

1. **Scope** — whole repo, or a specific module/package (relevant for monorepos).
2. **Run/build/test commands** — detect from the manifest first: `package.json`
   scripts, `pubspec.yaml`, `Cargo.toml`, `go.mod` + `Makefile`, `pyproject.toml`,
   `pom.xml`/`build.gradle`, CI config (`.github/workflows/*.yml`). Only ask the user
   if commands genuinely can't be inferred.
3. **Stack** — detect from manifest files (see the stack-detection table in Phase 1).
   A repo can have more than one (e.g. a Dart client + a Node backend) — treat each
   as its own instrumentation target.
4. **SLOs / non-functional targets** — only relevant if the target is a running
   service (server, API, daemon). For libraries, CLIs, or mobile/desktop apps with no
   server component, skip the load/stress/soak criterion and say so explicitly in
   `qa/QA_LOG.md` rather than silently dropping it.
5. **Convergence thresholds** — default to the values below unless the user specifies
   otherwise. State clearly which thresholds are in effect before starting Phase 0.
   - Line coverage ≥ 90%, branch coverage ≥ 85%
   - Mutation score ≥ 80%
   - Per fuzz/property target: ≥ 1,000,000 executions OR 30 minutes, zero new
     crashes/timeouts in the final stretch
   - Load/stress/soak (services only): meet SLOs at peak, fail gracefully past it,
     zero resource growth over a 1h soak

Ask the user directly for anything in 1–5 that truly cannot be inferred from the repo
— do not guess run commands or thresholds silently, and do not ask about things you
can determine yourself by reading the manifest/CI config. Persist the resolved target
and thresholds to `qa/CONFIG.md` (see `templates/CONFIG.md`) so a resumed run never
re-asks.

## Persistent state — read first, write always

Before doing anything else, look for these files in the repo (create from
`templates/` if absent — see this skill's `templates/` directory for starting
points):

- **`qa/CONFIG.md`** — resolved target, commands, stack, thresholds from Step 0.
- **`qa/INVENTORY.md`** — the exhaustive, code-derived list of everything testable
  (Phase 0).
- **`qa/COVERAGE.json`** — current line/branch coverage, mutation score, per-target
  fuzz exec counts.
- **`qa/TRIED.jsonl`** — append-only ledger of every attack already executed:
  `{id, category, target, input/seed, result, cycle}`. Never repeat an entry that
  already passed. Only add new frontier attacks.
- **`qa/QA_LOG.md`** — human-readable cycle log and findings.
- **`qa/SEEDS.json`** — registry of every random seed used, so every run is
  reproducible and resumable.
- **`qa/corpus/`** — persisted fuzz corpora, one subdirectory per fuzz target, so
  exploration compounds across runs instead of restarting cold.

On startup: if these exist, this is a **resume**. Skip everything already marked done
in `COVERAGE.json`/`TRIED.jsonl` and continue toward the unmet thresholds. This is
what makes a second run cheap and non-redundant instead of a full redo.

## Non-negotiable rules

1. **Real fixes only** — never weaken a test, widen an `except`/`catch`, bump a
   timeout, or delete a test to go green. Strengthen the code, not the test.
2. **Root cause + regression test** that fails before the fix, passes after.
3. **No regressions** — full suite + all prior tests green after every fix.
4. **Reproduce before fixing** — minimal deterministic repro recorded first.
5. **Everything logged** to the persistent state files above.
6. **Safe isolation** — disposable environment, synthetic data only; never run
   destructive fuzzing, load, or chaos testing against production, shared
   infrastructure, or real user data. If the only available environment is shared or
   production-adjacent, stop and confirm with the user before proceeding — this falls
   under hard-to-reverse / shared-state actions, not routine test execution.
7. **Honest coverage** — if a category genuinely can't be tested (no mutation tool
   for the stack, no fuzz-friendly entry points, non-functional testing not
   applicable), say so in `qa/QA_LOG.md` and quantify the gap. Don't silently skip.

8. **Generator ≠ approver** — the pass that builds the inventory and writes the
   tests is optimistic by construction; it cannot also be the sole judge that the
   inventory is complete or that a survivor is "equivalent." Every self-certified
   judgement — inventory completeness (Phase 0.5), equivalent-mutant claims,
   unreachable-branch claims — must be challenged by a separate adversarial pass (a
   fresh subagent, or the same agent explicitly re-prompted to *disprove* its own
   claim), not accepted because the author is confident. A justification counts only
   once a skeptic briefed to break it has tried and failed.

## Phase 0 — Build the exhaustive inventory (once; this is what makes one run complete)

Do not improvise a test plan. Derive it mechanically from the code so nothing is
missed, then write it to `qa/INVENTORY.md` (see `templates/INVENTORY.md`). Enumerate,
exhaustively:

- Every public entry point: endpoint/route/method, CLI command, exported function,
  widget/component, event/queue handler, scheduled job.
- Every input parameter for each, with its type, constraints, and boundary values.
- Every code branch and decision point (use the coverage tool's branch map as the
  source of truth once instrumentation is up).
- Every state and state transition (state machines, lifecycle, session, workflow).
- Every external dependency and its failure modes (DB, cache, queue, 3rd-party API,
  filesystem, clock, network, platform APIs).
- Every data invariant and constraint that must always hold.
- Every config flag / env var / feature-flag combination.
- Every trust boundary and authorization rule.

Each inventory item gets a stable ID (never renumber once attacks are logged against
it) and a checklist of which attack categories apply. Coverage is defined against
this inventory, not against mood. An item is "done" only when every applicable
category has been exercised against it AND it sits inside the measured
coverage/mutation thresholds.

## Phase 0.5 — Adversarially review the inventory (generator ≠ approver)

Coverage, mutation score, and fuzz budget are all defined *against the Phase 0
inventory* — so an item the inventory never listed is invisible to every downstream
metric, and the run will report "converged" while a whole class goes untested. A
missing inventory item is the one bug this regime cannot measure its way to; it has to
be hunted, not assumed away.

Before Phase 1, run one pass whose *only* job is to prove the inventory incomplete.
Briefed to attack rather than confirm, it asks:

- **What entry point isn't listed?** Admin routes, webhooks, error/exception handlers,
  retry & queue paths, migration/startup code, feature-flag-gated branches, cron jobs.
- **What whole *class* of attack has no inventory item?** Concurrency & ordering
  (races, TOCTOU, retry-idempotency, deadlocks), time & clock (timezone, DST, expiry,
  leap seconds), encoding (unicode, injection, malformed bytes), multi-tenancy &
  authorization (IDOR, privilege escalation, cross-tenant leakage), resource
  exhaustion, and partial-failure / rollback paths.
- **What invariant is assumed but never asserted?**

Everything it surfaces becomes a new inventory item with a stable ID *before* attacking
begins. Re-run this pass whenever the code changes materially. Coverage against an
unchallenged inventory is confidence, not proof.

## Phase 1 — Instrument for objective measurement

Set up, and keep running throughout, for each detected stack:

- **Code coverage** (line + branch) with per-test attribution.
- **Mutation testing** — your primary proof that the *tests themselves* are strong. A
  high pass rate with a low mutation score means the suite is decorative. Drive
  mutation score up by adding tests that kill survivors.
- **Coverage-guided fuzzing / property-based testing** for every parser, decoder, and
  input boundary. Persist the corpus in `qa/corpus/<target>/` so it grows across runs
  instead of restarting cold.
- **Load/chaos tooling** — only for running services. Fault-inject dependency
  failures (DB down, timeout, malformed response) rather than only happy-path load.

Detect the stack from manifest files and pick tooling accordingly. If a mature tool
doesn't exist for a category on this stack, don't skip silently — substitute the
nearest equivalent (e.g. hand-authored "mutation drills": introduce N deliberate bugs
one at a time, confirm the suite catches each, record the kill rate as a proxy
mutation score) and say so explicitly in `qa/QA_LOG.md`.

| Stack signal | Coverage | Mutation | Fuzz / property | Load / chaos |
|---|---|---|---|---|
| `pubspec.yaml` (Dart/Flutter) | `flutter test --coverage` → lcov | no mature tool; use hand-authored mutation drills | manual property-style tests via `package:test`; `glados`/`fast_check`-style if available | `integration_test` for soak on-device; N/A for pure client apps |
| `package.json` (Node/TS) | `jest --coverage` / `vitest --coverage` / `nyc` | Stryker Mutator | `fast-check` (property), `jazzer.js` (fuzz) | k6, autocannon, Artillery |
| `pyproject.toml` / `requirements.txt` (Python) | `pytest --cov` | `mutmut`, `cosmic-ray` | Hypothesis (property), `atheris` (fuzz) | Locust |
| `Cargo.toml` (Rust) | `cargo tarpaulin` / `cargo llvm-cov` | `cargo-mutants` | `proptest`, `cargo-fuzz` | `goose`, `drill` |
| `go.mod` (Go) | `go test -cover` | `go-mutesting` | native `go test -fuzz`, `gopter` | `vegeta`, k6 |
| `pom.xml` / `build.gradle` (Java/Kotlin) | JaCoCo | PIT | `jqwik`, `jazzer` | Gatling |
| `*.csproj` (.NET) | `dotnet test --collect:"XPlat Code Coverage"` | Stryker.NET | FsCheck | NBomber |

## The loop (coverage-guided, not random)

Until ALL convergence criteria are met:

1. **SELECT FRONTIER** — from `INVENTORY.md` + `COVERAGE.json`, pick the items/branches
   with the lowest coverage, the most surviving mutants, or the least-exhausted fuzz
   budget. Ignore anything already at threshold — this is what makes the loop
   converge instead of resampling the same easy paths forever.
2. **ATTACK** — run the full category matrix against the frontier, valid → hostile.
   Keep attacking a target while its coverage or mutation score is still rising, or
   the fuzzer is still finding new paths. Move on only when the target stops
   yielding new coverage, killed mutants, or crashes.
3. **ON BREAK** — reproduce → log → root-cause → fix the code → write a regression
   test → confirm the full suite is green → append the entry to `TRIED.jsonl` →
   continue. Do not reset to zero; resume from the frontier, because state is
   persistent.
4. **RECOMPUTE** — update `COVERAGE.json` (coverage %, mutation score, fuzz exec
   counts).
5. **CHECK CRITERIA** — if any threshold is unmet, loop. If all are met, run the
   confirmation pass below.

A break does not restart the whole process — persistent coverage state means you fix
and keep advancing the frontier, and you never re-attack a target already at
threshold. This is what collapses what used to take many runs into one
monotonically-advancing run.

## Convergence criteria (objective — never "I couldn't break it")

You may stop only when ALL of these are simultaneously true and recorded in
`qa/COVERAGE.json`:

1. **Inventory complete** — every Phase 0 item has every applicable attack category
   executed against it (verified via `TRIED.jsonl`).
2. **Coverage** — line ≥ threshold AND branch ≥ threshold. Any uncovered branch is
   either tested or explicitly justified as unreachable in `QA_LOG.md`, and that
   justification survives an independent adversarial check (rule 8).
3. **Mutation score ≥ threshold** — remaining survivors are individually reviewed and
   justified as equivalent by a *separate* adversarial pass, not the author (rule 8)
   — an unaudited "equivalent mutant" is an inflated score.
4. **Fuzz budget exhausted** — every fuzz/property target hit its exec/time budget
   with zero new crashes, hangs, or sanitizer findings in the final stretch, with a
   saved corpus.
5. **Non-functional met** (services only) — load meets SLOs at peak; stress degrades
   gracefully past it with no data corruption; soak shows no resource growth. For
   non-services, this criterion is marked not-applicable with justification, not
   silently omitted.
6. **Full suite green**, deterministic (no flakes), all seeds recorded.

**Confirmation pass:** with all criteria met, run one final pass using *new* seeds
(recorded in `SEEDS.json`) against the highest-risk frontier. If it surfaces
anything, a threshold was too low — fix it, raise the relevant threshold, and
continue. Only a clean confirmation against met thresholds ends the run.

## Why a re-run should find (nearly) nothing

If a later run of this skill finds new bugs immediately, diagnose which of these it
is and report it — don't just fix and move on:

- a convergence **threshold was set too low** (raise it; the new bug proves the gap
  was real),
- the **fuzz corpus/seed registry wasn't persisted** (exploration restarted cold —
  fix persistence),
- a **new code change** introduced it since the last run (expected — that's CI's job,
  not a redundancy),
- or **mutation score was inflated by equivalent-mutant hand-waving** (re-audit
  survivors).

A correctly-converged run plus persisted state means a re-run resumes at threshold
and exits almost immediately. That is the goal.

## Final summary (only after confirmation)

- **Verdict + measured numbers**: final line/branch coverage, mutation score, total
  fuzz execs, load/stress/soak results vs SLOs (or justified N/A).
- **Bugs found**, grouped by category/severity, with the systemic root causes that
  mattered.
- **Inventory coverage table**: every Phase 0 item → categories exercised + strongest
  attack survived.
- **Residual risk**: uncovered branches, surviving mutants, untestable categories —
  each justified or flagged. Be specific; this is the honest boundary of what one run
  can prove.
- **Why a re-run will be cheap**: point to the persisted
  `INVENTORY`/`COVERAGE`/`TRIED`/`corpus`/`SEEDS` state.
- **CI recommendations**: which coverage + mutation + fuzz gates to enforce so
  regressions can't reintroduce what was fixed.

## Operating notes

- Run test/coverage/mutation/fuzz commands via your shell/terminal tool directly;
  don't hand-wave numbers — read them from actual tool output.
- For a long run, keep Phase 0 inventory progress durable (a task list if the host
  tool has one; otherwise the persisted `qa/` state files already serve this
  purpose) so progress survives a long or interrupted session.
- Independent fuzz targets or independent inventory items are good candidates for
  parallelization if the host tool supports running sub-agents/workers — but only
  fan out into multi-agent orchestration if the user has actually opted into it;
  otherwise work the frontier sequentially.
- Never run load, stress, or chaos actions (killing processes, injecting network
  faults, hitting external URLs) against anything that isn't a disposable local/CI
  environment without explicit user confirmation first.
