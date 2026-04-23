# skills

Personal collection of [Claude Code](https://claude.com/claude-code) skills, installable via the [Skills CLI](https://skills.sh/).

## Installation

```bash
npx skills add bindon/skills
```

This installs all skills in this repo to `~/.agents/skills/` and symlinks them into `~/.claude/skills/`.

Update later with:

```bash
npx skills update
```

## Available Skills

### commentators

Spawns a 4-role team (planner / developer / security / qa) that analyzes source code from each perspective and adds role-prefixed doc comments (KDoc / Javadoc / JSDoc / docstring / etc., auto-detected by file extension).

**Usage**

```
/commentators              # scope=all (default: all source files under git root)
/commentators changed      # only files changed on the current branch
/commentators <path>       # a specific file or directory
/commentators roles=planner,dev,security,qa   # override role set
```

**What it does**

1. Creates a team `{project-name}-commentators` (or reuses an existing one).
2. Spawns 4 role agents: `planner`, `developer`, `security`, `qa`.
3. Reads the project's `CLAUDE.md` to pick up any role-prefix convention (falls back to English defaults `{Planner}/{Developer}/{Security}/{QA}` if none).
4. Walks the target files and processes them **one at a time**: planner → developer → security → qa, each adding a language-appropriate doc comment from their perspective.
5. Idempotent: skips symbols that already have that role's prefix.
6. No auto-commit — review `git diff` and commit yourself.

See [`skills/commentators/SKILL.md`](skills/commentators/SKILL.md) for the full spec.

## License

MIT
