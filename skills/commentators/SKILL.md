---
name: commentators
description: Spawns a 4-role team (planner / developer / security / qa) that analyzes source code from each perspective and adds role-prefixed doc comments. Use when the user wants code annotated from multiple viewpoints — planning intent, development rationale, security concerns, and QA test points — across a project. Trigger phrases include "commentators", "annotate from multiple perspectives", "multi-role comments", "역할별 주석", "팀으로 주석 달기", "4관점 리뷰 주석".
---

# commentators

A team-based skill that analyzes code from four perspectives (planning / development / security / QA) and adds language-appropriate doc comments (KDoc, Javadoc, JSDoc, docstrings, etc.), each prefixed with a role tag.

## Invocation

```
/commentators                # scope=all (default)
/commentators all            # all source files under the git root
/commentators changed        # only files changed on the current branch
/commentators <path>         # a specific file or directory
/commentators roles=planner,dev,security,qa   # override the role set (optional)
```

## Core Principles

1. **Idempotent** — If a symbol already has a comment with a given role's prefix (e.g. `{Planner}`), that role skips it.
2. **Sequential per file** — Within one file, exactly one role edits at a time, in order: planner → developer → security → qa. This prevents edit conflicts.
3. **No auto-commit** — The skill only edits files. The user reviews the diff and commits manually.
4. **Project conventions win** — If the project's root `CLAUDE.md` defines a role-prefix convention, follow it. Otherwise fall back to English defaults.
5. **Language-aware comment syntax** — Pick the doc-comment style based on file extension.

## Execution Order

When the skill is invoked, proceed in this exact order.

### 1) Parse arguments

- `scope` default: `all`
- `scope` ∈ { `all`, `changed`, `<path>` }
- If `roles=...` is present, override the role list for this run.

### 2) Resolve project context

- Run `git rev-parse --show-toplevel` to find the git root. If it fails, use `cwd`.
- Project name: `basename` of the git root (replace whitespace/special chars with `-`).
- Team name: `{project-name}-commentators`.

### 3) Decide role prefixes (in priority order)

Read the project root's `CLAUDE.md` and decide which prefix set to use:

1. **Project rule present** — If `CLAUDE.md` defines `{...}`-style role prefixes, use those.
   - Match sections like "주석 규칙", "Comment rules", or any usage of `{Planner}` / `{기획}` / etc.
   - If Korean role keywords (기획/개발/보안/QA) are defined, use the Korean prefix set.
2. **No rule (default)** — Use English prefixes:
   ```
   {Planner}    — planner
   {Developer}  — developer
   {Security}   — security
   {QA}         — qa
   ```
3. **Argument override** — If `roles=...` is given, it overrides the above.

Pass the resolved prefix map to every role agent.

### 4) Set up the team

- Check if `~/.claude/teams/{team-name}/config.json` exists.
- If not, create the team with `TeamCreate`, then spawn 4 `general-purpose` agents via `Agent`:
  - `name`: `planner`, `developer`, `security`, `qa`
  - `team_name`: the resolved team name
  - `run_in_background`: true
- If the team already exists, reuse it. Inspect the `members` array in the team config and only spawn missing roles.

Each role agent receives a briefing with:
- Project root path
- Their role and prefix (e.g. `planner` → `{Planner}`)
- The comment-writing rules (see section below)
- An instruction to wait for task assignments from team-lead

### 5) Build the target file list

Based on `scope`:

- **`all`** — Source files under the git root. Apply the rules below.
  - Included extensions: `.kt .java .ts .tsx .js .jsx .py .go .rs .swift .cpp .cc .c .h .hpp .cs .rb .php .scala .m .mm`
  - Excluded directories: `build/`, `.gradle/`, `node_modules/`, `dist/`, `target/`, `out/`, `.next/`, `.nuxt/`, `.venv/`, `venv/`, `__pycache__/`, `.idea/`, `.vscode/`, `vendor/`, `Pods/`, `DerivedData/`, `.git/`
  - Test directories (`test/`, `tests/`, `__tests__/`, `spec/`, `androidTest/`) are excluded by default.
  - Generated files (`*.g.kt`, `*.pb.go`, `*.generated.*`) are excluded.
- **`changed`** — `git diff --name-only $(git merge-base HEAD origin/main 2>/dev/null || echo main)...HEAD` plus modified/new files from `git status --porcelain`. Apply the same include/exclude rules.
- **`<path>`** — If the argument is a file, use it. If a directory, recurse and apply the rules above.

If the final file count exceeds **30**, ask the user to confirm before continuing.

### 6) Map extensions to comment styles

