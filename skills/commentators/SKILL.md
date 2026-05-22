---
name: commentators
description: Runs commentators workflow for planner/developer/security/qa annotations plus docs/commentators reviews in Claude or Codex.
version: 0.1.0
tags:
  - code-review
  - annotations
  - multi-agent
  - documentation
  - security
  - qa
agents:
  - claude
  - codex
---

# commentators

A team-based skill that analyzes code from four perspectives (planning / development / security / QA) and produces two layers of annotation:

- **In source** — one short line per role (KDoc, Javadoc, JSDoc, docstrings, etc.), each prefixed with a role tag. Easy to skim while reading code.
- **In `docs/commentators/`** — a mirrored markdown tree under the project's `docs/` directory, with the full multi-paragraph analysis per symbol per role. This is what future agent sessions read when they need the deeper rationale.

## Invocation

```
/commentators                # scope=all (default)
/commentators all            # all source files under the git root
/commentators changed        # only files changed on the current branch
/commentators <path>         # a specific file or directory
/commentators roles=planner,dev,security,qa   # override the role set (optional)
```

## Core Principles

1. **Two-layer output** — Every annotation produces both an in-source one-liner (≤ 100 chars) and a detailed section in a mirrored markdown file under `docs/commentators/`. The source stays skimmable; the rationale stays discoverable.
2. **Idempotent** — If a symbol already has a one-liner with a given role's prefix (e.g. `{Planner}`) in the source, that role skips both layers for that symbol.
3. **Sequential per file** — Within one file, exactly one role edits at a time, in order: planner → developer → security → qa. This prevents edit conflicts on both the source and the detail doc.
4. **No auto-commit** — The skill only edits files. The user reviews the diff and commits manually.
5. **Project conventions win** — If the project's root `AGENTS.md` defines a role-prefix convention, follow it. If not, check `CLAUDE.md` if it exists. Otherwise fall back to English defaults.
6. **Language-aware comment syntax** — Pick the doc-comment style based on file extension.
7. **Runtime-neutral output** — Claude and Codex may orchestrate agents differently, but the source comments and `docs/commentators/` files must have the same format.

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
- Detail-doc root: `{git-root}/docs/commentators/`. This is where per-file detail markdown lives. The directory is created lazily on first write.

### 3) Decide role prefixes (in priority order)

Read the project root's agent instruction files and decide which prefix set to use. `AGENTS.md` always has priority over `CLAUDE.md`.

1. **Project rule present in `AGENTS.md`** — If `AGENTS.md` defines `{...}`-style role prefixes, use those.
   - Match sections like "주석 규칙", "Comment rules", or any usage of `{Planner}` / `{기획}` / etc.
   - If Korean role keywords (기획/개발/보안/QA) are defined, use the Korean prefix set.
2. **Fallback rule present in `CLAUDE.md`** — If `AGENTS.md` has no matching rule and `CLAUDE.md` exists with `{...}`-style role prefixes, use those.
   - Apply the same matching rules as above.
3. **No rule (default)** — Use English prefixes:
   ```
   {Planner}    — planner
   {Developer}  — developer
   {Security}   — security
   {QA}         — qa
   ```
4. **Argument override** — If `roles=...` is given, it overrides the above.

Pass the resolved prefix map to every Claude role agent, Codex worker, or serial fallback pass.

### 4) Set up the runtime

Select one orchestration mode from the tools available in the current agent runtime.

#### Claude team mode

Use this when Claude team tools are available.

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

#### Codex worker mode

Use this when Codex multi-agent tools such as `multi_agent_v1.spawn_agent`, `wait_agent`, and `send_input` are available.

- Use Codex subagents only when the user explicitly invokes `commentators` or otherwise asks for team/subagent/multi-agent/parallel annotation work. If the skill triggers from a generic "annotate from multiple perspectives" request without explicit delegation, use serial fallback in Codex.
- Spawn Codex subagents with `agent_type="worker"` and omit model overrides unless the user requested a specific model.
- Do not use `~/.claude/teams`, `TeamCreate`, `Agent`, or `SendMessage` in Codex.
- Prefer file-owner workers over role-owner workers: each Codex worker owns a disjoint set of source files and their matching detail-doc files, then performs planner → developer → security → qa sequentially inside each owned file.
- Do not assign four Codex workers to the same source file by role. That creates same-file merge conflicts and violates disjoint write ownership.
- Delay spawning Codex workers until the target file list is known in steps 5-7.

