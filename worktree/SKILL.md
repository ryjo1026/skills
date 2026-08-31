---
name: worktree
description: Use when the user wants a git worktree for a task or branch — a new feature branch, or an existing branch/PR to review and edit — that external tools can open (any editor: Cursor/VS Code/Zed/nvim; plus lazygit, cmux, tmux). Creates a plain git worktree at a predictable sibling path, runs project setup, and prints the path. Do NOT let Claude Code create it natively (--worktree, or EnterWorktree with a name).
---

# Create an externally-openable git worktree

Produce a **real git worktree on disk at a predictable path** that any tool opens by path. That is the only tool-agnostic interface — every editor and multiplexer opens a directory.

**Core rule — applies to *creating* the worktree:** never let the agent create it (`--worktree`, or `EnterWorktree` with `name`). That places it under `.claude/worktrees/`, where no external tool (editor, lazygit, cmux, tmux) can discover it. Always create with plain `git worktree add` at the sibling path below.

*Entering* it is separate and fine — see step 4. `EnterWorktree` with `path` moves the session into a worktree you already created, which is what you want.

**Announce at start:** "Creating a git worktree with the worktree skill."

## Layout

All worktrees for a repo live in one **sibling** container next to the main checkout:

```
<parent>/<repo>/                 ← main checkout
<parent>/<repo>.worktrees/       ← one container per repo
  <slug>/                        ← one worktree per task/branch
```

Sibling (not `<repo>/.worktrees/`) so editors that open the main checkout never descend into and re-index the worktrees, and no `.gitignore` entry is needed — the container is outside the working tree.

## Steps

### 1. Detect existing isolation — never nest

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" && pwd -P)
```

If `GIT_DIR != GIT_COMMON`, first rule out a submodule:

```bash
git rev-parse --show-superproject-working-tree   # non-empty ⇒ submodule, treat as normal repo
```

If it's a linked worktree (not a submodule), stop — you're already isolated. Report the path and branch; don't create another.

### 2. Resolve paths

```bash
MAIN=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)  # main working tree
REPO=$(basename "$MAIN")
CONTAINER="$(dirname "$MAIN")/$REPO.worktrees"
```

Derive a filesystem-safe **slug** — default to the branch's last path segment, lowercased (`user/PROJ-42-add-search` → `proj-42-add-search`); accept an explicit slug if the user gives one:

```bash
SLUG=$(printf '%s' "${BRANCH##*/}" | tr '[:upper:]' '[:lower:]' | tr -c 'a-z0-9._-' '-' | sed 's/-\{2,\}/-/g; s/^-//; s/-$//')
PATH_WT="$CONTAINER/$SLUG"
```

### 3. Choose the branch

- **New feature branch:** `git worktree add -b "$BRANCH" "$PATH_WT" <base>` (base defaults to the current/default branch).
- **Existing branch / PR review:** fetch first, then check it out into the worktree:
  ```bash
  git fetch origin
  git worktree add "$PATH_WT" "$BRANCH"        # local branch
  # or track a remote-only branch:
  git worktree add -b "$BRANCH" "$PATH_WT" "origin/$BRANCH"
  ```
  For a PR by number, resolve its branch with `gh pr view <n> --json headRefName -q .headRefName` first.

If the user hasn't said new-vs-existing, ask.

### 4. Enter the worktree

Re-root into the worktree — the checkout you ran from is not where the work should happen. The worktree is a *sibling* of the main checkout, so a plain `cd` may leave file tools scoped to the old directory.

**In Claude Code**, move the whole session with `EnterWorktree` using **`path`** (never `name` — that would create a new buried one):

```
EnterWorktree({ path: "$PATH_WT" })
```

This re-roots the session itself: cwd, file-tool scope, and cwd-dependent caches (memory, plans). Because the worktree came from `git worktree add`, it's in `git worktree list` and accepted on first entry from the launch directory. `ExitWorktree` will not delete a worktree entered by `path` — use `action: "keep"` to return.

**Other agents / plain shells:** `cd "$PATH_WT"`, or pass the path explicitly (`git -C "$PATH_WT" …`) if cwd doesn't persist between commands. A Claude Code user can also type `/cd "$PATH_WT"` (v2.1.169+) by hand, or `/add-dir "$PATH_WT"` to allow edits without moving the session.

### 5. Project setup

Run the repo's setup in the new worktree so it's ready to build/test. If the repo ships its own setup skill or script, use that instead of hand-rolling. Otherwise detect the ecosystem:

- **Node monorepo (pnpm/npm/yarn):** install deps, then build the shared/workspace packages the app depends on — a fresh worktree has no built `dist/`.
- **Auto-detect otherwise:** `Cargo.toml`→`cargo build`; `go.mod`→`go mod download`; `requirements.txt`/`pyproject.toml`→pip/poetry install.

### 6. Report the path — stay tool-agnostic

Print the worktree path and the open commands, and stop. Do **not** launch or assume any specific tool.

```
Worktree ready: <PATH_WT>   (branch <BRANCH>)

