---
name: scribeng
version: 1.2.1
scope: session, agent
parent: captureng, agent.md §1
description: Agent scribe for claude.ai web sessions. Triggers on "capture session with scribeng", "write session to Entire checkpoint", after a git commit when Entire is enabled and session logging is desired. Two modes: checkpoint (metadata envelope, Entire-compatible, default) and sessionlog (--full-log, full turn-by-turn flux record from session start).
---

# scribeng

*The agent as scribe of its own relational history.*

Two modes serving the same underlying workflow - developer works with Claude in the web sandbox and wants the session recorded:

- **Checkpoint mode (default):** metadata envelope committed to `entire/checkpoints/v1`. Entire-compatible. Triggered at/after commit.
- **Full-log mode (`--full-log` / `activate sessionlog`):** complete flux record accumulated from session start. Every R formation written incrementally. Full transcript embedded in checkpoint at commit time.

## This skill does NOT

- Run `entire` CLI hooks (those require an active Claude Code process)
- Capture full transcript text (cost-prohibitive; use `export-memories` for transcript export)
- Require Entire auth or keyring (operates directly on git)
- Replace `captureng` - use both: captureng for in-session state, scribeng for Entire-linked git history

---

## Prerequisites

**[RULES]**

1. `entire enable` must have been run in the repo. Verify: `entire/checkpoints/v1` branch exists locally or on remote.
1. A commit must exist on the working branch to link the checkpoint to. If no commit yet, prompt user to commit first.
1. GIT_ASKPASS must be active (via `git-init-session.sh`) for any push operations. Never embed PAT in URL.

---

## Checkpoint Format

Derived from `entire` v0.6.1 reverse engineering (s08, 2026-05-18). Two files per checkpoint:

### Path structure

```
{cid[0:2]}/{cid[2:]}/metadata.json        # session-level summary
{cid[0:2]}/{cid[2:]}/0/metadata.json      # incremental checkpoint (index 0)
```

Where `cid` = first 12 hex chars of the commit SHA on `entire/checkpoints/v1` for this checkpoint.

### `metadata.json` (session-level)

```json
{
  "cli_version": "0.6.1",
  "checkpoint_id": "<cid>",
  "strategy": "manual-commit",
  "branch": "<working_branch>",
  "checkpoints_count": 0,
  "files_touched": ["<file1>", "<file2>"],
  "sessions": [
    {
      "metadata": "/<cid[0:2]>/<cid[2:]>/0/metadata.json",
      "prompt": "<first_user_message>"
    }
  ]
}
```

### `0/metadata.json` (incremental)

```json
{
  "cli_version": "0.6.1",
  "checkpoint_id": "<cid>",
  "session_id": "<session_name>",
  "strategy": "manual-commit",
  "created_at": "<ISO8601_UTC>",
  "branch": "<working_branch>",
  "checkpoints_count": 0,
  "files_touched": ["<file1>", "<file2>"],
  "agent": "claude-ai-web",
  "model": "<model_string>",
  "session_intent": "<first bullet of captureng §1 Knowledge Summary, or first user message>",
  "turn_id": "<first_12_chars_of_sha1_of_session_id>",
  "initial_attribution": {
    "calculated_at": "<ISO8601_UTC>",
    "agent_lines": <int>,
    "agent_removed": 0,
    "human_added": 0,
    "human_modified": 0,
    "human_removed": 0,
    "total_committed": <int>,
    "total_lines_changed": <int>,
    "agent_percentage": <0-100>,
    "metric_version": 2
  },
  "prompt_attributions": []
}
```

---

## Capture Procedure

**[ACTIONS]**

1. **Collect session metadata** from current context:
   - `session_id`: session name (e.g. `s08`)
   - `model`: current model string (e.g. `claude-sonnet-4-6`)
   - `created_at`: session start datetime (UTC ISO8601)
   - `working_branch`: `git branch --show-current` on target repo
   - `session_intent`: if a captureng checkpoint exists for this session, read the first bullet of §1 Knowledge Summary. Fallback: first user message of session.

2. **Derive files_touched** from git diff between the linked commit and its parent:
   ```bash
   git diff --name-only HEAD~1 HEAD
   ```
   If this is the first commit (no parent): `git diff --name-only HEAD`

3. **Calculate attribution** from git diff stats:
   ```bash
   git diff --stat HEAD~1 HEAD
   ```
   Sum insertions as `agent_lines` (all lines committed in this session are agent-authored). `total_committed` = total insertions + deletions. `agent_percentage` = agent_lines / total_lines_changed * 100.

4. **Derive turn_id**: `echo -n "<session_id>" | sha1sum | cut -c1-12`

5. **Get working branch HEAD SHA**: `git rev-parse HEAD` on trunk.

6. **Switch to checkpoints branch**:
   ```bash
   git checkout entire/checkpoints/v1
   ```

7. **Create checkpoint directory and write JSON files**:
   ```bash
   # Deterministic CID: blake3(session_id + created_at + head_sha), first 16 chars
   CID=$(echo -n "<session_id><created_at><head_sha>" | b3sum | cut -c1-16)
   mkdir -p ${CID:0:2}/${CID:2}/0
   ```
   Write `metadata.json` (session-level) and `0/metadata.json` (incremental) with values from steps 1-5. Include `session_intent` in `0/metadata.json`.

8. **Commit to checkpoints branch**:
   ```bash
   git commit \
     --message "Checkpoint: ${CID}" \
     --message "session: <session_id> branch: <working_branch>" \
     --trailer "Signed-off-by: claude-subagent <claude-subagent@users.noreply.github.com>"
   ```

