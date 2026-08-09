# Pallet spec — how a pallet gets written & shipped

A **pallet** is a self-contained project under `pallets/<name>/` that a cloud
agent implements. It is defined by a spec, and shipped as working code.

## Pallet anatomy

```
pallets/<name>/
├── SPEC.md        # REQUIRED — the spec: goal, sub-tasks, acceptance, run cmd
├── README.md      # REQUIRED — what it is, how to run it, any private-repo deps
└── <source...>    # the actual project
```

## Writing a SPEC.md

A pallet can be one feature or a bundle of sub-tasks. When it's a bundle, the
SPEC enumerates them (like `fruits-detector` = model + UI + camera + tests).

Keep it agent-actionable:

```
# <name>
Goal: <one-line outcome>
Scope: <what's in / out>
Stack: <lang, framework, SDKs>
Key facts: (the research that motivates it, e.g. "flutter_ml_kit detects
  objects but not fruits specifically — so we add a classifier on top")
Acceptance: (how to prove it works — run command, expected result)
Sub-tasks:
- [ ] ...
- [ ] ...
```

## Deriving a pallet

A pallet usually comes from a real need that is better as a standalone,
reusable artifact than a one-off patch:
- a private-repo concern (e.g. `flutter_ml_kit` object detection) generalized
  into its own project;
- a reusable piece you'd otherwise re-scaffold per app.

Write the SPEC first, then assign it as a pool task → a cloud agent implements
it in its own worktree.

## Shipping

- Every pallet is reviewed before/after merge (CodeRabbit); fix on its own
  branch, push, wait for review.
- An agent never commits outside `pallets/<its-pallet>/`.