---
name: logbook
version: "1.0"
author: Kristian Ole Schou-Pedersen
license: MIT
description: Living engineering logbook protocol for AI-assisted project work
---

# Logbook

Living engineering logbook for AI-assisted project work. Tracks sessions, decisions, issues, and backlog across agents and humans.

## Usage
```
/logbook <command> [args]
```

## Commands
- `init [path]` — Create LOGBOOK.md for a project (scans for existing state)
- `start` — Session start protocol (read, check freshness, surface issues)
- `end` — Session end protocol (write entry, update state, clean up)
- `audit` — Verify Current State claims against actual files
- `status` — Quick summary of what's in flight
- `issue [description]` — Add a known issue
- `decide [title]` — Record a decision

---

## Instructions for Claude

A logbook is project memory. Its job: any engineer or agent can pick up where the last one left off, cold, without asking questions. Every command either reads or writes that memory. Never skip the end protocol.

---

## Command: `init [path]`

### Steps
1. **Determine target path** — use argument if given, else current directory
2. **Scan for existing state:**
   - Count files and folders (rough size signal)
   - Look for README.md, INDEX.md, existing docs with status/completion info
   - Look for any file named LOGBOOK.md already (if found: confirm overwrite or update)
3. **Ask two questions:**
   - "What is this project?" (name + one-line description)
   - "What phase is it in?" (e.g. planning, in-progress, maintenance, complete)
4. **Synthesize Current State from scan:**
   - If nothing found: "New project — no content yet."
   - If content found: summarize what exists (file count, folder structure, any obvious completion signals)
5. **Create LOGBOOK.md** at target path using the template below
6. **Report** what was created and what `start` will do next session

### LOGBOOK.md Template

Generate this file, substituting scanned values into the YAML and Current State:

```markdown
---
logbook:
  project: {{PROJECT_NAME}}
  type: engineering-logbook
  version: "1.0"
  created: "{{TODAY}}"
  last_updated: "{{TODAY}}"
  last_updated_by: "{{NAME_OR_AGENT}}"

state:
  phase: "{{PHASE}}"
  status: idle          # idle | in-flight | blocked | complete
  sprint: null

sections:
  current_state:
    freshness_days: 7
    volatile: true
  decision_log:
    threshold_chars: 8000
    child_doc_pattern: "decisions/YYYY.md"
    ask_before_split: true
  session_log:
    threshold_chars: 10000
    child_doc_pattern: "logs/YYYY-MM.md"
    ask_before_split: true
  known_issues:
    threshold_chars: 5000
    freshness_days: 30
    child_doc_pattern: "ISSUES.md"
    ask_before_split: true
  backlog:
    threshold_chars: 5000
    child_doc_pattern: "BACKLOG.md"
    ask_before_split: true

active_sessions: []
child_docs: []
---

## Purpose & Philosophy

A logbook is not a todo list. It is not a sprint board. It is a memory.

Its job:
- **For humans:** "Where were we? Why did we make that decision? What's blocked?"
- **For agents:** "What do I need to continue? What changed last session? What's dangerous to touch?"
- **For new team members:** "What is this project? How does it work? How do I start contributing?"

Three properties that make it useful vs. a file that gets abandoned:
1. **Updated at session end** — not retrospectively. If the agent always writes the handoff note, it is always current.
2. **Lives near the work** — not in Notion, not in Slack. In the repo, next to the files it describes.
3. **Asks before growing** — threshold mechanism prevents it becoming a wall nobody reads.

---

## Agent Protocol

**SESSION START:**
1. Read Current State — understand what's in flight
2. Check `active_sessions` in YAML — if non-empty, coordinate before working
3. Check Current State freshness (compare `last_updated` to today vs `freshness_days`) — flag if stale
4. Scan Known Issues for OPEN items with HIGH severity — surface to human
5. Scan Decision Log for CURRENT decisions older than 90 days — note for review
6. Read `Next session start from:` in most recent Session Log entry
7. Add yourself to `active_sessions` in YAML

**SESSION END:**
1. Write session entry (format below — `Next session start from:` is mandatory)
2. Update Current State if anything changed
3. Remove yourself from `active_sessions` in YAML
4. Update `last_updated` and `last_updated_by` in YAML
5. Before appending to any section: check section size vs `threshold_chars` — if exceeded, ask user before creating child doc

---

## Current State

> **VOLATILE** — update this section at every session end.
> Freshness: 7 days. If `last_updated` in YAML is older, flag as stale.

**Phase:** {{PHASE}}
**Status:** {{STATUS_DESCRIPTION}}

**What's done:**
{{SCANNED_SUMMARY_OR_NOTHING_YET}}

**What's in flight:**
- Nothing active

**What's next:**
- {{FIRST_TASK_OR_TBD}}

---

## Active Sessions

> Populated by engineers/agents at session start. Cleared at session end.
> If this section is non-empty when you start: coordinate before working.

*(empty)*

---

## Decision Log

> Most recent first. Threshold: 8000 chars → ask before archiving to `decisions/YYYY.md`

<!-- THRESHOLD CHECK: before adding an entry, estimate this section's size.
     If it would exceed 8000 chars, ask user before creating decisions/YYYY.md -->

*(no decisions recorded yet)*

---

## Session Log

> Most recent first. Threshold: 10000 chars → ask before archiving to `logs/YYYY-MM.md`

<!-- THRESHOLD CHECK: before adding an entry, estimate this section's size.
     If it would exceed 10000 chars, ask user before creating logs/YYYY-MM.md -->

*(no sessions recorded yet)*

---

## Known Issues

> Threshold: 5000 chars → ask before archiving to ISSUES.md
> Staleness rule: issues open > 30 days get flagged as STALE at session start.

<!-- THRESHOLD CHECK: before adding items, estimate this section's size.
     If it would exceed 5000 chars, ask user before creating ISSUES.md -->

| ID | Description | Severity | Since | Status | Assigned |
|----|-------------|----------|-------|--------|----------|

---

## Backlog

> Threshold: 5000 chars → ask before archiving to BACKLOG.md

<!-- THRESHOLD CHECK: before adding items, estimate this section's size.
     If it would exceed 5000 chars, ask user before creating BACKLOG.md -->

### Gruntwork (< 2 hours each)

*(none yet)*

### Long-Term Engineering

| ID | Task | Priority | Est. Sessions | Status | Notes |
|----|------|----------|---------------|--------|-------|

---

## Child Documents

> Updated by agent when a section is split out (after user approval).

*(none yet)*

---

*Logbook version: 1.0 | Created: {{TODAY}}*
*To reuse: copy this file to a new project, clear live section content, adjust YAML frontmatter.*
```

