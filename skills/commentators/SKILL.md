---
name: commentators
description: Runs commentators workflow producing planner/developer/security/qa commentary as a docs/commentators tree in Claude or Codex. Source files are never modified.
version: 0.2.0
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

A team-based skill that analyzes code from four perspectives (planning / development / security / QA) and writes their commentary as a **separate documentation track** — never as inline source comments.

Think of it as a director's commentary: four commentators narrate the code on their own track, leaving the source itself untouched.

- **In `docs/commentators/`** — a mirrored markdown tree under the project's `docs/` directory, with the full multi-paragraph analysis per symbol per role. This is the single source of commentary, and it is what future agent sessions read when they need the deeper rationale.
- **Source files are read-only.** The skill reads source to decide what is worth commenting on, but never edits, creates, or annotates source files. Developers read the code directly; agents read the matching doc.

## Invocation

```
/commentators                # scope=all (default)
/commentators all            # all source files under the git root
/commentators changed        # only files changed on the current branch
/commentators <path>         # a specific file or directory
/commentators roles=planner,dev,security,qa   # override the role set (optional)
```

## Core Principles

1. **Single docs-only layer** — Every annotation is a section in a mirrored markdown file under `docs/commentators/`. The source tree stays pristine so code diffs show only real code changes; the rationale stays discoverable via the mirrored path.
2. **Source is read-only** — The skill never edits, creates, or comments source files. It only reads them to decide what to comment on. The only files it writes are under `docs/commentators/` plus the README/instruction pointers in §10.
3. **Idempotent** — Idempotency is judged from the doc, not the source: if `docs/commentators/<path>.md` already has a `### {Role}` sub-section under a symbol's `## {signature}` heading, that role skips that symbol.
4. **Sequential per file** — Within one file, exactly one role writes at a time, in order: planner → developer → security → qa. The shared resource is the single detail-doc `.md` file; sequencing prevents edit conflicts on it.
5. **No auto-commit** — The skill only writes doc files. The user reviews the diff and commits manually.
6. **Project conventions win** — If the project's root `AGENTS.md` defines a role-prefix convention, follow it for the `### {Role}` section headers and the team name. If not, check `CLAUDE.md` if it exists. Otherwise fall back to English defaults.
7. **Runtime-neutral output** — Claude and Codex may orchestrate agents differently, but the `docs/commentators/` files must have the same format.

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

Read the project root's agent instruction files and decide which prefix set to use. `AGENTS.md` always has priority over `CLAUDE.md`. The prefix set is used for the `### {Role}` headers in the detail docs.

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
- The commentary-writing rules (see section below)
- An instruction to wait for task assignments from team-lead

#### Codex worker mode

Use this when Codex multi-agent tools such as `multi_agent_v1.spawn_agent`, `wait_agent`, and `send_input` are available.

- Use Codex subagents only when the user explicitly invokes `commentators` or otherwise asks for team/subagent/multi-agent/parallel annotation work. If the skill triggers from a generic "annotate from multiple perspectives" request without explicit delegation, use serial fallback in Codex.
- Spawn Codex subagents with `agent_type="worker"` and omit model overrides unless the user requested a specific model.
- Do not use `~/.claude/teams`, `TeamCreate`, `Agent`, or `SendMessage` in Codex.
- Prefer file-owner workers over role-owner workers: each Codex worker owns a disjoint set of detail-doc files (one per source file in its batch), then performs planner → developer → security → qa sequentially inside each owned detail-doc file.
- Do not assign four Codex workers to the same detail-doc file by role. That creates same-file merge conflicts and violates disjoint write ownership.
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

These source files are read to decide what to comment on; they are never edited.

If the final file count exceeds **30**, ask the user to confirm before continuing.

### 6) Build the task list

- Create one task per file: `"[{filename}] 4-role commentary"`.
- Claude team mode: leave `owner` unset at the task level — team-lead coordinates roles internally for each file.
- Codex worker mode: partition target files into disjoint batches. Each worker owns only the `docs/commentators/<path>.md` files matching the source files in its batch.

### 7) Process files

For each file, compute the detail-doc path:
`{git-root}/docs/commentators/{relative-source-path}.md`.

The `.md` is **appended** to the original filename (not replacing the extension), so `Foo.kt` → `docs/commentators/.../Foo.kt.md` and `Foo.java` → `docs/commentators/.../Foo.java.md` don't collide.

#### Claude team mode

For each file, run the following in strict order:

1. Send a message to **planner** via `SendMessage`. The briefing must include:
   - The source file path (read-only).
   - The detail-doc path.
   - "For each symbol where planning intent is worth capturing: append a detailed `### {Planner}` section under that symbol's `## {signature}` heading in the detail-doc file. Create the heading if it doesn't exist. Skip symbols that already have a `### {Planner}` section. Do not edit the source file. Report when done."
