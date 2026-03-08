# TODO Task Manager

Manage tasks in `prompts/_TODO.md` and `prompts/_TODOHistory.md`.

## Usage

```
/todo <operation> <args>
```

## Operations

| Operation | Usage | Description |
|-----------|-------|-------------|
| `add-review` | `/todo add-review` | Create a new review task in `_TODO.md` |
| `add-feature` | `/todo add-feature` | Create a new feature task in `_TODO.md` |
| `done-review` | `/todo done-review <task_title_substring>` | Move a review task to `_TODOHistory.md` |
| `done-feature` | `/todo done-feature <task_title_substring>` | Move a feature task to `_TODOHistory.md` |

## Files

- **Active tasks**: `prompts/_TODO.md`
- **Completed tasks**: `prompts/_TODOHistory.md`

## File Structure

Both files use the SAME heading hierarchy:

```markdown
# TODO                          (or # TODO History)

## Review Task

### YYYYMMDD — Short Title Here

(task content...)

---

### YYYYMMDD — Another Task

(task content...)

---

## New Feature

### YYYYMMDD — Feature Title

(task content...)

---
```

**Rules:**
- `## Review Task` and `## New Feature` are the ONLY two category headings (h2)
- Each task is an `###` heading (h3) under its category
- Tasks are separated by `---` horizontal rules
- Task titles follow format: `### YYYYMMDD — Description`
- Both categories MUST always exist in both files (even if empty)

## Instructions

### Operation: `add-review`

1. **Read** `prompts/_TODO.md`
2. **Ask the user** what the review task is about (commit hash, what changed, what to verify)
3. **Generate** the task content following the review task template below
4. **Insert** the new `### YYYYMMDD — Title` block under `## Review Task`, BEFORE the first existing `###` task or BEFORE `## New Feature` if no review tasks exist
5. **Add** a `---` separator after the new task

**Review task template:**
```markdown
### YYYYMMDD — Review <What Was Changed>

**Execute on: YYYY-MM-DD ~ YYYY-MM-DD** (timeframe)
**Commit:** `<hash>` | YYYY-MM-DD | `<commit message>`

**Context:** <1-2 sentence description of what changed and why>

**Changed files:**
- `path/to/file1` — description
- `path/to/file2` — description

**Verification checklist:**

- [ ] Check 1
- [ ] Check 2
```

### Operation: `add-feature`

1. **Read** `prompts/_TODO.md`
2. **Ask the user** what the feature task is about
3. **Generate** the task content
4. **Insert** the new `### YYYYMMDD — Title` block under `## New Feature`, BEFORE the first existing `###` task under that section, or at the end of the file if no feature tasks exist
5. **Add** a `---` separator after the new task

### Operation: `done-review`

Move a completed review task from `_TODO.md` to `_TODOHistory.md`.

1. **Read** `prompts/_TODO.md`
2. **Find** the review task matching the title substring provided by the user. The task lives under `## Review Task` and starts with `### YYYYMMDD — ...`
3. **Identify the full task block**: from the `### ` heading line down to (but NOT including) the next `### ` heading or `## ` heading or end of file. Include the trailing `---` separator if present.
4. **Read** `prompts/_TODOHistory.md`
5. **Append** the task block under `## Review Task` in `_TODOHistory.md` (before `## New Feature`)
6. **Remove** the task block (including its `---` separator) from `prompts/_TODO.md`
7. **Mark any unchecked items** `[ ]` as `[x]` in the history copy (task is done)
8. **Report**: "Moved '### YYYYMMDD — Title' from _TODO.md to _TODOHistory.md"

**How to extract a task block:**
- Start: the line `### YYYYMMDD — Title`
- End: the line BEFORE the next `### ` or `## ` heading (whichever comes first), OR end of file
- Include the `---` separator line if it's the last line of the block
- When removing from `_TODO.md`, also remove any blank lines left between the previous separator and the next task

### Operation: `done-feature`

Same as `done-review` but:
- Find the task under `## New Feature` in `_TODO.md`
- Append under `## New Feature` in `_TODOHistory.md`

## Edge Cases

- If `## Review Task` section doesn't exist in a file, create it before `## New Feature`
- If `## New Feature` section doesn't exist in a file, create it at the end
- If the user's substring matches multiple tasks, list them and ask which one
- If no match found, list all tasks in the relevant section and ask the user to clarify