---

## Command: `start`

### Steps
1. Read LOGBOOK.md — load full context
2. Check `active_sessions` in YAML — if non-empty, report who's active and ask how to proceed
3. Compare `last_updated` date to today:
   - If older than `freshness_days` (default 7): output `⚠️ STALE — Current State last updated {date}. Verify before working.`
4. Scan Known Issues table — surface any OPEN + HIGH severity items to human
5. Check issues older than `freshness_days: 30` — flag as STALE inline
6. Scan Decision Log — flag any CURRENT decisions with date > 90 days ago
7. Read `Next session start from:` in most recent session entry — output it as "Resuming from: ..."
8. Add `{name}: {timestamp}` to `active_sessions` in YAML frontmatter
9. Output summary:
   ```
   Logbook: {{PROJECT_NAME}}
   Phase: {{PHASE}} | Status: {{STATUS}}

   Resuming from: {{NEXT_SESSION_START_FROM}}

   Open issues (HIGH): {{COUNT_OR_NONE}}
   Active sessions: {{LIST_OR_JUST_YOU}}
   ```

---

## Command: `end`

### Steps
1. Prompt for session summary if not provided:
   - "What did you work on this session? (brief, bullet points ok)"
   - "Did anything change in Current State?"
   - "What's the exact starting point for next session?"
2. Write session entry to Session Log (most recent first):

**Entry format:**
```markdown
### Session YYYY-MM-DD — [Name/Agent]
**Type:** gruntwork | engineering | review
**Focus:** [one-line summary]

**Worked on:**
- [x] [task] — [outcome]
- [ ] [task] — [why incomplete]

**State changes:** [updates to Current State section, or "none"]
**Open questions:** [needs human input, or "none"]
**Next session start from:** [exact file, task, known issue ID — mandatory]
```

3. If Current State changed: update it
4. Before appending: check session log size vs `threshold_chars: 10000` — if exceeded, ask user to archive
5. Remove yourself from `active_sessions` in YAML
6. Update `last_updated` and `last_updated_by` in YAML frontmatter
7. Confirm: "Session logged. Logbook updated."

---

## Command: `audit`

### Steps
1. Read Current State from LOGBOOK.md
2. Scan actual project files:
   - List all folders and approximate file counts
   - Check for any README or INDEX files that indicate completion status
   - Look for obvious mismatches between Current State claims and actual file existence
