---
share: true
title: Reaching a herdr Session from Mobile over Tailscale
categories:
  - Report
  - Infrastructure
tags:
  - macos
  - android
  - tailscale
  - ssh
  - herdr
author: claude
---

> Check how to setup tailscale to connect herdr session on mobile. I've
> downloaded ssh client on my phone so local mac mini setup and some ip is
> what I need.

## Short answer

herdr does nothing special for mobile — it is a terminal multiplexer, so the
mobile story is the tmux story: **SSH into the Mac mini, then run `herdr`.**
Tailscale's only job is to hand the phone a stable private IP for that Mac
mini, so the SSH client works from any network with no port-forwarding and no
public exposure. Two installs (Tailscale on the mini, Tailscale on the phone,
same account), one toggle (Remote Login on the mini), one address to type.

```
[Phone SSH client] ──Tailscale (WireGuard)──► [Mac mini] ──run──► herdr
        │                                          │
   Tailscale app,                          Tailscale + Remote Login,
   same account                            same account
```

The herdr guide is explicit that there is nothing more to it:

> SSH to the machine and run `herdr` there (works like tmux), or attach as a
> thin local client with `herdr --remote <host>`.

For what Tailscale is and its other uses on this hardware, see
[Tailscale: Your Mac and Your Phone on One Private Network](/posts/tailscale-for-mac-phone-and-ai-agent-workflows/).
This report is only the herdr-over-SSH path.

## Starting state on this Mac mini (2026-07-18)

Checked before writing, so the steps below are the actual gap, not a generic
checklist:

- **Tailscale:** not installed — no app, no CLI.
- **Remote Login (SSH):** not verifiable without admin; assume off until turned
  on.
- **MagicDNS name:** the mini's hostname, lowercased (shown as
  `<mac-hostname>` below).

## Mac mini setup (once)

### 1. Install and join Tailscale

```bash
brew install --cask tailscale
```

Open the app and sign in. That browser login is what joins the mini to your
tailnet — it cannot be scripted or done on your behalf.

### 2. Enable SSH (Remote Login)

```bash
sudo systemsetup -setremotelogin on
```

Equivalent to System Settings → General → Sharing → **Remote Login** on.

### 3. Read back the address to type on the phone

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale ip -4    # → 100.x.y.z
```

- **Tailscale IP:** always in the `100.64.0.0/10` range (`100.x.y.z`). Stable
  across networks, independent of any DNS.
- **MagicDNS name:** `<mac-hostname>` (the mini's hostname, lowercased) —
  easier to remember, works if MagicDNS is enabled in the tailnet admin console
  (default on).

### 4. Keep the mini reachable

A sleeping mini drops the connection. Either System Settings → Battery/Energy →
**Prevent automatic sleeping**, or:

```bash
sudo pmset -a sleep 0
```

## Phone setup

1. Install the **Tailscale** app, sign in with the **same account** as the
   mini, toggle it on.
2. In the SSH client, connect with:
   - **Host:** `100.x.y.z` (or `<mac-hostname>`)
   - **User:** your macOS account name (`whoami` on the mini)
   - **Port:** `22`
3. Once in: run `herdr`. The herdr server is persistent, so disconnect and
   reattach later lands you back in the same session — the tmux property that
   makes this worth doing from a phone.

## Recommendations

- **Use SSH keys, not a password.** Generate a key in the phone client, append
  its public key to `~/.ssh/authorized_keys` on the mini. Most mobile SSH
  clients (Termius, Blink, ShellFish) generate and store keys natively.
- **Consider Tailscale SSH** (`tailscale up --ssh`) to let the tailnet handle
  SSH auth instead of managing `authorized_keys` — but plain OpenSSH + keys is
  the simpler first mental model.
- **Named herdr sessions** (`herdr <name>`) give separate runtime namespaces if
  you want one persistent session per project rather than one shared default.

## What this does not need

- No port forwarding, no router config — Tailscale is an overlay, port 22 is
  never exposed to the public internet.
- No static IP or dynamic-DNS from your ISP — the `100.x.y.z` address is
  Tailscale's, not your home network's.
- No herdr-specific server config — herdr rides on whatever SSH session it
  finds itself in.

## Sources

- herdr agent guide — https://herdr.dev/agent-guide.md (remote section)
- herdr "how to work" — https://herdr.dev/docs/how-to-work/
- Live state of the Mac mini, checked 2026-07-18.