2. Wait for planner's completion report.
3. Repeat for **developer**, then **security**, then **qa** — each with their own prefix.
4. Once all four roles finish, mark the file's task `completed` and move on.

Role agents use `Read` (on the source, read-only) + `Read`/`Edit`/`Write` (on the detail-doc file, creating it the first time). Team-lead does not validate content — only orchestrates the sequence.

#### Codex worker mode

After building the target file list, spawn at most `min(4, target-file-count)` workers. For small single-file runs, one worker is enough.

Each worker prompt must include:

- "You are not alone in this codebase. Do not revert edits made by others; adapt to already-present changes."
- The project root path.
- The worker's exact owned detail-doc file list (and the source file each one mirrors, read-only).
- The resolved role prefix map.
- The commentary-writing rules (see section below).
- "For each owned file, process roles in this exact order: planner, developer, security, qa. For each role, add only that role's `### {Role}` detail-doc sub-section. Skip any role already present for a symbol. Never edit source files. Report counts per role and list every detail-doc path changed."

Codex workers must only edit their owned detail-doc files. The main agent initializes `docs/commentators/README.md` and root instruction-file pointers, waits for workers to finish, reviews/integrates returned changes, and then writes the final report.

#### Serial fallback

If no usable team/subagent tools exist, the main agent processes each file itself. Within each file, run planner → developer → security → qa in strict order and follow the same write rules.

### 8) Commentary-writing rules (shared by all role agents/workers)

Every role agent, Codex worker, and serial fallback pass uses these rules. There is **one layer**: the detail-doc section. Source files are never touched.

#### 8a) Detail-doc section

- **File path** — `{git-root}/docs/commentators/{relative-source-path}.md` (the `.md` is appended, see §7).
- **First write to a detail file** — Create the file with the header template from §8b. Subsequent writes only edit existing sections or append new ones.
- **One section per commented symbol**, identified by `## {symbol-signature}`. Do **not** put line numbers in the heading — the signature is the durable identifier and stays stable when code shifts up or down.
- **One sub-section per role** under that symbol: `### {Planner}`, `### {Developer}`, `### {Security}`, `### {QA}`. A role writes only its own sub-section.
- **Length** — A few sentences to a short paragraph per role. Cover the *why*, the trade-offs, the assumptions, the verification points.
- **Perspective specificity** — Don't restate *what* the code does. Capture *why / how to verify* from the role's viewpoint:
  - planner: user scenarios, rationale for the feature's existence, priority justification
  - developer: architectural or pattern choices, technical trade-offs
  - security: threat model, security assumptions, latent vulnerabilities, mitigations
  - qa: test points, edge cases, regression risks
- **Idempotency**:
  - If the same role's `### {Role}` section already exists for this symbol, **skip**.
  - If only other roles' sections are present under the `## {signature}` heading, **append** the new role's section there.
- **Skip trivial symbols** — Getters/setters, overridden `toString`, plain DTO fields, etc. Don't force commentary onto every symbol.
- **Role agents must not delete or rewrite another role's sub-section.** They only add or edit their own.
- **Never modify source.** If a role notices a bug while reading, it records the concern in its detail-doc section; it does not edit the code.

#### 8b) Detail-doc file template

When a role agent, Codex worker, or serial fallback pass first creates `docs/commentators/{relative-source-path}.md`, it uses this skeleton (filling in the source path):

```markdown
# {filename}

Source: `{relative-source-path}`

> Multi-role commentary from the `commentators` skill. Each `##` heading is a symbol; each `###` heading under it is one role's detailed perspective. Source files are not annotated — this doc is the only commentary track.

---
```

Then under that header, sections are appended as roles comment symbols:

```markdown
## class LoginViewModel

### {Planner}
Booking flow can't tolerate a hard stop at QR scan: 12% of sessions fail in low-light, and the cost of dropping them is higher than the cost of a manual-entry path…

### {Developer}
The viewmodel owns the fallback branch instead of the view because…

### {Security}
…

### {QA}
…

## fun authenticate(username: String, password: String)

### {Planner}
…
```

Symbols are listed in source order. If a role is commenting an existing symbol (other roles already wrote their sub-sections), it appends its own `### {Role}` section under that `##` heading without reordering or rewriting the others.

### 9) Initialize the `docs/commentators/` tree (once per run, lazily)

The first time any role agent, Codex worker, or serial fallback pass writes a detail file in this run, ensure the project root has:

