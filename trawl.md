---
name: trawl
version: "1.2"
variant: logbook-integrated
author: Kristian Ole Schou-Pedersen
license: MIT
dependencies:
  - skill: logbook
    version: ">=1.0"
    required: true
description: Coordinated multi-agent retrieval with protocol-aware subagents and logbook integration
---

# Trawl (Logbook-Integrated)

Coordinated multi-agent retrieval from texts too large for a single context window. A coordinator splits the source, spawns protocol-aware agents with identical instructions, and synthesizes their findings.

This variant integrates with the **logbook** skill. Trawls are tracked as logbook sessions. Decisions triggered by trawl findings are recorded in the logbook's decision log. Failed or problematic trawls surface as known issues.

**v1.2 change:** Workers are now native Claude Code subagents (`trawl-worker` defined in `~/.claude/agents/`). The coordinator spawns them via `Agent(trawl-worker)` — they get the protocol from their agent definition, not embedded in the prompt. Workers can read PDFs directly.

## Usage
```
/trawl <command> [args]
```

## Commands
- `plan [source] [query]` — Define a trawl mission: what to search, what to find
- `run` — Execute the trawl (spawn agents, collect results)
- `results` — View synthesized findings
- `rerun [chunk_ids]` — Re-trawl specific chunks (missed coverage, ambiguous results)

---

## Instructions for Claude

A trawl is a structured sweep across material that won't fit in one context window. The document defines everything: source, query, chunk boundaries, agent instructions, and output schema. Every agent gets the same mission brief. The coordinator never interprets raw source — only agent outputs.

This variant requires a LOGBOOK.md in the project. If none exists, prompt the user to run `/logbook init` first.

### Architecture: Coordinator and Subagents

**Coordinator** (you, the main agent):
- Owns TRAWL.md and LOGBOOK.md — reads and writes freely
- Creates the working directory `trawl-T-NNNN/` for agent result files
- Spawns `trawl-worker` subagents via `Agent(trawl-worker)` — one per chunk
- Passes chunk-specific parameters (trawl_id, chunk_id, paths, query, schema) as the task prompt
- After all agents return: reads result files, updates TRAWL.md, processes flags into LOGBOOK.md

**Subagents** (`trawl-worker` — defined in `~/.claude/agents/trawl-worker.md`):
- Protocol-aware by design — their agent definition contains the full worker protocol
- **Read** TRAWL.md at start — to understand the mission context
- **Read** their assigned chunk content (text files OR PDFs — the Read tool handles both)
- **Write** their result file to `trawl-T-NNNN/C-NNN.yaml` (isolated, no concurrent write conflicts)
- **Flag** issues/decisions/notes in structured YAML output — the coordinator processes these
- **Never write** to TRAWL.md or LOGBOOK.md directly (concurrent write safety)
- Have tools: Read, Write, Glob, Grep — no Edit (they create files, not modify them)

The workers are native Claude Code subagents. Their protocol lives in their agent definition, not embedded in the coordinator's prompt. This means the coordinator just passes the task parameters — the worker already knows how to behave.

### Logbook Integration Points

The trawl hooks into the logbook at these moments:

**On `plan`:**
- Verify LOGBOOK.md exists — abort if not
- The trawl ID (T-NNNN) is registered in the logbook's Current State under "What's in flight"
- Create working directory `trawl-T-NNNN/` for agent result files

**On `run`:**
- The trawl run is treated as a logbook session — add to `active_sessions` at start
- If the logbook already has active sessions: surface them, ask how to proceed (same as `/logbook start`)

**On `results` / completion:**
- Read all result files from `trawl-T-NNNN/`
- Process `flags` from agent results into logbook (issues, decision prompts)
- Write a session entry to the logbook's Session Log with trawl summary:
  - Type: `trawl`
  - Focus: query instruction
  - Worked on: chunk stats, finding count, flags processed
  - State changes: update Current State if trawl findings change project understanding
  - Next session start from: what to do with the findings
- Remove from `active_sessions`
- Update `last_updated` and `last_updated_by` in logbook YAML

**On failure or conflicts:**
- Persistent failures (chunks that fail after retry) → auto-create known issue via `/logbook issue`
- Agent-flagged issues → coordinator creates logbook issues with agent's description
- Conflicts flagged in synthesis → surface to human, offer to log as issue or decision

