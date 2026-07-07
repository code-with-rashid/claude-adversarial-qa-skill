# QA Inventory

Code-derived, exhaustive list of everything testable, per `adversarial-qa` Phase 0.
Derive every row from the actual code — entry points, branch maps, dependency
graphs — never hand-wave an item in.

## How to read this

- **ID**: stable identifier, referenced from `TRIED.jsonl` and `COVERAGE.json`. Never
  renumber once attacks have been logged against an ID.
- **Categories**: which attack categories apply — Valid / Boundary / Invalid /
  Hostile / Concurrency / Failure-injection / State-transition / Auth. Not all apply
  to all items.
- **Status**: `not-started` / `in-progress` / `done` (`done` = every applicable
  category exercised AND the item sits inside the coverage/mutation threshold).

## Entry points

| ID | Type | Location | Params | Categories | Status | Notes |
|----|------|----------|--------|------------|--------|-------|
| EP-001 | route / CLI / export / handler | `file:line` | | | not-started | |

## Branches / decision points

| ID | Location | Condition | Status | Notes |
|----|----------|-----------|--------|-------|
| BR-001 | `file:line` | | not-started | |

## State machines / lifecycles

| ID | Model | States | Transitions | Status | Notes |
|----|-------|--------|-------------|--------|-------|
| SM-001 | | | | not-started | |

## External dependencies & failure modes

| ID | Dependency | Failure modes to inject | Status | Notes |
|----|------------|--------------------------|--------|-------|
| DEP-001 | | | not-started | |

## Data invariants

| ID | Invariant | Where enforced | Status | Notes |
|----|-----------|-----------------|--------|-------|
| INV-001 | | | not-started | |

## Config / flags / env combinations

| ID | Flag(s) | Combination | Status | Notes |
|----|---------|-------------|--------|-------|
| CFG-001 | | | not-started | |

## Trust boundaries / authorization rules

| ID | Boundary | Rule | Status | Notes |
|----|----------|------|--------|-------|
| AUTH-001 | | | not-started | |
