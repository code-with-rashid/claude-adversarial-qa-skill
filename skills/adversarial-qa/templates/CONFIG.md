# QA Config

Resolved once at the start of the first run (Step 0 of SKILL.md). Re-read on every
resume so the run never re-asks the user for the same answers.

- **Scope**: <whole repo | path/to/module>
- **Stack(s) detected**: <e.g. Flutter/Dart, Node/TS>
- **Commands**:
  - Test: `<command>`
  - Coverage: `<command>`
  - Mutation: `<command, or "none available — using mutation drills">`
  - Fuzz / property: `<command, or "none available — see QA_LOG for justification">`
  - Load / soak: `<command, or "N/A — not a running service">`
- **Thresholds**:
  - Line coverage ≥ `90%`
  - Branch coverage ≥ `85%`
  - Mutation score ≥ `80%`
  - Fuzz budget per target: `1,000,000 execs OR 30 min`, zero new crashes in final stretch
  - Soak duration: `1h`, zero resource growth
- **Non-functional applicable?**: `yes | no — <reason>`
- **Resolved on**: `<date>`
- **Last resumed**: `<date>`
