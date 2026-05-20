---
name: atomic-commit
description: Author git commits with messages that follow the seven rules from cbea.ms/git-commit, and reorganize local-only commits so each one is atomic (squash related, split mixed-concern). Use when the user asks to commit, clean up commits, squash, split, rebase interactively, or fix up history before pushing.
---

# atomic-commit

Author and reshape commits so that each one is a single coherent change with a
message that follows the seven rules from <https://cbea.ms/git-commit/>.

A `commit-msg` git hook at `.githooks/commit-msg` enforces the rules
mechanically. This skill exists to help you write messages that pass the hook
the first time and to keep history atomic — one logical change per commit.

## The seven rules (the hook enforces 1–6)

1. Separate subject from body with a blank line.
2. Limit the subject line to ~50 characters (hook hard-fails at 72).
3. Capitalize the subject line.
4. Do not end the subject line with a period.
5. Use the imperative mood — "Add", not "Added"/"Adding"/"Adds".
6. Wrap the body at 72 characters.
7. Use the body to explain **what and why**, not how. The diff already shows
   how. The reader needs to know what problem this solves and why this
   approach.

The imperative-mood test: "If applied, this commit will _<subject>_" must
read naturally.

## When the user asks to commit

1. Run `git status` and `git diff --staged` (and `git diff` for unstaged) to
   understand what's actually changing.
2. Decide whether the staged changes are **one logical change** or several.
   If several, follow [Split overstuffed staged changes](#split-overstuffed-staged-changes)
   before writing any message.
3. For each commit, draft the subject:
   - Imperative verb first: Add, Fix, Remove, Refactor, Move, Rename, Extract,
     Inline, Document, Hoist, Simplify, Tighten, Replace, Wire, etc.
   - ≤50 chars when possible, ≤72 always.
   - Capitalized, no trailing period.
4. Write a body only if the **why** isn't obvious from the diff. Wrap at 72
   chars. Explain the problem, the chosen approach, and any non-obvious
   consequences. Reference issues/PRs in trailers at the bottom
   (`Resolves: #123`, `See: ABC-456`).
5. Commit with a heredoc to preserve formatting:

   ```bash
   git commit -m "$(cat <<'EOF'
   Subject line in imperative mood

   Body paragraph wrapped at 72 chars explaining what problem this
   solves and why this approach. Skip the body for trivially obvious
   changes — the hook does not require one.

   Resolves: #123
   EOF
   )"
   ```

6. If the hook rejects the commit, read the error, fix the message, and
   recommit. Do **not** use `--no-verify`. The rules are there for a reason.

## Split overstuffed staged changes

When `git diff --staged` covers more than one logical change, split before
committing.

1. Unstage everything: `git reset`.
2. For each logical group:
   - `git add -p <files>` (or `git add <file>`) to stage just that group.
   - Commit with a focused message.
3. Verify: `git log --oneline -n <N>` should read like a coherent sequence of
   independent changes.

Signs a commit is too big:
- You can't write a ≤50-char subject without using "and" or "&".
- The body needs more than one paragraph just to enumerate changes.
- The diff touches unrelated subsystems (e.g. handler + unrelated helper
  refactor + dependency bump).

## Reorganize local-only commits

**Only rewrite commits that have not been pushed.** Confirm with
`git log @{upstream}..HEAD` (or `git log origin/main..HEAD` on a fresh branch).
If a commit appears on a remote, leave it alone — rewriting published history
forces other people to recover.

### Squash related commits

When several local commits all belong to one logical change ("Fix typo",
"Fix another typo", "Actually fix the typo"):

```bash
git rebase -i @{upstream}    # or origin/main
```

In the editor:
- Keep the first commit as `pick`.
- Change subsequent related ones to `squash` (keeps their message) or
  `fixup` (discards their message).
- Save and exit. When git opens the combined message editor, write a single
  clean message that follows the seven rules.

The hook will validate the rewritten message. If it rejects, the rebase
pauses — fix the message via `git commit --amend` and `git rebase --continue`.

### Split one commit into several

When one commit bundles unrelated changes:

```bash
git rebase -i @{upstream}
```

Mark the commit `edit`. When the rebase pauses:

```bash
git reset HEAD^                       # unstage everything from that commit
# then for each logical piece:
git add -p <files>
git commit                            # write a focused message
# repeat...
git rebase --continue
```

### Reorder

Move lines around in the rebase todo to reorder. Useful when a later commit
logically belongs before an earlier one (e.g. extract a helper, then use it).

## Anti-patterns to avoid

- **Vague subjects**: "WIP", "Fixes", "Misc cleanup", "Small fixes",
  "Various improvements". The hook rejects these. Say what the commit does.
- **Past tense / gerunds / third-person**: "Added", "Adding", "Adds". Use
  imperative.
- **Subject as a sentence**: "This change fixes the bug where…". Just write
  "Fix bug where…".
- **Body explains how**: "Loops through the slice and calls .Close() on each".
  The diff shows that. Explain *why* you needed to close them — leak?
  ordering? shutdown sequencing?
- **`git commit --no-verify`** to bypass the hook. Don't. If the rule is wrong
  for a specific case (e.g. a trailer line legitimately needs to be long),
  ensure the hook's existing allowances cover it; otherwise propose adjusting
  the hook itself.
- **Rewriting pushed commits**. Once a commit is on the remote, it's frozen.
  Add a follow-up commit instead.
- **Rebasing onto a moving target without confirmation**. `git rebase -i
  @{upstream}` is safe because it touches only your local commits. Anything
  beyond that — confirm with the user first.

## Quick reference: imperative verbs

Use these as starters when you're stuck:

Add, Remove, Delete, Drop, Move, Rename, Extract, Inline, Hoist, Pull,
Push, Fix, Patch, Repair, Restore, Revert, Replace, Swap, Switch,
Refactor, Simplify, Tighten, Clean, Clarify, Reorganize, Reorder,
Reformat, Document, Annotate, Comment, Expose, Hide, Wire, Unwire,
Enable, Disable, Skip, Allow, Forbid, Guard, Validate, Sanitize,
Normalize, Migrate, Port, Upgrade, Downgrade, Bump, Pin, Unpin,
Cache, Invalidate, Throttle, Retry, Time out, Log, Trace, Instrument,
Measure, Profile, Tune, Optimize, Parallelize, Serialize, Lock, Unlock,
Test, Cover, Assert, Mock, Stub.
