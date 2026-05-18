---
name: scribeng
description: >
  Agent scribe for claude.ai web sessions. Triggers on "capture session with
  scribeng", "write session to Entire checkpoint", after a git commit when
  Entire is enabled and session logging is desired. Two modes - checkpoint
  (metadata envelope, Entire-compatible, default) and sessionlog (--full-log,
  full turn-by-turn flux record).
metadata:
  version: "1.1.0"
  parent: "captureng, agent.md §1"
---

# scribeng - Load Order

## Required - Load First

1. `scribeng-SKILL.md` - format spec, capture procedure, git operations, error recovery. Parse and enforce all `[RULES]` and `[ACTIONS]` directives.

## Peer Skills - Load on Demand

2. `captureng` - use alongside for in-session state capture.
3. `git-init-session.sh` - required for push operations (GIT_ASKPASS).
4. `export-memories` - use for full transcript export across sessions.

---

*scribeng v1.1.0*