If neither Claude team tools nor Codex multi-agent tools are available, run the same 4-role workflow serially in the main agent.

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
- Claude team mode: leave `owner` unset at the task level — team-lead coordinates roles internally for each file.
- Codex worker mode: partition target files into disjoint batches. Each worker owns the source files in its batch plus only their matching `docs/commentators/<path>.md` files.

### 8) Process files

For each file, compute the detail-doc path:
`{git-root}/docs/commentators/{relative-source-path}.md`.

The `.md` is **appended** to the original filename (not replacing the extension), so `Foo.kt` → `docs/commentators/.../Foo.kt.md` and `Foo.java` → `docs/commentators/.../Foo.java.md` don't collide.

#### Claude team mode

For each file, run the following in strict order:

1. Send a message to **planner** via `SendMessage`. The briefing must include:
   - The source file path.
   - The detail-doc path.
   - "For each symbol where planning intent is worth capturing: write **one short line** (≤ 100 chars) starting with `{Planner}` in the source's doc-comment style, AND append a detailed `### {Planner}` section under that symbol's heading in the detail-doc file. Skip symbols that already have a `{Planner}` line in source. Report when done."
2. Wait for planner's completion report.
3. Repeat for **developer**, then **security**, then **qa** — each with their own prefix.
4. Once all four roles finish, mark the file's task `completed` and move on.

Role agents use `Read` + `Edit` + `Write`. They edit the source file (existing) and read/edit/write the detail-doc file (creating it the first time). Team-lead does not validate content — only orchestrates the sequence.

#### Codex worker mode

After building the target file list, spawn at most `min(4, target-file-count)` workers. For small single-file runs, one worker is enough.

Each worker prompt must include:

- "You are not alone in this codebase. Do not revert edits made by others; adapt to already-present changes."
- The project root path.
- The worker's exact owned source file list.
- The matching detail-doc path for each owned source file.
- The resolved role prefix map.
- The comment-writing rules (see section below).
- "For each owned file, process roles in this exact order: planner, developer, security, qa. For each role, add only that role's in-source one-liner and matching detail-doc subsection. Skip any role already present for a symbol. Report counts per role and list every file path changed."

Codex workers must only edit their owned source files and matching detail-doc files. The main agent initializes `docs/commentators/README.md` and root instruction-file pointers, waits for workers to finish, reviews/integrates returned changes, and then writes the final report.

#### Serial fallback

If no usable team/subagent tools exist, the main agent processes each file itself. Within each file, run planner → developer → security → qa in strict order and follow the same write rules.

### 9) Comment-writing rules (shared by all role agents/workers)

Every role agent, Codex worker, and serial fallback pass uses these rules. There are **two layers** to write per symbol: the in-source one-liner, and the detail-doc section.

#### 9a) In-source one-liner

- **Placement** — Immediately above the declaration (function, class, property, significant block), using the language's doc-comment convention.
- **Form** — Exactly **one sentence**, ≤ 100 characters, starting with the role's prefix. No multi-sentence explanations here — those go in the detail doc.
  - Kotlin: `/** {Planner} QR fallback to manual entry keeps the booking flow unblocked. */`
  - TypeScript: `/** {Security} Token kept in memory only; persisting expands breach radius. */`
  - Python: `"""{QA} Empty list must return None, not raise."""`
- **Hard length cap (≤ 100 chars)** — The one-liner is a hard cap, not a soft target. Count characters of the content after the prefix. If the rationale won't fit, **trim it ruthlessly** (drop examples, drop function names, drop hedging) — the full reasoning belongs in the `.md`, not the source. Do NOT split into two `*` lines under the same role to dodge the cap; that produces multi-line role entries which downstream tools (extractors, summarisers) read as a single role-line.
- **Comment-closer ban (no `*/` inside C-style block comments)** — When you write inside a `/** ... */` (or `/* ... */`) doc block, the content must NOT contain the substring `*/`. A literal `*/` anywhere in the body — even quoted, even inside backticks, even as a reference to another piece of code (e.g. `/* eslint-disable ... */`) — closes the outer block prematurely and breaks compilation. If you need to mention such a marker, paraphrase it (e.g. write "ESLint sort-keys disable directive" instead of pasting the literal `/* eslint-disable sort-keys */`). This rule does not apply to languages whose doc-comments don't use `/* */` (e.g. Python `"""`, Ruby `#`, Go `//`).
- **Multiple roles on the same symbol** — Combine into a single doc block, one line per role, in role order:
  ```kotlin
  /**
   * {Planner} QR fallback to manual entry keeps the booking flow unblocked.
   * {Developer} Sealed result type makes the fallback branch impossible to forget.
   * {Security} Validate scanned payload length before parsing.
   * {QA} Test with malformed and over-length QR payloads.
   */
  ```