**On decisions prompted by findings:**
- Agent-flagged decisions → coordinator surfaces to human with context
- If trawl results lead to a decision → offer `/logbook decide` inline
- TRAWL.md references the decision ID, logbook decision references the trawl ID — bidirectional link

---

## Command: `plan [source] [query]`

### Steps
1. **Check for LOGBOOK.md** — if not found, prompt: "No logbook found. Run `/logbook init` first, or use `/trawl` standalone variant."
2. **Identify source material** — file path(s), URL(s), or directory
3. **Measure scope:**
   - Estimate total size (chars, pages, files)
   - Determine if chunking is by page range, file, section headers, or fixed char count
4. **Define the query** — what are we retrieving? Be specific:
   - Extraction: "pull all dates and amounts from invoices"
   - Search: "find references to X across all documents"
   - Classification: "categorize each section as technical/business/legal"
   - Summarization: "summarize key points per chapter"
5. **Generate TRAWL.md** at working directory using template below
6. **Update logbook** — add trawl to Current State "What's in flight"
7. **Report** chunk count, estimated agent count, and query summary

### TRAWL.md Template

```markdown
---
trawl:
  id: "T-{{NNNN}}"
  created: "{{TODAY}}"
  status: planned          # planned | running | complete | partial
  coordinator: "{{NAME_OR_AGENT}}"
  last_updated: "{{TODAY}}"
  last_updated_by: "{{NAME_OR_AGENT}}"

logbook:
  path: "{{PATH_TO_LOGBOOK_MD}}"
  session_id: null         # set when run starts
  linked_decisions: []     # D-NNN IDs from logbook
  linked_issues: []        # I-NNN IDs from logbook

source:
  description: "{{WHAT_IS_THIS_TEXT}}"
  paths:                   # list of file paths, URLs, or glob patterns
    - "{{PATH_1}}"
  total_size_estimate: "{{SIZE}}"
  chunk_strategy: "{{pages | files | sections | fixed_chars}}"
  chunk_size: "{{TARGET_SIZE_PER_CHUNK}}"
  overlap_chars: 200       # overlap between chunks to catch boundary content

query:
  type: "{{extraction | search | classification | summarization}}"
  instruction: "{{EXACT_INSTRUCTION_EACH_AGENT_RECEIVES}}"
  output_schema:           # every agent returns this exact shape
    - field: "{{FIELD_NAME}}"
      type: "{{string | list | number | boolean}}"
      description: "{{WHAT_THIS_FIELD_CAPTURES}}"

agents:
  model: "haiku"
  count: {{N}}             # determined by chunks
  timeout_seconds: 120
  max_retries: 1

workdir: "trawl-T-{{NNNN}}/"  # agent result files go here

chunks: []                 # populated during plan, format below
---

## Purpose & Philosophy

A trawl is not a summary. It is not a search engine. It is a structured sweep.

Its job:
- **For the coordinator:** "I can't read all of this. Send agents to bring back exactly what I need, in a shape I can merge."
- **For the agents:** "Here is one piece. Here is what to look for. Return findings in this exact shape. Nothing else."
- **For the human:** "I defined what I needed once. The document captured the full specification. Anyone can re-run it and get the same results."

Three properties that make it work:
1. **One query, many agents** — every agent gets the identical brief. No per-agent tweaking, no drift.
2. **Rigid output schema** — agents return YAML in a fixed shape. If it doesn't match, it's a failure, not a creative interpretation.
3. **The document is the spec** — TRAWL.md captures everything needed to reproduce the run. Source, query, chunks, results, synthesis.

This trawl is tracked by the project logbook at `{{PATH_TO_LOGBOOK_MD}}`.

---

## Worker Task Brief

> The coordinator passes this to each `trawl-worker` subagent via `Agent(trawl-worker)`.
> The worker already knows the protocol from its agent definition — this brief only provides chunk-specific parameters.

```
Process chunk {{CHUNK_ID}} of trawl {{TRAWL_ID}} ({{TOTAL_CHUNKS}} total chunks).

