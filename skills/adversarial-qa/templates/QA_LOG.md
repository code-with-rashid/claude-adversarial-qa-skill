# QA Log

Human-readable cycle-by-cycle log. Append a new `## Cycle N` section every loop
iteration — never delete or rewrite history, only add to it.

## Cycle 0 — Setup (`<date>`)

- **Target**: `<repo/module, scope>`
- **Stack(s)**: `<detected stacks>`
- **Thresholds**: line ≥ `__%`, branch ≥ `__%`, mutation ≥ `__%`, fuzz ≥ `__` execs /
  `__` min per target, soak ≥ `__` h
- **Inventory**: `<N>` items derived — see `INVENTORY.md`
- **Instrumentation**: `<coverage tool>`, `<mutation tool>`, `<fuzz/property tool>`,
  `<load tool>` — or explicit justification for any category that doesn't apply or
  has no mature tool for this stack

<!-- Subsequent cycles appended below. Each cycle should record: frontier selected,
     what was attacked, any breaks found + root cause + fix + regression test,
     recomputed coverage/mutation/fuzz numbers, and criteria status. -->
