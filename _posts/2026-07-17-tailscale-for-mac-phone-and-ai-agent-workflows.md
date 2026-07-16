---
share: true
title: "Tailscale: Your Mac and Your Phone on One Private Network"
date: 2026-07-17 00:10:00 +0900
categories:
  - Report
  - Infrastructure
tags:
  - macos
  - tailscale
author: claude
description:
---

## The short version

Tailscale puts your Mac and your S26 Ultra on one small private network that
behaves the same whether the phone is on your home Wi-Fi, on 5G in a cafe, or
abroad. Install it on both devices, log in with your Google account on each,
and from then on the phone can always reach the Mac at a stable name — no
router configuration, no public IP, nothing exposed to the internet.

For this setup it earns its place for three things, in this order:

1. **SSH into the Mac from anywhere** — attach to a tmux session running
   Claude Code from your phone and steer an overnight agent run from bed or
   a cafe.
2. **Showing dev servers** — `tailscale serve` turns `localhost:5173` on the
   Mac into an HTTPS URL your phone can open; `tailscale funnel` extends that
   to a friend on the public internet.
3. **Safe Wi-Fi** — route the phone's traffic through the Mac at home when
   you are on an untrusted network (exit node).

For APK delivery it is the runner-up you already evaluated: Taildrop is great
for throwing a one-off debug build at the phone, but it does not replace the
GitHub Releases + Obtainium pipeline from the 2026-07-16 report ("Automated
APK Delivery from a Mac to a Phone on 5G") — details below.

The free plan (6 users, unlimited devices since the April 2026 pricing
change) covers everything in this report. Ten-minute setup at the end.

## The problem it solves

Your Mac, by default, cannot be reached from outside your home. Three layers
conspire: your router does NAT, so the Mac has no address the internet can
route to; your ISP changes your home IP whenever it likes; and your phone on
5G sits behind carrier-grade NAT, so *it* cannot accept connections either.

The classic fixes are all small sysadmin jobs that never stay done. Port
forwarding means router configuration, an open port that internet scanners
find within minutes, and breakage every time your home IP changes. Dynamic
DNS patches the IP problem but not the exposure. A rented VPS as a relay
works but now you administer a VPS. Plain WireGuard is excellent but you
become its coordination server: generating keys, copying public keys between
devices by hand, updating endpoints when IPs change.

Tailscale's design goal is to delete that entire category of work.

## What it actually is

An app runs on each device. On first launch it generates a WireGuard keypair;
the private key never leaves the device. Then it registers with a
**coordination server** run by Tailscale, which works like a phonebook: it
stores each device's public key and current network location, and tells each
of your devices about all the others. That is essentially all it does — the
control plane "exchanges a few tiny encryption keys and sets policies," in
Tailscale's own words.

Your actual traffic does not go through Tailscale's servers. Once two devices
know each other's keys and locations, they talk **directly, peer-to-peer**,
encrypted with WireGuard. The coordination server could not read your traffic
even if it wanted to — it never has the private keys.

Direct connections work through most home routers via NAT traversal (both
sides send outbound packets simultaneously until a path opens — no port
forwarding needed, because nothing ever listens for unsolicited inbound
traffic). When both sides sit behind NATs too hostile for that — some
carrier-NAT combinations — traffic falls back to **DERP relays**
(Designated Encrypted Relay for Packets), servers Tailscale runs around the
world. The relay blindly forwards already-encrypted packets; it cannot
decrypt them. You pay latency, not privacy.

Two more pieces complete the picture:

- **Identity is your existing login.** You sign in with Google (or GitHub,
  etc.); there are no new accounts, no certificates to issue, no keys to copy
  around. A device is on your network because you logged into it. Everything
  the classic setups make you manage by hand — key distribution, endpoint
  tracking, identity — is what the coordination server automates.
- **Stable addresses and names.** Every device gets an IP in the
  `100.x.y.z` range (carrier-grade NAT space, so it never collides with your
  LAN) that never changes, plus a DNS name via **MagicDNS** like
  `your-mac.tailnet-name.ts.net`. `ssh your-mac` works from the phone
  regardless of which network either device is on. Real HTTPS certificates
  for those names come from Let's Encrypt via `tailscale cert` or
  automatically with Serve.

## What you would actually use it for

### SSH into the Mac — the AI-agent case

This is the headline feature for you. Enable Remote Login on the Mac (System
Settings → General → Sharing), install any SSH client on the phone (Termius
and friends), and `ssh your-mac` works from anywhere. The tailnet interface
is not reachable from the internet, so you get SSH without port forwarding,
without dynamic DNS, and without brute-force noise in your logs — your
existing SSH keys keep working unchanged.

Concretely: run Claude Code inside tmux on the Mac. From the phone, SSH in,
`tmux attach`, and you are looking at the live session — check on a
long-running agent, answer its permission prompt, kick off
`tools/release.sh` for the camera app, detach, pocket the phone. The same
applies to a self-hosted MCP server or any long-lived dev process on the
Mac: it becomes reachable from every device you own, from anywhere.

A note on the feature literally named "Tailscale SSH" (where tailscaled
itself answers on port 22 and authenticates you by tailnet identity, no SSH
keys at all): as a *server* on macOS it works only under the open-source
`tailscaled` variant, not the GUI apps. You do not need it — macOS's built-in
sshd over the tailnet gives you nearly all the value with any variant.

### Showing a dev server — Serve and Funnel

`tailscale serve 5173` takes a web app listening on localhost on the Mac and
makes it `https://your-mac.tailnet-name.ts.net` — real HTTPS, valid
certificate, visible only to devices on your tailnet. Open it on the phone to
test a UI on a real mobile screen. It also injects identity headers, so a
toy service can know who is calling without any auth code.

`tailscale funnel 5173` is the same idea aimed at the public internet — for
showing something to a friend who is not on your tailnet. Constraints, so you
are not surprised: still labeled beta, only ports 443/8443/10000, HTTPS only,
non-configurable bandwidth limits, and you must grant the `funnel` attribute
in the tailnet policy once. Fine for demos; not hosting.

One macOS quirk worth remembering: `tailscale serve` can also serve *files
and directories* directly (`tailscale serve /path/to/dir`), but due to app
sandbox limitations that mode works only on the open-source `tailscaled`
variant — the GUI variants proxy ports only. The workaround is trivial:
front the directory with `python3 -m http.server 8080` and serve the port.

### Taildrop — throwing an APK at the phone

Right-click the APK on the Mac → Share → Tailscale → pick the phone; or from
a script, `tailscale file cp dist/camera-0.5.0.apk your-phone:`. The phone
gets a notification and the file lands in Downloads (the Android app now asks
you to pick a save directory on first use). It works across the internet —
Mac at home, phone on 5G — and the file never touches a third-party server.
Own-devices only, labeled public alpha, no documented size limit.

Honest comparison with your existing pipeline: Taildrop is a manual push
with no version watching and no silent install — every delivery ends at the
system installer's confirm dialog, and Samsung Auto Blocker applies as
usual. The GitHub Releases + Obtainium + Shizuku pipeline stays the primary
rail (one command on the Mac, zero taps on the phone). Where Taildrop wins:
one-off debug builds you want on the phone *right now* without cutting a
release — and unlike `adb install`, it does not need the phone on the same
network or a wireless-debugging session. At your desk, adb is still faster;
away from your desk, Taildrop is the tool.

### Exit node — untrusted Wi-Fi through your home Mac

Advertise the Mac as an exit node (supported on the GUI variants; the Mac
must stay awake), and the phone can opt in with one toggle: all its traffic
then travels encrypted to your Mac and exits through your home connection.
Hotel or airport Wi-Fi sees only WireGuard noise. Costs: extra latency
(traffic makes a round trip through your home) and phone battery — routing
all traffic through an exit node is the one scenario Tailscale itself names
as the main cause of mobile battery drain.

### ACLs, in one paragraph

Tailscale has a policy file controlling which devices may talk to which. A
brand-new tailnet allows everything, and for a single-user tailnet — one
person, a Mac, a phone — that default is exactly right; do not write rules
you do not need. ACLs start mattering if you ever invite family or friends
onto your tailnet (the free plan now allows 6 users) or tag disposable
machines; cross that bridge then. The free plan caps you at 3 ACL groups,
which for a solo tailnet is 3 more than you will use.

## Honest limits

- **The coordination server is closed source and hosted by Tailscale.** The
  clients are open source (though the macOS GUI wrapper is not fully), DERP
  server code is open, but the control plane is theirs, and your network's
  metadata — device names, keys, connection times — lives on their servers.
  The self-hosted alternative is **Headscale**: an open-source
  reimplementation of the control server that official clients can log into.
  It is actively maintained (v0.29.2, July 2026) and Tailscale employs its
  lead maintainer precisely to support the project. The trade: you run and
  secure that server yourself — for one person, you would be reintroducing
  the sysadmin job Tailscale exists to delete.
- **Free plan limits** (since the 2026-04-08 pricing change): 6 users,
  unlimited devices, exit nodes and subnet routers included, "basic"
  Tailscale SSH up to 5 hosts, 3 ACL groups, 50 tagged resources. Paid starts
  at $8/user/month ("Standard"). Nothing in this report exceeds the free
  plan.
- **DERP latency.** When NAT traversal fails, you ride a relay: still
  private, noticeably slower. Check with `tailscale ping your-phone` —
  it tells you whether the path is direct or via DERP.
- **Android battery cost is nonzero.** An always-on VPN interface costs
  something even idle; Tailscale acknowledges mobile battery drain as a known
  issue "most commonly attributed" to exit-node use. Plain tailnet access
  (SSH, Taildrop, opening served URLs) is the cheap part; full-time exit-node
  routing is the expensive part. No trustworthy public measurements exist.
- **One VPN at a time on Android.** Tailscale occupies the system VPN slot,
  so it excludes any other VPN app while connected — already noted in the
  APK delivery report.
- **Funnel is beta** and narrowly scoped (three ports, bandwidth-limited,
  public). Treat it as a demo tool.

## Ten minutes to running, on exactly this hardware

1. **Mac**: `brew install --cask tailscale-app`. This is the Standalone GUI
   variant — the one Tailscale recommends starting with, and unlike the Mac
   App Store build it needs no Apple ID. Open Tailscale.app, log in with
   Google. For the CLI, enable "CLI integration" in the app's settings
   (installs `/usr/local/bin/tailscale` on macOS 13+); until then the binary
   lives at `/Applications/Tailscale.app/Contents/MacOS/Tailscale`. Do not
   start with the open-source formula (`brew install tailscale`) — it is
   headless and aimed at experienced admins; switch to it later only if you
   decide you want Tailscale SSH's keyless server or direct file serving.
2. **Phone**: Play Store → Tailscale → sign in with the same Google account →
   accept the VPN permission prompt.
3. **Admin console** (login.tailscale.com): MagicDNS is on by default for new
   tailnets; flip on "HTTPS Certificates" (needed for Serve). While there,
   disable key expiry for the Mac so its login never lapses while you are
   away from home.
4. **Verify**: `tailscale status` on the Mac should list the phone with a
   100.x address. `tailscale ping <phone-name>` shows whether you have a
   direct path or DERP. Enable Remote Login on the Mac and `ssh` in from the
   phone once, so the first use is not the day you need it.

## Sources

- How Tailscale works (control plane vs. data plane, key handling): [tailscale.com/blog/how-tailscale-works](https://tailscale.com/blog/how-tailscale-works)
- Open-source boundaries (clients open, coordination server closed): [tailscale.com/opensource](https://tailscale.com/opensource)
- macOS variants (Standalone / App Store / open-source tailscaled): [tailscale.com/kb/1065/macos-variants](https://tailscale.com/kb/1065/macos-variants); CLI locations: [tailscale.com/kb/1080/cli](https://tailscale.com/kb/1080/cli); Homebrew: [formula](https://formulae.brew.sh/formula/tailscale), [cask tailscale-app](https://formulae.brew.sh/cask/tailscale-app)
- Pricing v4 (2026-04-08: free plan 6 users/unlimited devices, Standard $8): [tailscale.com/blog/pricing-v4](https://tailscale.com/blog/pricing-v4); current table: [tailscale.com/pricing](https://tailscale.com/pricing)
- Taildrop (alpha status, own-devices only, Android receive flow): [tailscale.com/kb/1106/taildrop](https://tailscale.com/kb/1106/taildrop)
- Tailscale SSH (mechanism; macOS server requires open-source variant): [tailscale.com/kb/1193/tailscale-ssh](https://tailscale.com/kb/1193/tailscale-ssh)
- Serve (tailnet HTTPS, identity headers; file-serving needs tailscaled on macOS): [tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve), [tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- Funnel (beta; ports 443/8443/10000; bandwidth limits; funnel attribute): [tailscale.com/kb/1223/funnel](https://tailscale.com/kb/1223/funnel)
- MagicDNS and addressing: [tailscale.com/kb/1081/magicdns](https://tailscale.com/kb/1081/magicdns), [tailscale.com/kb/1015/100.x-addresses](https://tailscale.com/kb/1015/100.x-addresses); HTTPS certs: [tailscale.com/kb/1153/enabling-https](https://tailscale.com/kb/1153/enabling-https)
- Exit nodes (platform support, Android caveats): [tailscale.com/kb/1103/exit-nodes](https://tailscale.com/kb/1103/exit-nodes); mobile battery: [troubleshooting/mobile/battery-drains](https://tailscale.com/docs/reference/troubleshooting/mobile/battery-drains)
- DERP relays (encrypted blind forwarding, region selection): [tailscale.com/kb/1232/derp-servers](https://tailscale.com/kb/1232/derp-servers)
- Headscale (self-hosted control server; v0.29.2 2026-07-01; Tailscale's stance): [headscale.net](https://headscale.net/stable/), [github.com/juanfont/headscale](https://github.com/juanfont/headscale), [tailscale.com/opensource](https://tailscale.com/opensource)
- Prior vault report: "Automated APK Delivery from a Mac to a Phone on 5G" (2026-07-16, `reports/`) — the pipeline Taildrop is compared against.
