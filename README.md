# Adversarial QA — a Claude Code skill

A [Claude Code](https://claude.com/claude-code) skill that turns "run the tests and
see if it's fine" into a **measured, resumable hardening loop**: build an exhaustive,
code-derived test inventory, then drive coverage, mutation score, fuzzing, and (for
services) load/soak testing to explicit numeric thresholds — persisting all state so
a re-run picks up exactly where the last one left off instead of starting cold.

## Why this exists

Most "please find bugs" prompts stop when the agent *feels* done. This skill instead:

- Derives the test plan **from the code** (entry points, branches, state machines,
  invariants, trust boundaries) instead of improvising one.
- Defines "done" as **objective thresholds** — line/branch coverage, mutation score,
  fuzz exec counts, SLO adherence — not vibes.
- Persists everything (`qa/INVENTORY.md`, `qa/COVERAGE.json`, `qa/TRIED.jsonl`,
  `qa/QA_LOG.md`, `qa/SEEDS.json`, `qa/corpus/`) so a second run on the same repo is
  cheap: it resumes at the frontier instead of repeating work.
- Is **stack-agnostic** — it detects Dart/Flutter, Node/TS, Python, Rust, Go,
  Java/Kotlin, or .NET from the repo's manifest files and picks the matching
  coverage/mutation/fuzz/load tooling for each.

See [`skills/adversarial-qa/SKILL.md`](skills/adversarial-qa/SKILL.md) for the full
methodology.

## Install

Skills are just a folder with a `SKILL.md`. Copy `skills/adversarial-qa/` into
either:

**Per-project** (recommended — keeps `qa/` state alongside the repo it hardens):

```bash
mkdir -p /path/to/your-repo/.claude/skills
cp -r skills/adversarial-qa /path/to/your-repo/.claude/skills/
```

**Per-user** (available in every project):

```bash
mkdir -p ~/.claude/skills
cp -r skills/adversarial-qa ~/.claude/skills/
```

No build step, no dependencies — it's markdown.

## Use

Open Claude Code in the target repo and either ask for it in plain language
("harden this codebase", "run adversarial QA on this", "drive mutation score up on
the payments module") or invoke it directly:

```
/adversarial-qa
```

On first run it will ask only what it can't infer from your repo (run commands,
thresholds if you want non-default ones), write `qa/CONFIG.md`, then build
`qa/INVENTORY.md` and start the loop. On a later run in the same repo, it reads the
persisted state and resumes at the frontier — a well-converged repo should see the
next run exit almost immediately, having found nothing new.

## What "done" looks like

A run only stops when, simultaneously:

1. Every inventory item has every applicable attack category exercised against it.
2. Line coverage and branch coverage are both above threshold (default 90% / 85%).
3. Mutation score is above threshold (default 80%), with any survivors justified.
4. Every fuzz/property target hit its exec/time budget with zero new crashes.
5. For services: load/stress/soak meet SLOs, or non-functional testing is explicitly
   marked not-applicable with a reason.
6. The full suite is green and deterministic.

Full details, including the "why a re-run should find nothing" diagnostic and the
final-summary format, are in the skill file itself.

## License

MIT — see [LICENSE](LICENSE).
