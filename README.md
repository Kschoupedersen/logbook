# Harness

A workflow harness for AI-assisted project work in [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Three skills that compose: a logbook for project memory, a trawl for multi-agent document retrieval, and a loop for recurring tasks.

## Skills

| Skill | What it does |
|-------|-------------|
| **logbook** | Session tracking, decisions, issues, and handoff notes. Any engineer or agent can pick up cold. |
| **trawl** | Coordinated multi-agent retrieval across documents too large for one context window. Protocol-aware subagents extract structured findings in parallel. |
| **ralph** | Logbook-grounded loop for recurring or long-running tasks. Reads project state, works iteratively, writes session entries on completion. |

## Install

Copy the skill files into your Claude Code skills directory:

```
logbook.md  →  ~/.claude/commands/logbook.md
trawl.md    →  ~/.claude/commands/trawl.md
ralph.md    →  ~/.claude/commands/ralph.md
```

Trawl also requires a custom subagent definition:

```
~/.claude/agents/trawl-worker.md
```

See `trawl.md` for the worker protocol. Restart your session after adding the agent.

## Usage

```
/logbook init [path]    — Create a LOGBOOK.md for a project
/logbook start          — Session start protocol
/logbook end            — Session end protocol
/logbook audit          — Verify Current State against actual files
/logbook status         — Quick summary of what's in flight
/logbook issue [desc]   — Add a known issue
/logbook decide [title] — Record a decision

/trawl plan [source] [query]  — Define a trawl mission
/trawl run                    — Spawn agents and collect results
/trawl results                — View synthesized findings
/trawl rerun [chunk_ids]      — Re-trawl specific chunks

/ralph <prompt>                        — Start a logbook-grounded loop
/ralph <prompt> --max-iterations N     — Loop with iteration limit
```

## How they compose

- **Trawl requires logbook** — trawl runs are tracked as logbook sessions. Findings that need decisions flow into the logbook's decision log. Failed chunks become known issues.
- **Ralph reads logbook** — the loop grounds itself in Current State before starting work, and writes a session entry when done.
- **Logbook stands alone** — works without the other two. Add trawl and ralph when you need them.

## License

MIT — see [LICENSE](LICENSE).