trawl_path: {{PATH_TO_TRAWL_MD}}
chunk_path: {{CHUNK_FILE_PATH}}
workdir: {{TRAWL_WORKDIR}}
query_instruction: {{QUERY_INSTRUCTION}}
output_schema:
  {{SCHEMA_FIELDS}}
```

The worker reads TRAWL.md, reads the chunk (text or PDF), extracts findings per the query, writes to `{{TRAWL_WORKDIR}}/{{CHUNK_ID}}.yaml`, and flags anything unusual.

---

## Chunks

> Generated during `plan`. Each row = one agent assignment.

| Chunk ID | Source | Range | Size | Status | Agent |
|----------|--------|-------|------|--------|-------|

---

## Results

> Agent result files live in `trawl-T-{{NNNN}}/`. One YAML file per chunk.
> The coordinator reads these after all agents complete — never modify them.

*(empty until run — see workdir for raw agent outputs)*

---

## Synthesis

> Coordinator merges agent results after all agents complete.

**Merge rules:**
- Deduplicate findings that appear in overlapping regions
- Flag boundary_note items — check if adjacent chunk captured the continuation
- Sort findings by source order (chunk sequence), not discovery order
- Conflicts between agents on the same content: flag for human review, do not auto-resolve

*(empty until synthesis)*

---

## Run Log

| Run | Timestamp | Agents | Completed | Failed | Findings | Notes |
|-----|-----------|--------|-----------|--------|----------|-------|

---

## Logbook Trail

> Bidirectional links between this trawl and logbook entries.

| Type | ID | Description | Created |
|------|----|-------------|---------|

---

## Lifecycle

> What happens to this file and the workdir after the trawl completes.

- **Keep** if results are referenced by logbook decisions or issues
- **Archive** to `trawls/T-NNNN/` (move TRAWL.md + workdir together) if the working directory accumulates multiple trawls
- **Delete** if findings have been fully absorbed into the logbook — delete both TRAWL.md and `trawl-T-NNNN/`

Before deleting: verify no logbook entries reference this trawl ID. If they do, keep or archive — don't break links.

---

*Trawl version: 1.2 (logbook-integrated, native trawl-worker subagents, PDF support) | Created: {{TODAY}}*
*Source: {{SOURCE_DESCRIPTION}} | Query: {{QUERY_TYPE}} | Chunks: {{N}}*
*Workdir: trawl-T-{{NNNN}}/ | Logbook: {{PATH_TO_LOGBOOK_MD}}*
```

---

## Command: `run`

### Steps
1. Read TRAWL.md — load mission, chunks, query brief
2. Verify status is `planned` or `partial` (not already `complete`)
3. **Logbook check-in:**
   - Read LOGBOOK.md — check `active_sessions`
   - If non-empty: surface active sessions, ask how to proceed
   - Add trawl coordinator to `active_sessions`
4. **Create working directory** `trawl-T-NNNN/` next to TRAWL.md for agent result files
5. **Prepare chunks** — split source material according to `chunk_strategy` and `chunk_size`
   - Apply `overlap_chars` between adjacent chunks
   - Assign chunk IDs (C-001, C-002, ...)
   - Update chunks table in TRAWL.md
6. **Spawn `trawl-worker` subagents** — for each chunk:
   - Use `Agent(trawl-worker)` with the Worker Task Brief filled in:
     - `{{TRAWL_ID}}`, `{{CHUNK_ID}}`, `{{TOTAL_CHUNKS}}`
     - `{{PATH_TO_TRAWL_MD}}` — absolute path to TRAWL.md
     - `{{CHUNK_FILE_PATH}}` — absolute path to the chunk's source file (text or PDF)
     - `{{TRAWL_WORKDIR}}` — absolute path to `trawl-T-NNNN/`
     - `{{QUERY_INSTRUCTION}}` — from TRAWL.md query section
     - `{{SCHEMA_FIELDS}}` — from TRAWL.md output_schema
   - Workers already know the protocol from their agent definition — the brief only passes parameters
   - Workers run in parallel — each writes to its own file, no conflicts
   - Workers can read PDFs directly via the Read tool (use `pages` param for >10 pages)
7. **Collect results** — after all agents return:
   - Read each `trawl-T-NNNN/C-NNN.yaml` file
   - Update chunk status in TRAWL.md chunks table from each result's `status` field
   - If a result file is missing or malformed: mark chunk as `failed`, retry once
