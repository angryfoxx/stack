---
name: incremental-commit
description: >-
  Use this skill when the user asks to "commit incrementally",
  "incremental commit", "separated commits", "split commits", "commit changes
  separately", "commit my changes as incremental", or wants to break their
  pending changes into multiple logical atomic commits instead of one big commit.
  Also triggers when the user says "use incremental commits" or "make incremental
  commits for these changes".
---

# Incremental Commit

Break pending changes into multiple logical atomic commits, each with a proper Conventional Commit message, ordered by dependency.

## When to Use

Use this skill when the user wants to avoid a single large commit and instead have their changes committed in meaningful, separated chunks — one per logical concern.

## Allowed Tools

`Bash`, `Read`, `Glob`, `Grep`

## Commit Convention

Use Conventional Commits format:
- **Format**: `<type>(<scope>): <summary>`
- **Allowed types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `improve`
- **Header max length**: 72 chars
- **Rules**: Present tense, imperative mood, no trailing period
- **Body**: Optional short description if the summary alone is not enough — no structured labels

If the project has a commitlint config or commit convention documented (e.g., in `CLAUDE.md`, `CONTRIBUTING.md`, or `.commitlintrc`), follow that instead.

## Workflow

### Phase 1 — Discovery

Run these commands to understand the full state of changes:

```
git status
git diff
git diff --cached
git log --oneline -5
```

**Edge cases — stop and inform the user if:**
- `git status` shows no changes at all → nothing to commit
- `git status` shows conflict markers (`UU`, `AA`, `DD`) → refuse until resolved
- Changes contain only untracked files with no diff → clarify intent

**If only staged changes exist**: ask the user whether to commit staged as-is or re-analyze all staged + unstaged together.

### Phase 2 — Analysis and Grouping

Analyze the diffs semantically. Group files by **logical concern**, not file type or directory alone.

**Grouping heuristics:**

| Concern | What belongs together |
|---|---|
| Infrastructure / config | Build files, dependency lockfiles, environment configs, CI files, `docker/`, `Makefile` |
| Core / shared utilities | Shared library or utility code that other modules depend on |
| Database migrations | Migration files + the model/schema file they came from (same group) |
| Module / feature | All files within the same module/package that relate to the same feature change |
| Templates / UI | View templates or UI components — group with the feature if part of the same change, otherwise their own group |
| Tests | Group with the code being tested if part of the same feature commit; separate group if standalone test additions |
| Documentation | `docs/`, `*.md`, `CLAUDE.md`, `CONTRIBUTING.md` changes |
| Static assets | Static files, standalone |

**Rules for grouping:**
- Files serving the same feature change within the same module → one group
- Shared/core changes that multiple modules depend on → own group, committed before dependent modules
- If the entire changeset is one logical concern → single commit (do not force artificial splits)
- Assign whole files to groups (no interactive `git add -p` — file-level granularity only)

**Determine commit type from diff content:**
- New capability / new file / new field → `feat`
- Bug correction → `fix`
- Code restructure with no behavior change → `refactor`
- UI/CSS/formatting-only → `style`
- Test additions/updates → `test`
- Docs/comments → `docs`
- Build, deps, tooling → `chore`
- Enhancement to existing feature → `improve`

**Determine scope** from the module or directory name (e.g., `auth`, `api`, `core`, `ui`) or a functional area like `config`, `static`, `docker`.

### Phase 3 — Plan Presentation

Present the proposed commit plan as a numbered list **before executing anything**:

```
Proposed incremental commits:

1. chore(config): update dependency versions
   package.json, package-lock.json

2. refactor(core): extract shared pagination utility
   src/core/pagination.ts, src/core/index.ts

3. feat(users): add user profile endpoint
   src/users/routes.ts, src/users/service.ts, src/users/types.ts

4. test(users): add profile endpoint test coverage
   tests/users/profile.test.ts

Shall I proceed? You can ask me to reorder, merge, or split any group.
```

**This is a hard gate — wait for explicit user approval before executing.**

### Phase 4 — Sequential Execution

For each approved commit group, in order:

1. Stage **only** the specific files for that group:
   ```
   git add src/core/pagination.ts src/core/index.ts
   ```
   Never use `git add .`, `git add -A`, or `git add --all`.

2. Create the commit using a HEREDOC to ensure correct formatting:
   ```
   git commit -m "$(cat <<'EOF'
   refactor(core): extract shared pagination utility
   EOF
   )"
   ```

3. Verify the commit succeeded with `git status`. Confirm the staged files are gone.

4. **If a pre-commit hook fails**: stop immediately, report the error, fix the underlying issue, re-stage the same files, and create a **new** commit — never use `--amend`.

5. Proceed to the next group only after the previous commit succeeds.

### Phase 5 — Summary

After all commits are created, show the result:

```
git log --oneline -N
```

where N is the number of commits just created. Never push automatically — leave that to the user.

## Dependency Order

When ordering commits, follow this dependency sequence:

1. Infrastructure / config (others may depend on config values)
2. Core / shared library changes (multiple modules depend on these)
3. Database migrations + model/schema changes
4. Service / business logic changes
5. View / API / controller changes
6. Template / UI changes
7. Test changes (depend on the code they test)
8. Documentation changes

## Handling User Arguments

If the user passes arguments (e.g., `/incremental-commit focus on the auth changes first`), use the hint to adjust grouping priority or scope. Clarify if the hint is ambiguous before proceeding.
