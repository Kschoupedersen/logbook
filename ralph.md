---
description: "Start Ralph Loop with logbook grounding"
argument-hint: "PROMPT [--max-iterations N] [--completion-promise TEXT]"
---

# Ralph Loop (Logbook-Integrated)

Start a Ralph loop that is grounded in the project logbook. Parse arguments from: $ARGUMENTS

## Steps

### 1. Parse arguments

From `$ARGUMENTS`, extract:
- **PROMPT**: all non-flag words joined with spaces
- `--max-iterations N` → MAX_ITERATIONS (default: 0 = unlimited)
- `--completion-promise "TEXT"` → COMPLETION_PROMISE (default: null)

If no prompt provided, ask the user what task to loop on and stop.

### 2. Logbook grounding

Check if LOGBOOK.md exists in the working directory.

**If found:**
- Read it
- Note: Current State, any HIGH severity Known Issues, and "Next session start from" in the latest session entry
- Add `ralph-loop: {ISO timestamp}` to `active_sessions` in YAML frontmatter
- Output a brief grounding summary (phase, status, resuming from, relevant issues)

**If not found:**
- Output: "No logbook found — running without project context."

### 3. Construct logbook-aware prompt

Wrap the user's task. The wrapper must be concise — every iteration sees this text.

```
## Context
If LOGBOOK.md exists, read it before starting work. Note Current State and Known Issues relevant to your task. Do not redo completed work.

## Task
{PROMPT}

## On Completion
If LOGBOOK.md exists, before outputting your completion promise:
1. Write a session entry to Session Log (type: ralph-loop, include iteration count from .claude/ralph-loop.local.md)
2. Update Current State if anything changed
3. Remove yourself from active_sessions in YAML
4. Update last_updated and last_updated_by in YAML
```

If the user's PROMPT already mentions the logbook explicitly, skip the Context and On Completion sections — don't double-wrap.

### 4. Write state file

Use Bash to write `.claude/ralph-loop.local.md`:

```bash
mkdir -p .claude
cat > .claude/ralph-loop.local.md <<'STATEEOF'
---
active: true
iteration: 1
max_iterations: {MAX_ITERATIONS}
completion_promise: {COMPLETION_PROMISE as YAML — quoted string or null}
started_at: "{ISO timestamp}"
logbook_integrated: true
---

{WRAPPED_PROMPT}
STATEEOF
```

Use proper YAML quoting: `completion_promise: "TEXT"` if set, `completion_promise: null` if not.

### 5. Report

Output:
```
Ralph loop activated (logbook-integrated)

Iteration: 1
Max iterations: {N or unlimited}
Completion promise: {text or none}
Logbook: {grounded in Current State | not found}

Task: {first 100 chars of prompt}
```

Then begin working on the task immediately (this is iteration 1).

## Important

- The existing ralph-loop stop hook reads `.claude/ralph-loop.local.md` — same file, same format. The hook handles iteration counting, promise detection, and prompt replay automatically.
- `logbook_integrated: true` is metadata for the agent — the stop hook ignores unknown frontmatter fields.
- Keep the wrapper lean. The agent sees it every iteration. Verbose instructions waste context.
- The stop hook uses `session_id` for session isolation. Since we can't access `$CLAUDE_CODE_SESSION_ID` from the command context, omit it — the hook falls through gracefully for state files without session_id.
