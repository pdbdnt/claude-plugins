---
name: git-cp
description: Git commit and push workflow that aggregates ALL uncommitted changes (from current session and other Claude Code instances), reads plan files from the project root plans/ directory to build a detailed commit message, then commits and pushes. Use ONLY when user explicitly invokes "/git-cp".
---

# Git Commit & Push with Plan-Based Messages

Aggregate all uncommitted changes, build a detailed commit message from implemented plan files, then commit and push.

## Workflow

### Step 1: Survey All Changes

Run these commands to see everything uncommitted (from all sources):

```bash
git status
git diff --stat
git diff --cached --stat
git log --oneline -5
```

Note the current branch name. Identify ALL modified, added, and untracked files.

### Step 2: Read Plan Files (Only Since Last Commit)

Plans for implemented features are saved in the `plans/` directory at the **project root**.
Do NOT look in `C:\Users\denni\.claude\plans\` — that is Claude's global Plan Mode workspace and contains ALL plans including unimplemented ones.

Get the last commit timestamp, then find only plan files newer than it:

```bash
# Find plan files that are new or modified since last commit
git ls-files --others --modified plans/*.md

# Fallback: compare mod times
find plans/ -name "*.md" -newer "$(git log -1 --format=%H | xargs git show --format=%ci --no-patch | head -1)" 2>/dev/null
```

Only read plan files that are:
- **Untracked** (new, never committed) — shown by `git ls-files --others plans/`
- **Modified since last commit** — shown by `git ls-files --modified plans/`
- If a plan was already committed in a previous commit, skip it

**IMPORTANT — If no plan files found in `plans/`:**
Do NOT silently fall back to a vague commit. Instead, STOP and tell the user:
> "No plan files found in `plans/`. Please copy the implemented plan(s) from `C:\Users\denni\.claude\plans\` into the project root `plans\` directory, then re-run `/git-cp`."

For each qualifying plan file, extract:
- **Title** (H1 heading) - becomes a line item in the commit body
- **Context section** - explains the "why"
- **Changes section** - lists what was modified

### Step 3: Classify the Change Type

Determine the commit type prefix from plan titles and changes:

| Plan title contains | Prefix |
|---------------------|--------|
| "Fix", "Bug", "Error", "Patch" | `fix:` |
| "Refactor", "Move", "Rename", "Migrate" | `refactor:` |
| "Add", "Create", "Build", "Implement", new features | `feat:` |
| Mix of types | Use the dominant type, or `feat:` if unclear |

### Step 4: Build the Commit Message

Structure:

```
<type>: <concise subject summarizing ALL changes>

<For each plan that maps to actual changes:>

## <Plan Title>
- <Key change 1>
- <Key change 2>
- <Key change 3>

<If there are changes NOT covered by any plan:>

## Other Changes
- <Change 1>
- <Change 2>
```

Rules:
- Subject line: max 72 chars, imperative mood, covers ALL plans
- Body: one section per plan, with bullet points of key changes
- If multiple plans contributed, list them all
- Include file counts: e.g., "(12 files modified, 3 created)"
- End body with: `Co-Authored-By: Claude <noreply@anthropic.com>`

### Step 5: Stage Files

Stage all relevant files. NEVER stage:
- `.env*` files
- Credential/secret files
- `.claude/` internal files (plans, memory, settings)
- `node_modules/`
- OS files (`.DS_Store`, `Thumbs.db`)

Use specific file paths with `git add`, not `git add -A`. Group by category:
```bash
git add <file1> <file2> ...
```

For large changesets (20+ files), use directory-level adds with explicit exclusions.

### Step 6: Present Message for Approval

Show the user the full commit message and list of staged files. Use AskUserQuestion:

```
Ready to commit and push to <branch>:

<full commit message>

Staged files: <count>
```

Options: "Commit & Push" / "Commit Only (no push)" / "Edit Message First"

### Step 7: Execute

Based on user choice:

**Commit & Push:**
```bash
git commit -m "$(cat <<'EOF'
<commit message>
EOF
)"
git push origin <current-branch>
```

**Commit Only:**
```bash
git commit -m "$(cat <<'EOF'
<commit message>
EOF
)"
```

**Edit Message First:** Ask user what to change, update, then re-present.

### Step 8: Clean Up Plans (Optional)

After successful commit, ask the user if they want to:
- Move committed plan files to a `plans/archive/` directory
- Delete committed plan files
- Leave them as-is

## Safety Rules

- NEVER force push (`--force` or `--force-with-lease`)
- NEVER push to `main` or `master` without explicit user confirmation
- NEVER commit `.env*` or secret files
- NEVER use `--no-verify` to skip hooks
- ALWAYS show the full message before committing
- If push fails, show the error and suggest resolution (pull, rebase, etc.)

## Edge Cases

**No plan files found:** STOP and tell the user to copy the relevant implemented plan(s) from `C:\Users\denni\.claude\plans\` into the project root `plans\` directory, then re-run. Do not silently build a vague commit message.

**Plans don't match changes:** Some plans may describe work not yet reflected in git changes (future work). Only include plans whose described files overlap with actual changes.

**Merge conflicts on push:** Do NOT force push. Show the error and suggest:
```bash
git pull --rebase origin <branch>
```
Then re-attempt push after user resolves any conflicts.
