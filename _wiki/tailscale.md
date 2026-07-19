---
share: true
title: Tailscale
categories:
  - Wiki
tags:
  - tailscale
  - networking
  - ssh
  - vpn
author: claude
type: concept
created: 2026-07-17
updated: 2026-07-18
sources:
  - "[[2026-07-17-tailscale-for-mac-phone-and-ai-agent-workflows]]"
  - "[[2026-07-18-reaching-a-herdr-session-from-mobile-over-tailscale]]"
aliases:
  - mesh-vpn
  - taildrop
  - tailnet
---

# Tailscale

A mesh VPN that puts one's own devices on a private overlay network (a
*tailnet*). Relevant here as the remote-access layer between the Mac and the
Galaxy S26 Ultra, and as the runner-up APK rail in
[Personal APK distribution](/wiki/personal-apk-distribution/). Facts verified 2026-07-17.

## Architecture

- **Control plane vs data plane**: a hosted coordination server stores each
  device's WireGuard *public* key, current endpoints, and policy — nothing
  else; private keys never leave the device. Actual traffic is peer-to-peer
  [WireGuard](https://tailscale.com/blog/how-tailscale-works).
- **NAT traversal, then relays**: direct connections are punched through most
  NATs without port forwarding. Fallback order since 2026: self-hosted **peer
  relays** (GA 2026-02) → Tailscale-run **DERP** relays, which blindly
  forward already-encrypted packets (privacy preserved, latency paid).
  `tailscale ping <device>` reveals direct vs relayed.
- **Identity**: SSO login (Google/GitHub/…) — no new accounts, keys, or
  certificates to manage. Devices get stable IPs in `100.64.0.0/10` (CGNAT
  range) and **MagicDNS** names `<machine>.<tailnet>.ts.net`; Let's Encrypt
  certs for those names via `tailscale cert` or automatically with Serve.
- **Open-source boundary**: clients open source (macOS/iOS/Windows GUI
  wrappers not fully), DERP server open, **coordination server closed**.
  Self-hosted alternative: [Headscale](https://headscale.net) — actively
  maintained (v0.29.2, 2026-07), lead maintainer employed by Tailscale,
  works with official clients.

## macOS variants — the recurring gotcha

Three installs with different capabilities
([kb/1065](https://tailscale.com/kb/1065/macos-variants)):

| Variant | Install | GUI | Tailscale SSH server | `serve` files/dirs |
|---|---|---|---|---|
| Standalone (recommended start) | `brew install --cask tailscale-app` | yes | no | no (ports only) |
| Mac App Store | App Store (Apple ID) | yes | no | no (ports only) |
| Open-source `tailscaled` | `brew install tailscale` | no | yes | yes |

- GUI variants' CLI: `/Applications/Tailscale.app/Contents/MacOS/Tailscale`;
  the Standalone app can install `/usr/local/bin/tailscale` (macOS 13+).
- File/directory serving on GUI variants is blocked by the app sandbox —
  front a `python3 -m http.server` and serve the port instead.
- Plain macOS Remote Login (sshd) over the tailnet works on **any** variant
  and replaces most of what Tailscale SSH's keyless server would add. This is
  the rail for reaching a [herdr](/wiki/herdr/) session from a phone — SSH in over the
  tailnet, run `herdr`; walkthrough in
  [Reaching a herdr Session from Mobile over Tailscale](/posts/reaching-a-herdr-session-from-mobile-over-tailscale/).

## Features and their constraints

- **Taildrop** (file push between own devices): GUI share menu or
  `tailscale file cp <file> <device>:`; lands in Android Downloads with a
  notification. Public alpha, own-devices only, no documented size limit.
  As an APK channel: manual, no watcher, system-installer confirm every time
  — see [Personal APK distribution](/wiki/personal-apk-distribution/) for where it ranks.
- **Serve**: expose a local port tailnet-only over HTTPS with identity
  headers. **Funnel**: expose to the public internet — beta, ports
  443/8443/10000 only, bandwidth-limited, needs the `funnel` node attribute.
- **Exit nodes**: route a device's whole traffic through another (e.g. phone
  through home Mac on untrusted Wi-Fi). macOS can advertise on all variants
  (keep it awake); Android as exit-node *user* is fine, as exit node "not
  performant". Exit-node use is the main documented mobile battery cost.
- **Tailscale SSH**: tailscaled answers port 22 with tailnet identity, no
  keys. Server: Linux + macOS-tailscaled-only. Free plan: "basic", 5 hosts.
- **ACLs**: default allow-all; a single-user tailnet needs none. Free plan
  caps at 3 groups.
- Android holds the single system VPN slot while connected — excludes other
  VPN apps (see [Android sideloading and silent updates](/wiki/android-sideloading-and-silent-updates/) context).

## Plans (pricing v4, 2026-04-08)

Free "Personal": **6 users, unlimited devices**, exit nodes and subnet
routers included, 50 tagged resources, 1,000 ephemeral-resource
minutes/month. Personal Plus retired; "Starter" renamed **Standard**
($8/user/mo); Premium $18. Solo use fits free entirely.
([pricing](https://tailscale.com/pricing),
[pricing-v4](https://tailscale.com/blog/pricing-v4))
