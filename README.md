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
npx skills add codemeall/scoped-commit
```

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
