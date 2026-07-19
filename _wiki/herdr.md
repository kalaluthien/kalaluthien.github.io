---
share: true
title: herdr
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-18
updated: 2026-07-18
sources:
  - "[[2026-07-18-reaching-a-herdr-session-from-mobile-over-tailscale]]"
aliases:
  - herdr session
---

# herdr

A terminal multiplexer that works like tmux
([docs](https://herdr.dev/docs/how-to-work/)): a persistent server holds
running sessions so a client can detach and reattach without losing state.
Relevant here as the thing reached over [Tailscale](/wiki/tailscale/) from a phone — the
session persistence is what makes a mobile connection worth keeping.

## Session model

- The **herdr server is persistent**. Disconnect and reattach later lands back
  in the same session — the tmux property that survives a dropped or
  backgrounded mobile link.
- **Named sessions**: `herdr <name>` opens a separate runtime namespace, giving
  one persistent session per project instead of one shared default.

## Running it

- **On the host**: SSH in and run `herdr` there. It rides on whatever SSH
  session it finds itself in and needs no herdr-specific server config.
- **As a thin client**: `herdr --remote <host>` attaches from the local machine
  instead ([agent guide](https://herdr.dev/agent-guide.md), remote section).

## Mobile access pattern

herdr does nothing special for mobile — the mobile story is the tmux story:
**SSH into the host, then run `herdr`.** [Tailscale](/wiki/tailscale/) supplies the stable
private IP (`100.x.y.z`, or a MagicDNS name) so a phone SSH client reaches the
Mac mini from any network with no port forwarding, and macOS Remote Login
answers port 22. Full walkthrough:
[Reaching a herdr Session from Mobile over Tailscale](/posts/reaching-a-herdr-session-from-mobile-over-tailscale/).