| Extension | Style | Format |
|---|---|---|
| `.kt` | KDoc | `/** ... */` block immediately above the declaration |
| `.java` | Javadoc | `/** ... */` block |
| `.ts .tsx .js .jsx` | JSDoc | `/** ... */` block |
| `.py` | Docstring | `"""..."""` as the first statement of the function/class body |
| `.go` | Line comments | `// ...` immediately above the declaration |
| `.rs` | Doc comment | `/// ...` |
| `.swift` | Doc comment | `/// ...` |
| `.cpp .cc .c .h .hpp` | Doxygen | `/** ... */` |
| `.cs` | XML doc | `/// <summary>...</summary>` |
| `.rb` | YARD/RDoc | `# ...` |
| `.php` | PHPDoc | `/** ... */` |
| `.scala` | ScalaDoc | `/** ... */` |
| `.m .mm` | Doc / line | `/// ...` |

### 7) Build the task list

- Create one task per file: `"[{filename}] 4-role annotation"`.
- Leave `owner` unset at the task level — team-lead coordinates roles internally for each file.

### 8) Process files sequentially

For each file, run the following in strict order:

1. Send a message to **planner** via `SendMessage`: "For this file, add a `{Planner}`-prefixed doc comment (in the correct language style) to any symbol where planning intent is worth capturing. Skip symbols that already have a `{Planner}` comment. Report when done."
2. Wait for planner's completion report.
3. Repeat for **developer**, then **security**, then **qa** — each with their own prefix.
4. Once all four roles finish, mark the file's task `completed` and move on.

Role agents edit the file directly with `Read` + `Edit`. Team-lead does not validate content — only orchestrates the sequence.

### 9) Comment-writing rules (shared by all role agents)

Every role agent is given these rules:

- **Placement** — Place the doc comment **immediately above** the declaration (function, class, property, significant block), using the language's convention.
- **Form** — Inside a language-appropriate doc block, write **one or two sentences** that start with the role's prefix and express that role's perspective.
  - Example (Kotlin): `/** {Planner} QR scan failures must fall back to manual entry so users aren't blocked. */`
  - Example (TypeScript): `/** {Security} Token stays in memory only; persisting it would expand the breach radius. */`
  - Example (Python): `"""{QA} Edge case: an empty list should return None, not raise."""`
- **Perspective specificity** — Do not restate *what* the code does. Instead, capture *why / how to verify* from the role's viewpoint:
  - planner: user scenarios, rationale for the feature's existence, priority justification
  - developer: architectural or pattern choices, technical trade-offs
  - security: threat model, security assumptions, latent vulnerabilities, mitigations
  - qa: test points, edge cases, regression risks
- **Symbols that already have a doc block**:
  - If the same role's prefix is already present, **skip**.
  - If only other roles' prefixes are present, **append** a new line inside the existing doc block for this role.
- **Do not annotate trivial symbols** — Getters/setters, overridden `toString`, plain DTO fields, etc., don't need role commentary. Don't force a comment onto every symbol.
- **Never modify existing code** — Only add or extend comments. Do not touch logic, signatures, imports, or formatting.

### 10) Optional build check

Skip for small, targeted runs. For large `scope=all` runs over a language that has a fast check:

- Kotlin/Java (Gradle): try `./gradlew compileDebugKotlin` or `./gradlew compileKotlin`.
- TypeScript: try `tsc --noEmit` if a `tsconfig.json` is present.
- If the command is missing or fails for unrelated reasons, ignore and note it in the final report.

### 11) Final report

Team-lead reports to the user:
- Number of files processed
- Approximate count of comments added per role (based on agent reports)
- Summary of skipped files/symbols
- Build-check result (if run)
- **Next steps reminder** — "No commits were made. Review `git diff` and commit when you're ready."

## Edge cases

- **Not a git repo** — Only allow `scope=<path>`. Reject `changed`.
- **Zero target files** — Report and exit.
- **No `CLAUDE.md`** — Use English default prefixes.
- **Only unsupported extensions present** — Report "no supported files" and exit.
- **Agent edit fails** — Skip that role for the file and continue; note the failure in the report.
- **Existing team has a different prefix convention than this run** — Either spawn a new team with a hash suffix on the team name, or ask the user whether to reuse the existing team.

## Guardrails

- **Role agents use only Read + Edit** — Do not let them run `Bash`, git commands, builds, or create files. Team-lead coordinates everything else.
- **Always pass `team_name` when spawning role agents** — so the caller (team-lead) remains the team lead.
- **No commits, pushes, or PRs** — This skill's scope ends at file edits.
- **Skip sensitive files** — Extension filters already exclude most secrets, but explicitly skip any file whose path contains `secret` or `credential`.

## Termination

- Ends when every target file has completed the 4-role loop.
- Ends if the user cancels.
- Ends immediately if the target file list is empty.

**Keep the team alive** on exit so subsequent invocations can reuse it. Only shut down teammates (via `SendMessage` with `shutdown_request`) when the user explicitly asks to clean up the team.