8a. **Return to working branch and add trailer commit** (makes checkpoint discoverable via `entire checkpoint list`):
   ```bash
   git checkout <working_branch>
   git commit --allow-empty \
     --message "<working_branch>: link checkpoint <CID>" \
     --trailer "Entire-Checkpoint: <CID>" \
     --trailer "Signed-off-by: claude-subagent <claude-subagent@users.noreply.github.com>"
   ```
   Without this step, the checkpoint exists on the branch but is invisible to the Entire CLI and any compatible viewer.

9. **Push if PAT available**:
    Push both `entire/checkpoints/v1` and working branch to remote using GIT_ASKPASS pattern. If no PAT, surface: "Checkpoint committed locally. Push entire/checkpoints/v1 and <working_branch> to remote when PAT available."

---

## Minimum Viable Output

Token budget critically low: emit only the two JSON file contents as inline code blocks with path labels. Do not attempt git operations. User copies manually.

Error during git operations (branch not found, dirty working tree): halt, surface specific error, restore original branch (`git checkout <original_branch>`), do not leave repo in detached HEAD state.

---

## Sessionlog Mode (Full Flux Capture)

Activated by `--full-log` or `activate sessionlog` at any point in the session. The agent writes its own R history incrementally to `/home/claude/session-log.jsonl` from the activation turn onward. At checkpoint time, the log file is embedded as a blob in the `entire/checkpoints/v1` commit.

Mid-session activation is explicitly valid. Turns prior to activation are not captured by this channel - use `export-memories` for retrospective coverage of pre-activation turns. Forward capture from activation point is sufficient and valuable independently.

**Why incremental over retrospective:** the full transcript lives in Ψ (the platform's storage). Selective tier channels (`conversation_search`, `recent_chats`) are flux-bounded - they return snippets, not verbatim content. Retroactive full retrieval is structurally impossible through these channels. Incremental forward capture is the only zero-loss path from activation onward.

### Trigger

**[RULES]**

1. Trigger phrases (case-insensitive): `activate sessionlog`, `--full-log`, `enable full log`. Scope: remainder of session.
1. When active, include `sessionlog: active` in all sub-agent handoff contexts. Sub-agents receiving this token must continue appending to the log before any output.
1. Agent must not skip append steps to save tokens. Logging is non-negotiable when sessionlog is active.
1. Each append is a mandatory bash_tool call - not optional prose. Omitting it loses the event permanently. Skipping is a spec violation.
1. Append failure (bash_tool error, disk full, path missing): emit `⚠️ sessionlog append failed: [error]` inline. Do not silently continue. Attempt recovery (recreate file if missing); resume appending next turn.

### Log Format

JSONL, one object per event, appended to `/home/claude/session-log.jsonl`.

```jsonl
{"event":"session_start","session_id":"<id>","model":"<model>","datetime":"<ISO8601>","first_prompt":"<text>"}
{"event":"session_gap","turns_missed":<int>,"reason":"activated at turn N; prior turns not captured","datetime":"<ISO8601>"}
{"event":"user_turn","turn":1,"content":"<verbatim user message>","datetime":"<ISO8601>"}
{"event":"agent_turn","turn":1,"content":"<verbatim agent response>","datetime":"<ISO8601>"}
{"event":"tool_call","turn":1,"tool":"<name>","input":"<summary or full>","output":"<summary or full>","datetime":"<ISO8601>"}
{"event":"file_write","path":"<path>","blake3":"<8-char>","bytes":<int>,"datetime":"<ISO8601>"}
{"event":"session_end","turn_count":<int>,"duration_est":"<HH:MM>","commit_sha":"<sha>"}
```

**[ACTIONS]**

1. On trigger: create `/home/claude/session-log.jsonl`; write `session_start` event. If activated mid-session (turn > 1): immediately write `session_gap` event with `turns_missed` = number of prior turns not captured.
1. After each user message: append `user_turn` event via bash_tool.
1. After composing agent response, before delivering: append `agent_turn` event via bash_tool.
1. After each tool call: append `tool_call` event via bash_tool.
1. After each file write tracked in registry: append `file_write` event via bash_tool.
1. At checkpoint time (scribeng triggered): write `session_end` event; read log; embed as `session-log.jsonl` blob alongside `0/metadata.json` in `entire/checkpoints/v1` commit; add `"transcript_file": "/<cid[0:2]>/<cid[2:]>/0/session-log.jsonl"` to `0/metadata.json`.

### Updated `0/metadata.json` when sessionlog active

Add field:
```json
"transcript_file": "/<cid[0:2]>/<cid[2:]>/0/session-log.jsonl"
```

The transcript blob is committed alongside `metadata.json` on `entire/checkpoints/v1`:
```
{cid[0:2]}/{cid[2:]}/0/metadata.json
{cid[0:2]}/{cid[2:]}/0/session-log.jsonl    ← added when sessionlog active
```

---

## Strategy Checklist

- [x] Processing (I): transforms session R history into Entire-compatible git objects, optional full flux log
- [x] Mediation (II): serves "developer wants this web session recorded at chosen fidelity" workflow
- [x] Forgetting (III): checkpoint mode captures envelope only; full-log is opt-in; no Entire auth; no Claude Code hooks
- [x] Integrity (IV): all behaviors declared; both modes fully specified; git operations explicit; log format declared
- [x] Resilience: minimum viable output defined; error recovery specified; sessionlog degrades to summary under token pressure
- [x] Under 500 lines

---

*scribeng-SKILL.md v1.2.1*
