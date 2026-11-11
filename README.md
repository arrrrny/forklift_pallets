<!-- ZURAFFA-LAUNCH-PROMO (valid until Dec 1, 2026) -->
## 🦒 [Zuraffa Browser](https://zuraffa.com) — now live

<div align="center">
  <a href="https://zuraffa.com"><img src="https://zuraffa.com/img/zuraffa-browser-home.jpg" alt="Zuraffa Browser — 10 Browsers. One window." width="800"></a>
</div>

**[Zuraffa Browser](https://zuraffa.com)** — the multi-tasking browser for AI agents & humans — is live on the [Mac App Store](https://apps.apple.com/app/id6811352058).

🎁 **Launch offer: 6 months free with code `ZURAFFA`** — no credit card, no subscription, no payment. Redeem in-app until **Dec 1, 2026**.

---

# forklift_pallets

Modular, self-contained projects ("pallets") implemented by cloud agents.
Each pallet lives in its own folder under `pallets/`, is fully independent, and
never touches another pallet's files.

## Why this repo

- **1 repo, unlimited projects.** No need to create a new repo per idea — every
  project is a `pallets/<name>/` folder inside this single public repo.
- **Concurrent agents via git worktrees.** Pallets are isolated from each other,
  so multiple agents can work at once — each in its own worktree pointing at
  `pallets/<their-pallet>/`. No two agents ever touch the same folder.

## Structure

```
forklift_pallets/
├── README.md
├── docs/
│   ├── pallet-spec.md   # how to write a pallet + its spec
│   └── worktrees.md     # the concurrent git-worktree workflow
└── pallets/
    ├── .gitkeep
    └── <pallet-name>/           # one folder per project
        ├── README.md           # what it is, how to run
        ├── SPEC.md             # the spec (how it's implemented/shipped)
        └── ...                 # the actual project
```

## Concept

A pallet is a "modular artifact" derived from a real need. Example: instead of
a generic ML detection task, define a pallet **fruits-detector** — a fully
working app that detects fruits/vegetables (a spec with multiple sub-tasks).

Each fork/private-repo concern that's better as a standalone, reusable project
becomes a pallet here and gets assigned as a task to a cloud agent.

## Worktree flow (concurrent agents)

```
git fetch origin
git worktree add ../worktree-<pallet> origin/main
cd ../worktree-<pallet> && work in pallets/<pallet-name>/
git add/commit/push   (never touch any folder but your pallet)
```

See [docs/worktrees.md](docs/worktrees.md) for the full workflow and safety
rules.