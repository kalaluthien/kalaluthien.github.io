---
share: true
title: Personal APK distribution
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-16
updated: 2026-07-17
sources:
  - "[[2026-07-16-automated-apk-delivery-from-mac-to-phone]]"
  - "[[2026-07-17-tailscale-for-mac-phone-and-ai-agent-workflows]]"
aliases:
  - obtainium
  - apk-delivery-channels
---

# Personal APK distribution

Getting a self-built APK from a build machine onto one's own Android phone
over the internet, scriptably and without stores. Platform install mechanics
(permissions, silent updates, Auto Blocker, Shizuku) live in
[Android sideloading and silent updates](/wiki/android-sideloading-and-silent-updates/).

## Obtainium — the phone-side keystone

[Obtainium](https://github.com/ImranR98/Obtainium) (actively maintained;
v1.6.9 on 2026-07-16) watches arbitrary sources for new APK versions,
notifies, and installs.

- **Sources**: GitHub/GitLab/Codeberg releases, F-Droid repos, Jenkins,
  direct APK link, and a generic **HTML source** that scrapes any page or
  directory listing for APK links (link-filter regex; version detection via
  link/ETag/partial-APK hash or a version-extraction regex). Custom request
  headers supported. Cleartext HTTP sources work (broken TLS is the problem
  case, not plain HTTP).
- **Private GitHub repos**: works with a **classic PAT (`repo` scope)**
  entered in the GitHub source settings — the token is sent on asset
  downloads too (fixed in v0.14.11-beta,
  [issue #857](https://github.com/ImranR98/Obtainium/issues/857)). Avoid
  fine-grained tokens: open 403 bug
  ([#2694](https://github.com/ImranR98/Obtainium/issues/2694)).
- **Update flow**: background poll (default 6 h) → notification → ~2–3 taps;
  with Shizuku as installer and background updates enabled, **zero-tap
  silent updates** (conditions: Android 12+, Obtainium is installer-of-record,
  app targets a recent SDK). Background installs are async and best-effort —
  **failures produce no notification**.

## Distribution rails, ranked by fit

- **Private GitHub Releases** — least infrastructure: `gh release create
  v<x> app.apk` from the build machine; 2 GiB/asset, no bandwidth caps; pair
  with Obtainium's GitHub source + classic PAT. A dedicated assets-only
  private repo keeps source off GitHub.
- **[Tailscale](/wiki/tailscale/) serve + Obtainium HTML** — strongest privacy
  (APK never leaves own infra), works over cellular via NAT traversal/DERP.
  Free plan (post April 2026): 6 users, unlimited devices. Gotchas: serving
  files/directories headlessly requires the **open-source `tailscaled`**
  variant (GUI variants only proxy ports — front a `python3 -m http.server`
  instead); Android allows one active VPN at a time. (Both plan limits and
  the variant gotcha re-verified against tailscale.com on 2026-07-17.)
- **Personal F-Droid repo** — the only **zero-tap path without Shizuku**:
  F-Droid client 1.19+ (2024-02) does unattended background updates on
  Android 12+ for apps it installed (installer-of-record). Cost: highest
  setup (fdroidserver via brew/pip, repo keystore, `fdroid update` per
  release; least tested on macOS). Alternative clients: Droid-ify, Neo Store.
- **Firebase App Distribution** — free, 2 GiB limit, CLI/Gradle upload,
  tester email + App Tester app; ~3–4 taps per release, no silent updates,
  every build goes to Google. Weak fit for a solo self-owned workflow.
- **Cloudflare Tunnel** — named tunnels need a domain on Cloudflare and TLS
  terminates at their edge; quick tunnels mint a random URL per process
  (unusable as a stable watcher source). ngrok free: 1 GB/month cap +
  interstitial (bypassable via header). Both dominated by Tailscale here.

## Consumer channels

- **Taildrop** ([Tailscale](/wiki/tailscale/)): `tailscale file cp camera.apk <phone>:` — the
  APK lands in the phone's Downloads with a notification, over the tailnet
  from anywhere (no shared network or wireless-debugging session, unlike
  adb). Manual push, no watcher, system-installer confirm every time — the
  right tool for one-off debug builds, not a steady-state update rail.
- **Telegram bot**: `sendDocument` takes any file type; hosted Bot API caps
  uploads at **50 MB** (self-hosted `telegram-bot-api` server: 2000 MB). One
  curl per release; phone gets a real push; ~4 taps to installed (Telegram
  is the install-unknown-apps grantee). Best as a "build shipped"
  notification bell alongside a real rail.
- **Gmail/email**: **dead end** — `.apk` is on Gmail's block list in both
  directions, including inside zips and password-protected archives
  ([policy](https://support.google.com/mail/answer/6590)). Outlook.com blocks
  it too.
- **Google Drive**: rclone upload works, but own uploads never notify, no
  watcher can poll Drive's tokenized download URLs (Obtainium has no Drive
  source), and Drive's scanner false-positives APKs. Dominated.

## Design rule of thumb

Notification, transport, and installer are separable concerns. The tap floor
is set by the installer path (system confirm dialog unless the
installer-of-record does a silent update — see
[Android sideloading and silent updates](/wiki/android-sideloading-and-silent-updates/)), not by the transport. So pick
the transport for scriptability and privacy, and spend the effort on the
installer side (Obtainium + Shizuku, or F-Droid repo) if zero-tap matters.
