# Logbook

A self-contained protocol for AI-assisted project work. Tracks sessions, decisions, issues, and backlog — so any engineer or agent can pick up where the last one left off, cold, without asking questions.

Built for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) as a skill.

## Install

Copy `logbook.md` into your Claude Code skills directory (e.g. `.claude/skills/` or wherever you keep custom skills).

## Usage

```
/logbook init [path]    — Create a LOGBOOK.md for a project
/logbook start          — Session start protocol
/logbook end            — Session end protocol
/logbook audit          — Verify Current State against actual files
/logbook status         — Quick summary of what's in flight
/logbook issue [desc]   — Add a known issue
/logbook decide [title] — Record a decision
```

## How it works

The logbook creates a `LOGBOOK.md` in your project that acts as living memory:

- **Session start/end protocol** — agents read context on start, write a handoff note on end. The file stays current by default.
- **"Next session start from:"** — mandatory field in every session entry. Eliminates cold-start overhead.
- **Threshold-based splitting** — sections auto-detect when they're getting too long and ask before archiving to child docs.
- **Staleness detection** — flags stale state and old issues so you don't trust outdated information.
- **Multi-engineer safety** — active session tracking so parallel workers don't step on each other.

## What's in the box

```
logbook.md   — The skill definition (drop this into your skills)
LICENSE      — MIT
README.md    — You're reading it
```

## License

MIT — see [LICENSE](LICENSE).
