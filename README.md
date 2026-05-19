# scribeng

*The agent as scribe of its own relational history.*

Agent scribe for claude.ai web sessions. Two modes: checkpoint (metadata envelope, Entire-compatible, linked to a git commit on `entire/checkpoints/v1`) and sessionlog (`--full-log`, full turn-by-turn flux record accumulated from session start). Bridges Entire CLI's Claude Code integration to the claude.ai web and sandbox environment.

## Set of files

| File | Purpose |
|---|---|
| `SKILL.md` | Router - load order |
| `scribeng-SKILL.md` | Content - format spec, capture procedure, git operations, error recovery |
| `scribeng.skill` | Packaged archive for upload to SKILL directory |
| `.github/workflows/build-skill.yml` | CI workflow - auto-builds and publishes `scribeng.skill` on push |
| `evals/evals.json` | Test cases for skill evaluation |
| `README.md` | This file |

## Install

- **claude.ai web platform:** upload `scribeng.skill` via skill settings. Add to any project using git repos with `entire enable` active.
- **Claude Code desktop app:** copy contents of this folder into `~/.claude/skills/scribeng/`.

## When It Triggers

- User says "capture session as Entire checkpoint", "write session to entire", "scribeng"
- After a git commit in a session where Entire is enabled and session logging is wanted
- Explicitly: "save this session to entire/checkpoints/v1"

## What It Produces

Two `metadata.json` files committed to `entire/checkpoints/v1`, compatible with `entire checkpoint list`, `entire session`, and the entire.io web viewer:

```
{cid[0:2]}/{cid[2:]}/metadata.json       # session-level summary
{cid[0:2]}/{cid[2:]}/0/metadata.json     # incremental checkpoint
```

Where `cid` = first 16 hex chars of `blake3(session_id + created_at + head_sha)` - a deterministic, content-addressable identifier derived from the session, its start time, and the HEAD commit it is linked to. This format was derived from `entire` v0.6.1 (2026-05-18).

## What It Does Not Do

- Does not run `entire` CLI hooks (requires active Claude Code process)
- Does not capture full transcript text (use `export-memories` for that)
- Does not require Entire auth or keyring
- Does not replace `captureng` - use both together
- Has not been tested on AI-enabled platform+harness+model systems other than claude.ai web - adaptations for other systems are welcome from the wider ecosystem

## Peer Skills

- **[prompteng](https://github.com/ecological-codes/prompteng)** - core prompt engineering config; session init framework scribeng operates within
- **[captureng](https://github.com/ecological-codes/captureng)** - in-session state snapshot; use alongside scribeng
- **[packageng](https://github.com/ecological-codes/packageng)** - `.skill` file validation + packaging
- **[safe-skill-creator](https://github.com/ecological-codes/safe-skill-creator)** - skill design + iteration
- **[export-memories](https://github.com/ecological-codes/export-memories)** - cross-session transcript synthesis

## Prerequisites

1. `entire enable` run in the target repo (`entire/checkpoints/v1` branch must exist)
2. At least one commit on the working branch to link the checkpoint to
3. `git-init-session.sh` sourced if push to remote is needed (GIT_ASKPASS pattern)
4. `b3sum` available for CID generation (`apt-get install -y b3sum`)
5. **[agent.md](https://github.com/ecological-codes/user-prefs/blob/trunk/agent.md)** loaded - provides `session_id`, datetime, and model context scribeng depends on

## Style Convention

Directives use the following section headers with numbered lists:

- **[RULES]** - enforceable constraints applied at runtime.
- **[ACTIONS]** - autonomous steps agent executes in normal workflow.

## References

- Inspired by [https://github.com/entireio](https://github.com/entireio)

## License

See [LICENSE](./LICENSE). (C) Copyright 2026 - Sameer Khan

---
README.md v1.0.0 - Human Approved
