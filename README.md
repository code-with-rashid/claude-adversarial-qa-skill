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

### Recommended: as a plugin

Inside a Claude Code session, in any project:

```
/plugin marketplace add code-with-rashid/claude-adversarial-qa-skill
/plugin install adversarial-qa@claude-adversarial-qa-skill
```

Then run `/reload-plugins` (or start a fresh `claude` session) — a plugin installed
mid-session doesn't retroactively load into that session's context, so the skill
won't be visible yet, and "harden this codebase" won't trigger it, until you do.
Confirm with `/plugin list`, or `claude plugin list` from a regular terminal.

By default this installs at **user scope** (available in every project). To scope it
to just the current project instead, run the equivalent from a terminal:

```bash
claude plugin marketplace add code-with-rashid/claude-adversarial-qa-skill --scope project
claude plugin install adversarial-qa@claude-adversarial-qa-skill --scope project
```

To remove it later: `/plugin uninstall adversarial-qa@claude-adversarial-qa-skill`.

<details>
<summary>Alternative: manual copy (no plugin system)</summary>

This isn't a package — it's a folder with a `SKILL.md`. **Clone this repo first**,
then copy `skills/adversarial-qa/` into a `.claude/skills/` directory. (The commands
below assume you have *not* already `cd`'d into a checkout of this repo — if you
have, skip the `git clone` line and use `skills/adversarial-qa` as the source path.)

Use this if you'd rather not use the plugin system, or want to read/vet the skill's
contents before trusting it in a project — a skill's instructions run with whatever
tool access you grant it, so reviewing it first is reasonable practice either way.

#### macOS / Linux (bash/zsh)

```bash
git clone https://github.com/code-with-rashid/claude-adversarial-qa-skill.git ~/claude-adversarial-qa-skill

# per-project (recommended — keeps qa/ state alongside the repo it hardens)
mkdir -p /path/to/your-repo/.claude/skills
cp -r ~/claude-adversarial-qa-skill/skills/adversarial-qa /path/to/your-repo/.claude/skills/

# per-user (available in every project, instead of per-project)
mkdir -p ~/.claude/skills
cp -r ~/claude-adversarial-qa-skill/skills/adversarial-qa ~/.claude/skills/
```

#### Windows (PowerShell)

```powershell
git clone https://github.com/code-with-rashid/claude-adversarial-qa-skill.git $HOME\claude-adversarial-qa-skill

# per-project
New-Item -ItemType Directory -Force -Path "C:\path\to\your-repo\.claude\skills" | Out-Null
Copy-Item -Recurse -Force "$HOME\claude-adversarial-qa-skill\skills\adversarial-qa" "C:\path\to\your-repo\.claude\skills\"

# per-user
New-Item -ItemType Directory -Force -Path "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse -Force "$HOME\claude-adversarial-qa-skill\skills\adversarial-qa" "$HOME\.claude\skills\"
```

No build step, no dependencies — it's markdown. Only the copied
`skills/adversarial-qa` folder matters; the rest of the clone (this README, the
license) can be deleted afterward, or kept around so `git pull` can fetch future
updates to the skill before you re-copy it.

#### If you install it both ways

Don't — pick one. If a project ends up with both the plugin installed *and* a manual
copy under its own `.claude/skills/adversarial-qa/`, the project-level copy wins for
the bare `/adversarial-qa` command (project skills take priority over plugin skills
with the same name). The plugin is still reachable explicitly at
`/adversarial-qa:adversarial-qa`, but having both around is confusing for no benefit.

</details>

## Use

Open Claude Code in the target repo and either ask for it in plain language
("harden this codebase", "run adversarial QA on this", "drive mutation score up on
the payments module") or invoke it directly:

```
/adversarial-qa
```

(If you installed it as a plugin *and* separately have a manual copy in the same
project, use `/adversarial-qa:adversarial-qa` to be explicit about which one runs —
see [If you install it both ways](#if-you-install-it-both-ways) above.)

On first run it asks only what it can't infer from your repo. In practice, that
means it reads your manifest and test config, states the stack/tooling/thresholds
it's going to use, and only stops to ask when something is genuinely ambiguous (for
example, whether to scope a monorepo run to specific packages) — it does not ask
for information already sitting in `package.json`, `pubspec.yaml`, or CI config.
It then writes `qa/CONFIG.md`, builds `qa/INVENTORY.md`, and starts the loop. On a
later run in the same repo, it reads the persisted state and resumes at the
frontier — a well-converged repo should see the next run exit almost immediately,
having found nothing new.

### If natural-language phrases like "harden this codebase" don't trigger it

Skill auto-invocation is Claude deciding your request matches a skill's
description — it's a judgment call, not a guarantee, and it competes with every
other skill you have installed. Two things to check if it doesn't fire:

1. **Did you just install it?** A plugin installed mid-session doesn't
   retroactively load into that session — run `/reload-plugins` or start a fresh
   session first (see the reload note under [Install](#recommended-as-a-plugin)).
2. **Do you have a lot of other skills/plugins installed?** Skill descriptions
   share a limited context budget, and a skill you've never invoked is the first
   to get shortened or dropped when it overflows. Run `/doctor` to check whether
   `adversarial-qa` is being truncated.

Either way, `/adversarial-qa` (typed explicitly) always works regardless of
auto-detection, since it's how this was actually verified end-to-end.

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
