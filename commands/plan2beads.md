# plan2beads

Import a Superpowers plan (Markdown) or a Shortcut story into Beads as an epic with child tasks. Supports both phase-based and task-level dependencies.

> **Note:** This command uses `temp/*.md` files with `--body-file` for descriptions instead of inline `-d "..."` arguments. This avoids Claude Code permission prompts caused by newlines in bash commands breaking pattern matching. The `temp/` directory must exist at the repo root.
>
> **Permission Avoidance Rules:**
> - Use `--body-file temp/*.md` for multi-line descriptions (avoids newline issues)
> - Use single-line `--acceptance` with semicolons: `"Criterion 1; Criterion 2; Criterion 3"` (newlines trigger prompts)
> - Do NOT delete temp files (rm triggers permission prompts) - leave for human cleanup

## Usage

```
/plan2beads <path-to-plan.md>
/plan2beads SC-1234
/plan2beads 1234
```

## Instructions

### Step 1: Load the Plan

**For local markdown files:**
- Use the Read tool to load the file content

**For Shortcut story IDs (SC-1234 or just 1234):**
- Run: `short story <numeric-id> -f=markdown`
- Note: Output starts with `#36578 Title` (no space after #, not valid H1)
- Extract epic title: everything after `#XXXXX ` on the first line
- Store the numeric ID for `--external-ref`

### Step 2: Parse the Plan Structure

| Element | How to Identify | Maps To |
|---------|----------------|---------|
| **Epic Title** | First H1 (`# Title`) OR first line of Shortcut output | Epic issue |
| **Context Sections** | H2 like "Problem Statement", "Executive Summary", "Goals", "Architecture" | Epic description |
| **Phase Groupings** | H2 matching "Phase N: ..." or "Stage N: ..." | Label for child tasks |
| **Individual Tasks** | H3 (`### Task N:`) with or without numbered prefix | Child tasks |
| **Task Dependencies** | `**Depends on:**` line within task | `--deps` flag |
| **Task Files** | `**Files:**` section within task | Preserved in description |
| **Success Criteria** | H2 "Success Metrics/Criteria" with checkboxes `- [ ]` | Epic acceptance criteria |

**Parsing Rules:**
- H2s that are NOT phases/stages/success-metrics are context (epic description)
- Tasks inherit phase label from their containing Phase H2 (if any)
- Strip numbered prefixes ("1. Task Name" → "Task Name")
- **Parse `Depends on:` line for task-level dependencies** (see Step 4)
- **Extract `Files:` section and preserve in task description**

### Step 3: Create the Epic

**IMPORTANT:** Use `temp/*.md` files for descriptions to avoid permission prompts from multi-line bash commands.

1. Write the epic description to a temp file using the Write tool:
```
Write tool → temp/epic-desc.md
Content: Concatenated context sections (Problem Statement, Goals, Architecture, etc.)
```

2. Create the epic using `--body-file`:
```bash
bd create --silent --type epic "Epic Title" --body-file temp/epic-desc.md --acceptance "Criterion 1; Criterion 2; Criterion 3" -p 1
```

**IMPORTANT:** Use semicolons to separate acceptance criteria on one line. Newlines in `--acceptance` trigger permission prompts.

- Add `--external-ref "sc-1234"` for Shortcut stories
- If no H1 found, use the filename (without extension) as title

**Capture the returned ID** (e.g., `hub-abc`) - you'll need it for child tasks.

### Step 4: Create Child Tasks (Dependency-Aware)

**IMPORTANT:** Parse `**Depends on:**` line from each task to determine dependencies.

#### 4a. Build Task ID Map

As you create tasks, maintain a mapping:
```
Task 1 → hub-abc.1
Task 2 → hub-abc.2
Task 3 → hub-abc.3
...
```

#### 4b. Parse Dependencies

For each task, look for `**Depends on:**` line:

| Depends on Value | Beads --deps |
|------------------|--------------|
| `None` | (no --deps flag) |
| `Task 1` | `--deps "hub-abc.1"` |
| `Task 1, Task 3` | `--deps "hub-abc.1,hub-abc.3"` |
| `Task 2 (User model)` | `--deps "hub-abc.2"` (ignore parenthetical) |

**Parsing regex:** `\*\*Depends on:\*\*\s*(.+)$`
- If value is "None", no dependencies
- Otherwise, extract task numbers: `Task\s+(\d+)` → map to beads IDs

**Validation:**
- **Forward references:** If Task 2 says "Depends on: Task 5" (higher number), warn and skip—forward dependencies indicate a plan ordering issue
- **Self-references:** If Task 3 says "Depends on: Task 3", warn and skip—task cannot depend on itself
- **Non-existent tasks:** If "Depends on: Task 10" when only 4 tasks exist, warn and skip
- Continue creating the task without invalid dependencies
- Report all warnings in Step 6 summary

**Edge case - No tasks found:**
If parsing finds 0 tasks (no H3 headings matching `### Task N:`):
- Warn: "No tasks found in plan - check format"
- Create epic only, no children
- Human should review and fix plan format

#### 4c. Create Tasks in Order

Create tasks **in numeric order** so dependencies can reference earlier IDs.

**IMPORTANT:** Use `temp/*.md` files for task descriptions to avoid permission prompts from multi-line bash commands.

For each task:
1. Write task description to `temp/task-desc.md` using the Write tool
2. Create the task with `--body-file temp/task-desc.md`
3. Reuse the same temp file for each task (overwrite)

```bash
# Task 1: No dependencies
# (Write tool creates temp/task-desc.md with task 1 description)
bd create --silent --parent hub-abc "User Model" --body-file temp/task-desc.md -p 2
# Returns: hub-abc.1

# Task 2: No dependencies (can be parallel with Task 1 at execution time)
# (Write tool overwrites temp/task-desc.md with task 2 description)
bd create --silent --parent hub-abc "JWT Utils" --body-file temp/task-desc.md -p 2
# Returns: hub-abc.2

# Task 3: Depends on Task 1
# (Write tool overwrites temp/task-desc.md with task 3 description)
bd create --silent --parent hub-abc "Auth Service" --body-file temp/task-desc.md --deps "hub-abc.1" -p 2
# Returns: hub-abc.3

# Task 4: Depends on Task 2 and Task 3
# (Write tool overwrites temp/task-desc.md with task 4 description)
bd create --silent --parent hub-abc "Login Endpoint" --body-file temp/task-desc.md --deps "hub-abc.2,hub-abc.3" -p 2
# Returns: hub-abc.4
```

#### 4d. Task Description Format

Include the full task content in the description, preserving structure:

```markdown
## Files
- Create: `apps/api/src/models/user.model.ts`
- Modify: `apps/api/src/models/index.ts`
- Test: `apps/api/src/__tests__/models/user.test.ts`

## Implementation Steps
**Step 1: Write the failing test**
...

**Step 2: Run test to verify it fails**
Run: `pnpm test -- --grep "user model"`
Expected: FAIL
...
```

**CRITICAL:** The `## Files` section MUST be preserved - it enables parallel execution safety.

**If `## Files` section is missing:**
- Warn: "Task N missing Files section - cannot parallelize safely"
- Still create the issue, but execution will treat it as conflicting with all others

#### 4e. Phase Labels (Optional)

If the plan uses Phase groupings, add phase labels:
```bash
bd create --silent --parent hub-abc "Task Name" -d "..." -l "phase:1" --deps "hub-abc.1" -p 2
```

If no phases, omit the `-l` flag.

### Step 5: Verify Dependency Structure

**REQUIRED:** After creating all tasks, verify the dependency structure:

```bash
# Show tasks that are ready (no blockers)
bd ready

# Show tasks that are blocked
bd blocked

# Visual dependency graph
bd graph hub-abc
```

**Check for issues:**
- Tasks with `Depends on: None` should appear in `bd ready`
- Tasks with dependencies should appear in `bd blocked`
- Graph should show expected dependency flow

### Step 6: Display Results

Show summary:
```
Created epic: hub-abc "Epic Title"

Created N tasks:
  Ready (no blockers): hub-abc.1, hub-abc.2
  Blocked: hub-abc.3 (by hub-abc.1), hub-abc.4 (by hub-abc.2,hub-abc.3)

Dependency verification:
  bd ready shows: 2 tasks ready
  bd blocked shows: 2 tasks blocked

Parallelization potential:
  Wave 1: Tasks 1, 2 (parallel)
  Wave 2: Task 3 (after 1)
  Wave 3: Task 4 (after 2, 3)

Next commands:
  bd show hub-abc        # View epic details
  bd graph hub-abc       # Visual dependency graph
  bd ready               # Start working
```

## bd CLI Quick Reference

| Flag | Purpose |
|------|---------|
| `--silent` | Output only ID (for capturing) |
| `--type epic` | Create epic instead of task |
| `--parent <id>` | Create as child of epic |
| `--body-file <path>` | Read description from file (use for multi-line content) |
| `-d "text"` | Description (single-line only, prefer --body-file) |
| `-l "label"` | Labels (comma-separated for multiple) |
| `-p N` | Priority (0-4) |
| `--deps "id1,id2"` | Dependencies (blocked by these) |
| `--acceptance "text"` | Acceptance criteria |
| `--external-ref "ref"` | External link (e.g., "sc-1234") |

## Backward Compatibility

**Phase-based plans (old format):**
If the plan uses Phase H2 groupings WITHOUT `**Depends on:**` lines:
- Phase 0 tasks: no dependencies
- Phase N tasks: depend on ALL Phase N-1 tasks

**Task-level dependencies (new format):**
If tasks have `**Depends on:**` lines:
- Use specific task references
- Phase labels are optional (for filtering)
- This enables finer-grained parallelism
