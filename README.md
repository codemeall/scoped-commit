# scoped-commit

An [Agent Skill](https://agentskills.io) for AI coding agents (Claude Code, Codex, Cursor and others). When you ask for commit commands, it gives you paste-able git commands that commit **only the files the feature changed**. Your local tweaks stay out of the commit, whether they are staged or unstaged: env files, `.vscode`, `package.json`, lockfiles, deploy config.

- Commits by path (`git commit -- <files>`), so files you already staged stay staged and are left out.
- For a file that mixes feature changes with local tweaks, commits only the feature's hunks.
- Rehearses the commands on a copy of your index before showing them.
- Writes a Conventional Commits message in your repo's style, with no `Co-Authored-By` or AI attribution line.
- Prints the commands by default, and you run them. Push is a separate step after a check. If you explicitly tell the agent to commit or raise a PR, it does that with the same rules.

## Install

Any agent that supports Agent Skills:

```sh
npx skills add codemeall/scoped-commit                          # asks which agents and where
npx skills add codemeall/scoped-commit -g -a claude-code -y     # Claude Code, all projects, no prompts
```

Update later with `npx skills update scoped-commit`.

Claude Code, as a plugin:

```
/plugin marketplace add codemeall/scoped-commit
/plugin install scoped-commit@scoped-commit
```

Manually: copy or symlink `skills/scoped-commit` into `~/.claude/skills/`.

Use one install method per machine. The plugin and the npx skill together give you two copies.

## Use

After the agent finishes a change:

```
/scoped-commit
```

or just ask: "give me commit commands for only the changed files".

### Things you can say

| You say | You get |
|---|---|
| `/scoped-commit` | Commands and a message for the work just done |
| "give me the commit message and command for the fix" | The same, scoped to the fix |
| "commit only the console changes" | One block, for the `console/` repo only |
| "give me commands to commit in two batches, backend then frontend" | Two commits, each of which builds on its own |
| "commit it to a new branch feature/x and raise a PR" | The agent runs it for you, with the same rules |

### Example

Your working tree before:

```
$ git status --short
M  .vscode/settings.json          # staged local tweak
M  package.json                   # staged local tweak
 M .env.local                     # unstaged local tweak
 M src/billing/invoice.ts         # the fix
 M src/billing/config.ts          # the fix on line 12, plus a local URL on line 40
?? src/billing/invoice.test.ts    # new test for the fix
```

You type `/scoped-commit`. The agent replies:

> Safe to commit. `config.ts` mixes the fix with your local URL, so only the fix lines go in.
>
> ```sh
> cd ~/code/shop
>
> # feature patch (made by the agent): /tmp/scoped-commit-feature.patch, line 12 of config.ts only
> T=$(mktemp)
> GIT_INDEX_FILE=$T git read-tree HEAD
> GIT_INDEX_FILE=$T git apply --cached --unidiff-zero /tmp/scoped-commit-feature.patch
> GIT_INDEX_FILE=$T git add -- src/billing/invoice.ts src/billing/invoice.test.ts
> GIT_INDEX_FILE=$T git diff --cached --stat HEAD     # expect 3 files
>
> GIT_INDEX_FILE=$T git commit -F - <<'MSG'
> fix(billing): round invoice totals per line, not per invoice
>
> Totals were rounded once at the end, so multi-line invoices could be
> off by a cent against the line items. Adds a regression test.
> MSG
> rm -f "$T"
> git reset -q -- src/billing/invoice.ts src/billing/invoice.test.ts src/billing/config.ts
>
> git show --stat --oneline HEAD     # expect: invoice.ts, invoice.test.ts, config.ts
> git status --short                 # expect: M .vscode/settings.json, M package.json,
>                                    #         M .env.local, M config.ts (local URL)
> ```
>
> Then, if the check matched: `git push`
>
> **Committed:** `invoice.ts`, `invoice.test.ts`, line 12 of `config.ts`.
> **Left out (stays local):** `.vscode/settings.json`, `package.json` (still staged), `.env.local`, and the local URL in `config.ts`.
>
> Commit with these commands only. A plain `git commit`, `git commit -a`, or your editor's Commit button would also commit your staged local files.

When no file is mixed, which is the usual case, the commands are shorter:

```sh
git add -- src/billing/invoice.test.ts           # new files only
git commit -F - -- src/billing/invoice.ts src/billing/invoice.test.ts <<'MSG'
fix(billing): round invoice totals per line, not per invoice
MSG
```