8. **Process flags** — for each agent result:
   - `flags.issues` → create logbook issues via `/logbook issue`, link in TRAWL.md
   - `flags.decisions` → surface to human, offer `/logbook decide`
   - `flags.notes` → include in synthesis section
9. Update run log
10. Set status to `complete` (all done) or `partial` (some failed)
11. Proceed to synthesis automatically

---

## Command: `results`

### Steps
1. Read TRAWL.md — load synthesis
2. Read all result files from `trawl-T-NNNN/` — load raw findings and flags
3. If synthesis is empty: run synthesis first (merge, dedup, sort)
4. Output:
   ```
   Trawl: {{ID}}
   Source: {{DESCRIPTION}}
   Query: {{INSTRUCTION}}

   Chunks: {{TOTAL}} ({{DONE}} done, {{EMPTY}} empty, {{FAILED}} failed)
   Findings: {{COUNT}}

   {{SYNTHESIZED_FINDINGS}}

   Boundary flags: {{COUNT_OR_NONE}}
   Conflicts: {{COUNT_OR_NONE}}

   Agent flags:
     Issues raised: {{COUNT}} ({{PROCESSED}} processed into logbook)
     Decisions flagged: {{COUNT}}
     Notes: {{COUNT}}
   ```
5. **Surface unprocessed flags** — if any agent flags weren't processed during `run`:
   - Show each flag with its chunk context
   - Offer logbook actions for each
6. **Offer logbook actions:**
   - "Record a decision based on these findings? (`/logbook decide`)"
   - "Log an issue from the boundary flags or conflicts? (`/logbook issue`)"
7. **Write logbook session entry:**
   - Type: `trawl`
   - Focus: "Trawl T-NNNN — {{QUERY_INSTRUCTION}}"
   - Worked on: "{{CHUNK_COUNT}} chunks, {{FINDING_COUNT}} findings, {{FLAG_COUNT}} flags"
   - State changes: update Current State if findings changed project understanding (ask human)
   - Next session start from: "Review trawl T-NNNN synthesis" or human-specified
8. Remove coordinator from logbook `active_sessions`
9. Update logbook `last_updated` and `last_updated_by`

---

## Command: `rerun [chunk_ids]`

### Steps
1. Read TRAWL.md
2. If no chunk_ids given: rerun all `failed` chunks
3. If chunk_ids given: rerun those specific chunks (even if previously `done`)
4. Clear previous results for those chunks
5. Follow same spawn/collect flow as `run` (including logbook active_sessions check)
6. Re-synthesize with updated results
7. Update run log with rerun entry
8. If previously linked issues are now resolved (chunk succeeded on rerun): ask to close logbook issue

---

## Behavior Rules

### Core
- The `trawl-worker` agent definition (`~/.claude/agents/trawl-worker.md`) is the SINGLE SOURCE OF TRUTH for worker behavior
- The coordinator passes only chunk-specific parameters — the worker protocol is in the agent definition
- Workers must never see other workers' results — they work in isolation
- Workers READ shared documents (TRAWL.md) but WRITE only to their isolated result file
- Workers can read text files AND PDFs — the coordinator doesn't need to convert formats
- The coordinator never reads raw source — only worker result files
- Chunk overlap exists to catch boundary content — synthesis must deduplicate

### Determinism
- Same source + same query + same chunking = same agent instructions every time
- The TRAWL.md file captures the full specification — any coordinator can re-execute it
- Agent output schema is rigid YAML — malformed result files are treated as failures
- Result files in `trawl-T-NNNN/` are the raw evidence — don't modify them after collection

### Safety
- Agents read TRAWL.md for context but never write to it — the coordinator owns shared docs
- Agents write ONLY to `trawl-T-NNNN/C-NNN.yaml` — one file per agent, no concurrent write conflicts
- If source contains sensitive data: note in TRAWL.md header, do not log raw content in results
- Failed agents are retried once, then flagged — never silently dropped

### Logbook Integration
- LOGBOOK.md must exist before `plan` — if missing, prompt for `/logbook init`
- Every trawl run is a logbook session — check in at start, check out at end
- Agent `flags.issues` become logbook issues — coordinator creates them with agent context
- Agent `flags.decisions` are surfaced to human — coordinator offers `/logbook decide`
- Failures become logbook issues — bidirectional link via IDs
- Never delete a TRAWL.md or its `trawl-T-NNNN/` directory while linked logbook entries exist

