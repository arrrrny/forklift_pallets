# Git worktrees for concurrent pallet agents

Each pallet is an isolated folder, so multiple agents can work at once — each
in its **own worktree**. Only this repo uses worktrees (the pool repos do not).

## Why worktrees

One working tree is shared mutable state. If 3 agents all check out `main`,
they stomp each other. A worktree is a full separate checkout of the repo
linked to a folder, so agent A edits `pallets/fruits-detector`, agent B edits
`pallets/ocr-app`, and neither ever sees the other's uncommitted work.

## Worktree actor is ONE per agent

A worktree lives on the agent's own machine (or its own branch). Since each
pallet is a disjoint folder, worktrees never collide — even on the same branch,
the merged result is clean because no two agents commit to the same path.

## Flow

```bash
# one clone is the remote source of truth
git clone git@github.com:arrrrny/forklift_tasks.git pallets-src

# an agent working on "fruits" adds ONE worktree
cd pallets-src
# software — I only touch pallets/fruits-detector
git worktree add ../fruits-agent origin/main -b pallet/fruits-detector
cd ../fruits
# do all work inside pallets/fruits-detector/ ONLY
git add pallets/fruits-detector
git commit -m "fruits-detector: detection app"
git push -u origin pallet/fruits-detector
```

The agent then RETURNS the PR; the release-review merges it to `main`. Other
agents keep their own worktrees on their own branches — zero overlap.

## Safety rules

- Commit ONLY files under your pallet's folder. Any edit outside it is a
  signal violation and will be rejected.
- Push your branch, open a PR back to `main`; never push directly to `main`
  while others have open worktrees on it.
- `git worktree list` shows all live trees; `git worktree remove` to clean up.
- Rebasing is repo-aggressive — disagree on shared `main` push.

## Add an agent to this repo

Give the cloud agent:
- the public repo URL + the specific pallet folder,
- the SPEC under `pallets/<name>/SPEC.md` as the task prompt,
- the rule: "work only in `pallets/<your-pallet>/`, push a branch, open a PR."

## Oldest sane rule

If two pallets ever (accidentally) overlap, that's an anti-pattern to fix by
renaming — never by two agents editing the same folder.