Open it in:
  editor    cursor <PATH_WT>   |  code <PATH_WT>  |  zed --add <PATH_WT>  |  nvim <PATH_WT>
  git UI    lazygit -p <PATH_WT>
  terminal  cmux <PATH_WT>     |  tmux new-window -c <PATH_WT>
```

## Stacked PRs (`gh stack`) in this layout

`gh stack` has no worktree support — its help names worktrees zero times. Every command that
moves between layers runs `git checkout` in the current tree, and git refuses a branch that a
second worktree already holds:

```
fatal: 'user/PROJ-42b-api' is already used by worktree at '<repo>.worktrees/proj-42b-api'
```

So with one worktree per branch, **do not run** any navigation command (`top`, `bottom`, `up`,
`down`, `trunk`, `switch`), nor `checkout`, `rebase`, or `sync`. Each has a plain-git substitute
that works because the owning worktree already has that branch checked out:

| Instead of | Run |
| --- | --- |
| any `gh stack` navigation command | `cd` to the worktree that owns the branch |
| `gh stack rebase` | `git rebase <parent-branch>` inside the owning worktree |
| `gh stack sync` | `git fetch`, then that rebase, per worktree |

Safe here: `gh stack view` (reads refs only) and `gh stack link` (holds no local state).

**Prefer `gh stack link`.** It exists for branches managed by external tools and gives the
GitHub stack UI — the visual chain and PR-to-PR navigation — without local stack tracking:

```bash
# per worktree: rebase onto the parent, then push
git rebase <parent-branch> && git push --force-with-lease
gh pr create --base <parent-branch>
# once, bottom to top — re-run when the branch set changes
gh stack link <bottom-branch> <top-branch>
```

Reach for `gh stack init` only when every branch lives in one working tree. Adopting a stack
whose branches are spread across worktrees leaves navigation and rebase permanently broken.

## Switching worktrees in one editor window (VS Code / Cursor)

Not part of this skill — one-time editor setup. The no-reload trick is swapping the workspace _folder set_ in place (`updateWorkspaceFolders`), which preserves terminals and editor state, unlike "Open Folder" (a full reload).

- **Cursor / VS Code:** prefer the built-in worktree support — VS Code 1.103+ has native `Git: Open Worktree in Current Window`, and _Add Folder to Workspace_ does the no-reload folder-set swap. Third-party switcher extensions layer on top if wanted. All discover worktrees via `git worktree list`, so this sibling layout needs no extra config.
- **Zed:** `zed --add <path>` adds the worktree to the current project.
- **nvim:** tab-local `:tcd <path>`.

## Teardown

Only remove worktrees under `*.worktrees/` (ones this skill created); never remove a harness-owned worktree. Run from the main checkout, not inside the worktree:

```bash
cd "$MAIN"
git worktree remove "$PATH_WT"   # add --force if it has uncommitted changes you intend to drop
git worktree prune
```

Removing the worktree does not delete the branch; delete it separately if wanted.