- `docs/commentators/README.md` — explaining the layout so future agent sessions can pick it up automatically. Use this template (only create if it doesn't already exist):

  ```markdown
  # docs/commentators/

  Detailed multi-role commentary for source files in this repository, written by the `commentators` skill.

  Layout mirrors the source tree. For a source file at `<path>`, its commentary lives at `docs/commentators/<path>.md`.

  Each markdown file has one `##` section per commented symbol and one `### {Role}` sub-section per role (Planner / Developer / Security / QA). Source files are not annotated — this tree is the only commentary track.

  When working on a source file, also read the matching `docs/commentators/<path>.md` (if it exists) for the full per-symbol rationale.
  ```

- A pointer in the project's root `AGENTS.md` (create the file if it doesn't exist; otherwise append a section if not already present). If a root `CLAUDE.md` already exists, append the same pointer there if it is not already present; do not create `CLAUDE.md`.

  ```markdown
  ## Multi-role commentary

  Detailed per-symbol rationale lives under `docs/commentators/`, mirroring the source tree (e.g. `src/auth/Login.kt` → `docs/commentators/src/auth/Login.kt.md`). Source files themselves are not annotated. When working on a source file, read the matching detail file for planner / developer / security / QA reasoning.
  ```

Team-lead/main agent does both of these (not role agents or Codex workers) so the wording stays consistent.

### 10) Final report

Team-lead/main agent reports to the user:
- Number of source files read
- Approximate count of `### {Role}` sections added per role (based on role-agent, worker, or serial pass reports)
- Number of detail-doc files created vs. updated under `docs/commentators/`
- **Orphan sections** — any `## {signature}` heading in a detail doc whose symbol no longer exists in the matching source file (noticed for free while checking idempotency). List them so the user can prune; do not auto-delete.
- Whether `docs/commentators/README.md`, the `AGENTS.md` pointer, and any existing `CLAUDE.md` pointer were created or already present
- Summary of skipped files/symbols
- **Next steps reminder** — "No commits were made, and no source files were modified. Review `git diff` (only `docs/commentators/` and the instruction pointers should appear) and commit when you're ready."

## Edge cases

- **Not a git repo** — Only allow `scope=<path>`. Reject `changed`.
- **Zero target files** — Report and exit. Do not create the `docs/commentators/` tree.
- **No `AGENTS.md` or `CLAUDE.md`** — Use English default prefixes for the `### {Role}` headers. Team-lead creates a minimal `AGENTS.md` containing only the "Multi-role commentary" pointer (§9) when the first detail file is written.
- **Both `AGENTS.md` and `CLAUDE.md` present** — `AGENTS.md` wins for role-prefix conventions. `CLAUDE.md` is only used as a fallback when `AGENTS.md` has no matching rule.
- **Only unsupported extensions present** — Report "no supported files" and exit.
- **Agent write fails** — Skip that role or worker-owned file and continue; note the failure in the report.
- **Detail file path collides with an existing non-doc file** — If `docs/commentators/<path>.md` already exists but is not a commentators-format file (no `Source:` header), abort that file and report; do not overwrite.
- **Existing Claude team has a different prefix convention than this run** — Either spawn a new team with a hash suffix on the team name, or ask the user whether to reuse the existing team.

## Guardrails

- **Source files are strictly read-only** — Role agents and Codex workers `Read` source files but never `Edit`, `Write`, or create them. The only writable files are the matching detail-doc files under `docs/commentators/`.
- **Role agents and Codex workers must not run `Bash`, git commands, or builds** — they only read source and read/edit/write their owned detail-doc files.
- **Team-lead/main agent alone manages `docs/commentators/README.md`, the root `AGENTS.md` pointer, and any existing root `CLAUDE.md` pointer** — role agents and Codex workers do not write to these.
- **Claude only: always pass `team_name` when spawning role agents** — so the caller (team-lead) remains the team lead.
- **Codex only: keep worker write sets disjoint** — never assign separate workers to edit the same detail-doc file.
- **No commits, pushes, or PRs** — This skill's scope ends at doc-file writes.
- **Skip sensitive files** — Extension filters already exclude most secrets, but explicitly skip any file whose path contains `secret` or `credential`. Their detail-doc paths under `docs/commentators/` must also not be created.

## Termination

- Ends when every target file has completed the 4-role loop.
- Ends if the user cancels.
- Ends immediately if the target file list is empty.

**Claude:** Keep the team alive on exit so subsequent invocations can reuse it. Only shut down teammates (via `SendMessage` with `shutdown_request`) when the user explicitly asks to clean up the team.

**Codex:** Do not create persistent team state. Close spawned workers when they are no longer needed, after their changes have been reviewed/integrated and their final reports captured.
