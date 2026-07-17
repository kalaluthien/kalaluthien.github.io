---
share: true
title: Sharing Dialog Between Claude Code Sessions
categories:
  - Report
  - Claude Code
tags:
  - claude-code
  - agents
  - workflow
author: claude
---

> What is current best practice for sharing dialog between Claude Code sessions without introducing too many dependencies?

## Short answer

**Files first, servers last.** Filesystem-based approaches deliver about 90% of the value while staying inspectable with plain `cat` — reach for MCP/servers only once you actually need cross-machine or programmatic access. Match the mechanism to the case: `--resume`/`--continue` for one thread continued over time, `/export` or a summary doc for a one-time handoff, a file mailbox for two live sessions. Findings current as of 2026-07-12.

## Ranked options (lowest → highest dependency)

1. **Built-in resume/continue (zero dependency)** — `claude -c` / `--continue` resumes the most recent session for the current working directory; `claude -r` / `--resume` picks a specific session by ID/name via an interactive picker. Restores full fidelity (every prompt, tool call, tool result, response). Right choice when "two sessions" are really the *same* thread continued over time.
   - Gotcha: sessions are stored under `~/.claude/projects/<encoded-cwd>/*.jsonl`, keyed to the working directory. Resuming from a different directory won't find the session — context does not carry over unless you resume the right one.

2. **`/export` + hand off (zero dependency)** — dumps the current conversation (messages + tool output) to a file or clipboard as readable text. Best for a one-time handoff to another session or archiving a decision log.

3. **Shared memory / summary doc (near-zero dependency)** — at the end of a session, have Claude write a decisions/context summary to a shared markdown file that the next session reads. This is the lightweight way to share *distilled* context rather than a raw transcript (matches the `MEMORY.md` pattern already in use).

4. **File-based mailbox for live two-way communication (one small dependency)** — for two sessions running *concurrently* that need to message each other, the emerging pattern (see `session-bridge`, blog.shreyaspatil.dev):
   - Each session gets a directory at `~/.claude/session-bridge/sessions/<id>/` with `inbox/` and `outbox/` subfolders holding messages as JSON files.
   - Messages carry a status field that atomically transitions `pending → read`, preventing duplicate processing across polling instances.
   - Deliberately minimal implementation: 9 bash scripts + `jq` — no Node.js, no Python, no runtime/server.
   - Design rationale (author, quoted): files were chosen over MCP or a local HTTP server for debuggability — "literally `cat` a message to see what happened. No logs needed. No server to start."
   - Limitations: single-machine only, ~5-10s end-to-end latency due to polling, context depth depends on what's active in that session's own conversation.

5. **MCP / shared SessionStore adapter (heaviest — only if needed)** — for cross-machine or serverless sharing, mirror transcripts to shared storage via a `SessionStore` adapter (Agent SDK). More infra than the problem usually warrants unless you genuinely need multi-device or programmatic session access.

## Decision rule

- Same logical thread → `--resume` / `--continue`.
- One-time handoff → `/export` or a shared summary markdown.
- Two live sessions coordinating → file-mailbox approach (bash + `jq`), not MCP.
- Only adopt MCP/shared-store once you need to cross machines.

## Pitfalls / gotchas

- Claude Code session context does **not** persist across sessions by default — isolated sessions won't automatically remember previous work without deliberate sharing (resume, export, or summary doc).
- Session storage is directory-keyed (`~/.claude/projects/<encoded-cwd>/*.jsonl`); running `--resume`/`--continue` from the wrong working directory means it can't find the session.
- The file-mailbox approach only works on a single machine and has several-second polling latency — not suitable for cross-machine or low-latency coordination.

## Sources

code.claude.com/docs (sessions, agent-sdk/sessions), kentgigger.com conversation-history post, blog.shreyaspatil.dev session-bridge post, support.claude.com chat search/memory article.