3. For new/empty projects: report "Project appears empty or new — Current State is accurate."
4. For existing projects:
   - List what Current State claims vs what was found
   - Flag discrepancies (claimed files that don't exist, existing files not mentioned)
   - Check for broken internal links if markdown files are present
5. Propose updates to Current State and Known Issues for any discrepancies found
6. Ask before writing any changes

---

## Command: `status`

### Output format
```
Logbook: {{PROJECT_NAME}}
Phase: {{PHASE}} | Status: {{STATUS}} | Sprint: {{SPRINT_OR_NONE}}
Last updated: {{DATE}} by {{WHO}}

In flight: {{WHAT_OR_NOTHING}}
Open issues: {{COUNT}} ({{HIGH_COUNT}} HIGH)
Backlog: {{GRUNTWORK_COUNT}} gruntwork, {{ENGINEERING_COUNT}} engineering tasks
Active sessions: {{LIST_OR_NONE}}

Next action: {{NEXT_SESSION_START_FROM_LATEST_ENTRY}}
```

---

## Command: `issue [description]`

### Steps
1. Read Known Issues table from LOGBOOK.md
2. Determine next ID (I-NNN, incrementing from highest existing)
3. Ask for severity if not obvious: HIGH / MED / LOW
4. Ask for assignee (default: unassigned)
5. Append row to Known Issues table:
   `| I-NNN | {description} | {severity} | {today} | OPEN | {assignee} |`
6. Check table size vs `threshold_chars: 5000` — if exceeded, ask user before creating ISSUES.md
7. Confirm: "Added I-NNN."

---

## Command: `decide [title]`

### Steps
1. Read Decision Log from LOGBOOK.md
2. Determine next ID (D-NNN, incrementing from highest existing)
3. Ask for:
   - Problem: what needed deciding?
   - Options considered (A, B, etc.)
   - Chosen option and rationale
   - Made by (default: current user/agent)
   - Review date (default: N/A)
4. Append entry to Decision Log (most recent first):

```markdown
### D-NNN — {title} — {today}
**Problem:** {problem}
**Options:** A: {desc} — {why} | B: {desc} — {why}
**Chosen:** {option} — {rationale}
**Made by:** {name} | **Review date:** {date_or_NA}
**Status:** CURRENT
```

5. Check log size vs `threshold_chars: 8000` — if exceeded, ask user before archiving
6. Confirm: "Recorded D-NNN."

---

## Behavior Rules

### Core rules
- ALWAYS read LOGBOOK.md before doing any work in a project
- ALWAYS write a session entry at end — `Next session start from:` is mandatory
- NEVER auto-split sections to child docs — always ask first
- NEVER mark old decisions as deleted — mark as `SUPERSEDED-BY-D-NNN` or `INVALIDATED`
- Check threshold_chars before appending to any section

### Multi-engineer safety
- `active_sessions` is non-binding intent signaling, not a hard lock
- If active_sessions is non-empty: surface it, don't block
- Coordinate by reading the most recent session entry first

### Staleness
- Current State older than `freshness_days: 7` → flag as stale, don't trust it blindly
- Known Issues older than `freshness_days: 30` → flag as STALE inline
- Decision Log entries older than 90 days with status CURRENT → flag for review

---

## Example Session

```
User: /logbook init ./my-project

Claude: Scanning ./my-project...
        Found: 3 folders, 12 .md files, 1 README.md
        README suggests project is "roughly 60% complete"

        Two questions:
        1. What is this project?
        > API documentation rewrite for Acme platform

        2. What phase is it in?
        > in-progress

        Creating LOGBOOK.md...

        Current State seeded with:
        - 12 markdown files found across 3 folders
        - README suggests ~60% complete
        - No existing issues or decisions logged

        Done. Run /logbook start at the beginning of each session.

---

User: /logbook start

Claude: Logbook: my-project
        Phase: in-progress | Status: idle
        Last updated: 2026-03-03 by init

        No previous session found — this is session 1.

        Open issues (HIGH): none
        Active sessions: just you

        Recommendation: run /logbook audit to verify Current State before working.

---

User: /logbook end

Claude: What did you work on?
        > Updated auth section, fixed 3 broken links, started rate-limiting docs

        Anything change in Current State?
        > Auth section now complete, rate-limiting 50% done

        Where should next session start?
        > rate-limiting/README.md — finish the examples section

        Writing session entry...
        Updating Current State...
        Clearing active_sessions...

        Session logged. Logbook updated.
```

---

