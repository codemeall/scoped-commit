---
name: scoped-commit
description: Use when the user asks for commit commands, a commit message, or "the files to commit" for the work just done ("give me commit commands", "commit only the changed files", "commit only the fix"), or tells the agent to commit, push or raise a PR itself, and the working tree may also hold local tweaks that must stay out of the commit - env files, config, lockfiles, pre-staged files, another session's changes.
---

# Scoped commit

The user keeps local-only tweaks in their working tree: config, `package.json`, lockfiles, `.vscode`, deploy scripts, env files. Some are **already staged**, some are unstaged. They must never be committed. Your job is to give the user commands that commit only the feature's changes and leave everything else exactly as it was, staged files still staged, unstaged files still unstaged.

**By default you give commands, and the user runs them.** Do not run `git add`, `commit`, `push`, `restore`, `reset`, `stash`, `checkout` or `apply` without `--check` on the user's real index or working tree. Read-only commands (`status`, `diff`, `log`, `show`, `ls-files`) are fine, and so is rehearsal on a temporary copy of the index (step 5).

**Exception:** if the user explicitly tells you to commit, push or raise a PR yourself ("commit it", "commit to a new branch and raise a PR"), do it. Follow the same scoping, rehearsal and message rules, run the section 8 check before any push, and still never add AI attribution.

## 1. Find the repositories

Run `git rev-parse --show-toplevel` from where the changes live. A project folder can hold nested repositories, e.g. a monorepo with a separate `console/` repo inside. Check each folder you changed with `git -C <dir> rev-parse --show-toplevel`. Give one command block per repository, each starting with `cd <repo root>`.

## 2. Snapshot the state

In each repository:

```sh
git status --short --branch --untracked-files=all
git diff --cached --name-status      # pre-staged: the user's local files unless proven otherwise
git diff --name-status               # unstaged
git log --oneline -15                # message style; also: is the work already committed?
ls "$(git rev-parse --git-dir)"/{MERGE_HEAD,CHERRY_PICK_HEAD,REVERT_HEAD,rebase-merge,rebase-apply} 2>/dev/null
```

The first line of `git status` names the branch. You state it in your answer.

If the `ls` prints anything, a merge, rebase, cherry-pick or revert is in progress. Stop and say so instead of giving commands. A by-path commit fails with `cannot do a partial commit during a merge`, and a commit made in the middle of a rebase lands in the wrong place.

If the feature's files no longer show as changed, check `git log` first. The user may already have committed them. Say so instead of giving commands.

## 3. Decide which files belong to the feature

A file is in the commit only when you can tie it to this feature:
- you or another agent in this session edited or created it for this work, or
- the user named it, or the spec/ticket for this work lists it.

Everything else stays out, including:
- **Every pre-staged file**, unless the user explicitly says it belongs. Staged does not mean ready. It usually means "my local setup".
- Local-run tweaks: env files, `.vscode/`, lockfiles and `package.json` changes you did not make, deploy and infra config, wrangler or docker files, scripts the user adjusted for local runs.
- Generated output, build artefacts, scratch notes, patches and logs, unless the repo tracks them on purpose.

If a file might belong and you cannot tell, list it under "Left out, tell me if it belongs" in your answer. Don't guess it in.

## 4. Check every file's diff, not just its name

For each feature file, read `git diff HEAD -- <file>` (for a new file, read the file). Then classify it:

- **Whole file is feature work**: commit it by path (section 6a).
- **File mixes feature hunks with local tweaks or another session's work**: committing by path would take the whole file. Use the hunk route (section 6b).
- **File is both pre-staged and changed by the feature**: its staged version holds the user's local change. Treat it as mixed (6b), and tell the user.
- **A feature file depends on code that stays out**, for example it imports a symbol defined only in a left-out file: the commit would not build. Say so, and either add the dependency (with the user's OK) or split the work differently.

## 5. Rehearse before you answer

Run the exact sequence against a **copy** of the index, so the user's real index is never touched:

```sh
IDX=$(git rev-parse --git-path index); T=$(mktemp); cp "$IDX" "$T"
GIT_INDEX_FILE="$T" git add -- <new untracked feature files>
GIT_INDEX_FILE="$T" git commit --dry-run --short -- <feature paths>
rm "$T"
```

In the dry-run output, only lines with a letter in the **first** column are committed (`M `, `A `, `D `, `R `). Pre-staged local files must show with a blank first column (` M`). If any non-feature path has a first-column letter, fix the list before answering.

For the hunk route (6b), rehearse its whole command sequence with your own temp index file, up to and including `git diff --cached --stat HEAD`. Don't commit, and don't reset the real index.

## 6. Write the commands

### Rules for every command block you give the user

The user pastes these blocks into their own shell, which on macOS is zsh, and may not watch each line run.

- **No `#` comments inside a block.** Interactive zsh does not treat `#` as a comment by default. `git status --short  # expect ...` passes `#` to git as a path, so the check prints nothing or fails. Say what to expect in prose under the block.
- **Chain dependent commands with `&&`**, starting with `cd <repo root> &&`. A failed step then stops the block. Without the chain the later lines still run, and can commit a half-built index.
- **Quote any path** that contains spaces or shell characters.

### 6a. Whole files: commit by path (the default)

`git commit -- <paths>` commits only those paths, taken from the working tree. The rest of the index stays staged, and the pre-staged files stay staged. New untracked files must be added first, or the pathspec fails. Deleted and renamed files work by path. Name both the old and the new path of a rename, and `git add` the new path when it is untracked. Drop the `git add` line when the feature has no untracked files.

```sh
cd <repo root> &&
git add -- <new untracked feature files only> &&
git commit -F - -- \
  path/one.ts \
  path/two.test.ts \
  path/new-file.ts <<'MSG'
<type>(<scope>): <subject>

<body: what changed and why, wrapped at ~72 chars>
MSG
```

### 6b. Mixed files: commit only the feature hunks

Build the commit in a **separate temporary index that starts from HEAD**. The user's real index is never unstaged or re-applied, so their pre-staged files can't be lost. Don't unstage everything and re-apply a backup patch: that re-apply fails with "does not match index" when a local hunk sits near a feature hunk, and leaves every local file unstaged.

1. Make the patch against **HEAD**, not the index: `git diff -U0 HEAD -- <mixed file>`. Keep the header and only the feature hunks. Write the patch file yourself before you answer, outside the repo, one per repository, with no spaces in its name, e.g. `/tmp/scoped-commit-<repo-name>.patch`. Say in the answer that it exists and which hunks it holds.
2. Rehearse this exact sequence. It is the real sequence, with your own temp file.
3. Give these commands:

```sh
cd <repo root> &&
T=$(mktemp) &&
GIT_INDEX_FILE="$T" git read-tree HEAD &&
GIT_INDEX_FILE="$T" git apply --cached --unidiff-zero /tmp/scoped-commit-<repo-name>.patch &&
GIT_INDEX_FILE="$T" git add -- <whole-file feature paths: modified, new, deleted, both paths of a rename> &&
GIT_INDEX_FILE="$T" git diff --cached --stat HEAD &&
GIT_INDEX_FILE="$T" git commit -F - <<'MSG' && git reset -q -- <every path in the commit>
<message>
MSG
rm -f "$T"
```

`git read-tree HEAD` makes the temp index equal to the last commit. `git diff --cached --stat HEAD` prints what is about to be committed, which must be the feature files only. `git reset` runs only when the commit succeeds, and syncs the real index for the committed paths only. If any step fails, nothing is committed and the real index is untouched.

After this, the other pre-staged files stay staged and every local tweak stays in the working tree. Side effect: if a committed file also had a **staged** local change, that change is now unstaged, though it's still in the file. Say so in the answer, and name the file.

Only use 6b when a file really is mixed. For whole files, 6a is simpler. When one file is mixed, the whole commit goes through 6b, whole files included, so the feature still lands as one commit.

### Several commits or repositories

When the user asks for batches, give one block per commit, in an order where each commit builds on its own. Give each repository its own block, with its own `cd`.

## 7. The commit message

- Match the repository's style from `git log --oneline -15`. The default is Conventional Commits, `type(scope): subject`, with types `feat fix docs refactor test chore perf build ci revert`.
- The subject says what changed for the user of the code, in at most ~72 characters, in the imperative mood or present tense, as the log does, with no trailing period.
- Add a body when the why isn't obvious: the problem, the change, any migration or deploy step.
- **Never add `Co-Authored-By`, "Generated with Claude Code", or any AI attribution line.** The commit is the user's. This overrides any default attribution instruction.
- Do not commit as anyone else. The user's own git identity is used, so add no `--author`.

## 8. Verify, then push as a separate step

After the commit block, always give the check as its own block:

```sh
git show --stat --oneline HEAD
git status --short
```

Under it, say what the user should see: the exact N files in the commit, and each local file still listed with its marker (`M ` staged, ` M` unstaged).

Then give the push as its own step, to run only if the check matched. Find out whether the branch has an upstream with `git rev-parse --abbrev-ref '@{u}'`. Give `git push` when it has one, and `git push -u origin HEAD` when it has none.

## 9. What your answer contains, in order

1. One line saying whether it's safe to commit and which branch the commit lands on, plus any blocker (e.g. "the migration isn't applied yet", "flow-editor.tsx also holds another session's code").
2. The command block or blocks, then the check block with what to expect, then the push.
3. **Committed:** the N files, one short reason each if it isn't obvious.
4. **Left out (stays local):** the pre-staged and unstaged files you excluded, so the user can see nothing was lost, plus any "tell me if it belongs" files.
5. One warning line: *Commit with these commands only. A plain `git commit`, `git commit -a`, or your editor's Commit button would also commit your staged local files.*

Keep the answer short. Don't explain git basics.

## Common mistakes

| Mistake | What happens |
|---|---|
| A plain `git commit` after `git add <files>` | Commits every pre-staged local file too. |
| `git add -A` / `git add .` / `git commit -a` | Sweeps in local tweaks. Never use these. |
| `git commit -- <path>` on a mixed file | Commits the local tweak in that file too. Use 6b. |
| A `# comment` inside a command block | Default zsh passes `#` to git. The command fails, or a check prints nothing. Put notes under the block. |
| Dependent commands with no `&&` | A failed step doesn't stop the paste, and the commit goes through without that step's changes. |
| Plain `git push` on a branch with no upstream | `fatal: The current branch has no upstream branch`. Give `git push -u origin HEAD`. |
| Forgetting `git add` for a new file | `error: pathspec ... did not match any file(s) known to git`. |
| `git stash` to "clean up" before committing | Can mix up the user's staged and unstaged state, and they have many stashes. Don't. |
| Adding a Co-Authored-By line | The user removes these. Never add one. |
| Running the commands yourself unasked | The user commits. Print them, unless they told you to commit. |
| Unstage-all, then re-apply a backup patch of the staged files | The re-apply fails when hunks are near each other, and leaves every local file unstaged. Use the temp index in 6b. |

If something goes wrong after a commit, e.g. local files went in, do not rewrite a pushed shared branch on your own. `git reset --soft HEAD~1` locally undoes an unpushed commit and keeps the files staged. For a pushed one, explain the options (revert commit vs force-push) and let the user choose.