### Scaling
- Default to 3+ agents minimum (below 3, just read the text directly)
- No upper limit, but chunk_size should stay large enough for meaningful retrieval
- If chunk count exceeds 20: ask user to confirm before spawning
- All agents run in parallel — isolated result files prevent conflicts at any scale

---

## Example Session

```
User: /trawl plan ./contracts "extract all payment terms, deadlines, and penalty clauses"

Claude: Checking logbook... found LOGBOOK.md (project: acme-migration)

        Scanning ./contracts...
        Found: 8 PDF files, ~240 pages total (~480,000 chars)

        Trawl plan:
        - Source: 8 contract PDFs
        - Query type: extraction
        - Chunk strategy: by file (each PDF = one chunk)
        - Agents: 8 (protocol-aware)
        - Output: payment_term, deadline, penalty_clause per finding

        Creating TRAWL.md and trawl-T-0001/...
        Updating logbook Current State — added T-0001 to "What's in flight"
        Done. Run /trawl run to execute.

---

User: /trawl run

Claude: Logbook check: no other active sessions. Adding trawl coordinator.

        Spawning 8 protocol-aware agents...
        Each agent reads TRAWL.md, processes its chunk, writes to trawl-T-0001/

        C-001 acme-2025.pdf       → trawl-T-0001/C-001.yaml   done   3 findings
        C-002 globex-2024.pdf     → trawl-T-0001/C-002.yaml   done   5 findings
        C-003 initech-2025.pdf    → trawl-T-0001/C-003.yaml   done   2 findings  1 flag
        C-004 umbrella-2026.pdf   → trawl-T-0001/C-004.yaml   done   4 findings
        C-005 wayne-2025.pdf      → trawl-T-0001/C-005.yaml   empty  0 findings
        C-006 stark-2024.pdf      → trawl-T-0001/C-006.yaml   done   6 findings  2 flags
        C-007 oscorp-2025.pdf     → trawl-T-0001/C-007.yaml   failed
        C-008 lexcorp-2026.pdf    → trawl-T-0001/C-008.yaml   done   3 findings

        7/8 agents complete. 23 findings across 6 files. 3 flags raised.

        Processing flags:
        - C-003 flagged issue (MED): "Payment terms reference master agreement in C-002"
        - C-006 flagged note: "Contract references superseded policy v2.1"
        - C-006 flagged decision: "Penalty clause has two interpretations depending on jurisdiction"
        - C-007 failed after retry → created logbook issue I-004

        Synthesizing (from 7 successful chunks)...
        Done. Run /trawl results to view.

---

User: /trawl results

Claude: Trawl: T-0001
        Source: 8 contract PDFs (~240 pages)
        Query: extract payment terms, deadlines, penalty clauses

        Chunks: 8 (6 done, 1 empty, 1 failed)
        Findings: 23

        Payment terms:
        - Acme: Net 30, invoiced monthly...
        - Globex: Net 45, quarterly in arrears...
        [...]

        Deadlines:
        - Initech: Delivery by 2025-09-01...
        [...]

        Penalty clauses:
        - Umbrella: 1.5% per month on overdue...
        [...]

        Boundary flags: 1
        - C-003: payment terms reference master agreement (confirmed in C-002)

        Agent flags:
          Issues raised: 2 (1 processed → I-004, 1 → I-005)
          Decisions flagged: 1 (pending)
          Notes: 1

        Unprocessed decision from C-006:
          "Penalty clause has two interpretations depending on jurisdiction"
          Options: A) UK interpretation (strict) | B) DK interpretation (lenient)
          Record this decision? (/logbook decide)

---

User: yes, let's go with DK interpretation — and also decide on Acme's payment terms as standard

Claude: Recording decisions...
        D-007 — Penalty clause jurisdiction: DK interpretation (lenient)
        D-008 — Standard payment terms: Acme Net 30
        Both linked to trawl T-0001.

        Writing logbook session entry...
        Session logged. Logbook updated.
        Trawl T-0001 complete.
```
