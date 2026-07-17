---
name: worktree
description: Use when the user wants a git worktree for a task or branch — a new feature branch, or an existing branch/PR to review and edit — that external tools can open (any editor: Cursor/VS Code/Zed/nvim; plus lazygit, cmux, tmux). Creates a plain git worktree at a predictable sibling path, runs project setup, and prints the path. Do NOT use Claude Code's native --worktree / EnterWorktree feature for this.
---

# Create an externally-openable git worktree

Produce a **real git worktree on disk at a predictable path** that any tool opens by path. That is the only tool-agnostic interface — every editor and multiplexer opens a directory.

**Core rule:** Never use Claude Code's native worktree feature (`--worktree`, `EnterWorktree`) for this. It places the worktree under `.claude/worktrees/` and moves only Claude's own process cwd, so no external tool (editor, lazygit, cmux, tmux) can discover it. Use plain `git worktree add` instead.

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

### 4. Project setup

Run the repo's setup in the new worktree so it's ready to build/test. If the repo ships its own setup skill or script, use that instead of hand-rolling. Otherwise detect the ecosystem:

- **Node monorepo (pnpm/npm/yarn):** install deps, then build the shared/workspace packages the app depends on — a fresh worktree has no built `dist/`.
- **Auto-detect otherwise:** `Cargo.toml`→`cargo build`; `go.mod`→`go mod download`; `requirements.txt`/`pyproject.toml`→pip/poetry install.

### 5. Report the path — stay tool-agnostic

Print the worktree path and the open commands, and stop. Do **not** launch or assume any specific tool.

```
Worktree ready: <PATH_WT>   (branch <BRANCH>)

Open it in:
  editor    cursor <PATH_WT>   |  code <PATH_WT>  |  zed --add <PATH_WT>  |  nvim <PATH_WT>
  git UI    lazygit -p <PATH_WT>
  terminal  cmux <PATH_WT>     |  tmux new-window -c <PATH_WT>
```

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