- **Perspective specificity** — Don't restate *what* the code does. Capture *why / how to verify* from the role's viewpoint:
  - planner: user scenarios, rationale for the feature's existence, priority justification
  - developer: architectural or pattern choices, technical trade-offs
  - security: threat model, security assumptions, latent vulnerabilities, mitigations
  - qa: test points, edge cases, regression risks
- **Idempotency**:
  - If the same role's prefix already appears in the doc block for this symbol, **skip** (both source and detail doc).
  - If only other roles' prefixes are present, **append** the new role's line inside the existing doc block.
- **Skip trivial symbols** — Getters/setters, overridden `toString`, plain DTO fields, etc. Don't force a comment onto every symbol.
- **Never modify existing code** — Only add or extend comments. Do not touch logic, signatures, imports, or formatting.

#### 9b) Detail-doc section

- **File path** — `{git-root}/docs/commentators/{relative-source-path}.md` (the `.md` is appended, see §8).
- **First write to a detail file** — Create the file with the header template from §9c. Subsequent writes only edit existing sections or append new ones.
- **One section per annotated symbol**, identified by `## {symbol-signature} (line {N})`. Use the symbol's declaration line at write time; if the line later shifts, that's fine — the heading is the durable identifier.
- **One sub-section per role** under that symbol: `### {Planner}`, `### {Developer}`, `### {Security}`, `### {QA}`. A role writes only its own sub-section.
- **Length** — A few sentences to a short paragraph per role. Cover the *why*, the trade-offs, the assumptions — anything that didn't fit in the one-liner.
- **Role agents must not delete or rewrite another role's sub-section.** They only add or edit their own.

#### 9c) Detail-doc file template

When a role agent, Codex worker, or serial fallback pass first creates `docs/commentators/{relative-source-path}.md`, it uses this skeleton (filling in the source path):

```markdown
# {filename}

Source: `{relative-source-path}`

> Multi-role annotations from the `commentators` skill. Each `##` heading is a symbol; each `###` heading under it is one role's detailed perspective. The matching one-liner in the source file is a pointer to the section here.

---
```

Then under that header, sections are appended as roles annotate symbols:

```markdown
## class LoginViewModel (line 12)

### {Planner}
Booking flow can't tolerate a hard stop at QR scan: 12% of sessions fail in low-light, and the cost of dropping them is higher than the cost of a manual-entry path…

### {Developer}
The viewmodel owns the fallback branch instead of the view because…

### {Security}
…

### {QA}
…

## fun authenticate(username: String, password: String) (line 28)

### {Planner}
…
```

Symbols are listed in source order. If a role is annotating an existing symbol (other roles already wrote their sub-sections), it appends its own `### {Role}` section under that `##` heading without reordering or rewriting the others.

### 10) Initialize the `docs/commentators/` tree (once per run, lazily)

The first time any role agent, Codex worker, or serial fallback pass writes a detail file in this run, ensure the project root has:

- `docs/commentators/README.md` — explaining the layout so future agent sessions can pick it up automatically. Use this template (only create if it doesn't already exist):

  ```markdown
  # docs/commentators/

  Detailed multi-role annotations for source files in this repository, written by the `commentators` skill.

  Layout mirrors the source tree. For a source file at `<path>`, its detailed annotations live at `docs/commentators/<path>.md`.

  Each markdown file has one `##` section per annotated symbol and one `### {Role}` sub-section per role (Planner / Developer / Security / QA). The matching one-line comments inside the source file are pointers into here.

  When working on a source file with `{Planner}` / `{Developer}` / `{Security}` / `{QA}` one-liners, also read the corresponding file under `docs/commentators/` for the full rationale.
  ```

- A pointer in the project's root `AGENTS.md` (create the file if it doesn't exist; otherwise append a section if not already present). If a root `CLAUDE.md` already exists, append the same pointer there if it is not already present; do not create `CLAUDE.md`.

  ```markdown
  ## Multi-role annotations

  Detailed per-symbol rationale lives under `docs/commentators/`, mirroring the source tree (e.g. `src/auth/Login.kt` → `docs/commentators/src/auth/Login.kt.md`). When you see `{Planner}` / `{Developer}` / `{Security}` / `{QA}` one-line comments in a source file, read the matching detail file for the full reasoning.
  ```

Team-lead/main agent does both of these (not role agents or Codex workers) so the wording stays consistent.

### 11) Optional build check

Skip for small, targeted runs. For large `scope=all` runs over a language that has a fast check:

- Kotlin/Java (Gradle): try `./gradlew compileDebugKotlin` or `./gradlew compileKotlin`.
- TypeScript: try `tsc --noEmit` if a `tsconfig.json` is present.
- If the command is missing or fails for unrelated reasons, ignore and note it in the final report.

### 12) Final report

Team-lead/main agent reports to the user:
- Number of source files processed
- Approximate count of one-liners added per role (based on role-agent, worker, or serial pass reports)
- Number of detail-doc files created vs. updated under `docs/commentators/`
- Whether `docs/commentators/README.md`, the `AGENTS.md` pointer, and any existing `CLAUDE.md` pointer were created or already present
- Summary of skipped files/symbols
- Build-check result (if run)
- **Next steps reminder** — "No commits were made. Review `git diff` and commit when you're ready. The `docs/commentators/` tree is also a new (or updated) set of files in your working tree."

## Edge cases

- **Not a git repo** — Only allow `scope=<path>`. Reject `changed`.
- **Zero target files** — Report and exit. Do not create the `docs/commentators/` tree.
- **No `AGENTS.md` or `CLAUDE.md`** — Use English default prefixes for in-source comments. Team-lead creates a minimal `AGENTS.md` containing only the "Multi-role annotations" pointer (§10) when the first detail file is written.
- **Both `AGENTS.md` and `CLAUDE.md` present** — `AGENTS.md` wins for role-prefix conventions. `CLAUDE.md` is only used as a fallback when `AGENTS.md` has no matching rule.
- **Only unsupported extensions present** — Report "no supported files" and exit.
- **Agent edit fails** — Skip that role or worker-owned file and continue; note the failure in the report. If a role wrote the source one-liner but failed on the detail doc (or vice-versa), report the inconsistency so the user can re-run.
- **Detail file path collides with an existing non-doc file** — If `docs/commentators/<path>.md` already exists but is not a commentators-format file (no `Source:` header), abort that file and report; do not overwrite.
- **Existing Claude team has a different prefix convention than this run** — Either spawn a new team with a hash suffix on the team name, or ask the user whether to reuse the existing team.

## Guardrails

- **Role agents and Codex workers use Read + Edit + Write only for the two layers** — the source file (Edit only; never create source files) and the matching detail-doc file under `docs/commentators/` (Read/Edit/Write). They must not run `Bash`, git commands, builds, or touch any file outside those two paths.
- **Team-lead/main agent alone manages `docs/commentators/README.md`, the root `AGENTS.md` pointer, and any existing root `CLAUDE.md` pointer** — role agents and Codex workers do not write to these.
- **Claude only: always pass `team_name` when spawning role agents** — so the caller (team-lead) remains the team lead.
- **Codex only: keep worker write sets disjoint** — never assign separate workers to edit the same source file or detail-doc file.
- **No commits, pushes, or PRs** — This skill's scope ends at file edits.
- **Skip sensitive files** — Extension filters already exclude most secrets, but explicitly skip any file whose path contains `secret` or `credential`. Their detail-doc paths under `docs/commentators/` must also not be created.

## Termination

- Ends when every target file has completed the 4-role loop.
- Ends if the user cancels.
- Ends immediately if the target file list is empty.

**Claude:** Keep the team alive on exit so subsequent invocations can reuse it. Only shut down teammates (via `SendMessage` with `shutdown_request`) when the user explicitly asks to clean up the team.

**Codex:** Do not create persistent team state. Close spawned workers when they are no longer needed, after their changes have been reviewed/integrated and their final reports captured.
