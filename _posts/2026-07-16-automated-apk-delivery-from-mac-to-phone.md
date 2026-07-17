---
share: true
title: Automated APK Delivery from a Mac to a Phone on 5G
categories:
  - Report
  - Android
tags:
  - macos
  - android
  - workflow
author: claude
---

## Recommendation

**Private GitHub Releases as the distribution point, Obtainium on the phone as the watcher/installer, Shizuku as the silent-install backend.** One `gh release create` command on the Mac per release; zero taps on the phone per update in steady state.

- Mac side: `gh release create v<version> dist/camera-<version>.apk --repo <you>/camera-releases` — a dedicated private repo holding only release assets, so the camera source never leaves the Mac. `gh` is already installed and authenticated; GitHub allows 2 GiB per asset with no bandwidth caps, and the APK is 2.9 MB.
- Phone side: Obtainium (actively maintained, v1.6.9 released 2026-07-16) polls the GitHub source in the background (default every 6 h), authenticating with a **classic personal access token with `repo` scope** — token auth on private release *assets*, not just the API, has worked since v0.14.11-beta (issue #857, fixed 2023-09). Avoid fine-grained tokens: an open 403 bug (#2694) affects them.
- Silent updates: with Shizuku running and Obtainium's "Shizuku integration for elevated silent installs" plus background updates enabled, every versionCode bump installs with **zero taps** — Obtainium is the installer-of-record, the app targets SDK 36 (Android 16 requires the updated app to target API 34+ for user-action-free updates), and Shizuku supplies shell privilege. Without Shizuku the same pipeline still works at ~2–3 taps per update (notification → Install → confirm).
- The two recurring costs, stated plainly: **Samsung Auto Blocker must stay off** (it ships ON on current Galaxy flagships and has no per-app exception), and **Shizuku must be restarted after every reboot** (a system limitation with no non-root workaround). If the reboot ritual annoys more than one tap per update, drop Shizuku and keep everything else.

Runner-up if self-hosting is preferred: `tailscale serve` on the Mac Mini + Obtainium's HTML source — strongest privacy (the APK never leaves your infrastructure), same tap counts, but more moving parts (headless file serving requires the open-source `tailscaled` variant; Android allows only one active VPN). The only zero-tap path that needs no Shizuku is a personal F-Droid repo with the F-Droid client's unattended updates — but it has the highest setup cost of all options. Details below.

## Comparison

| Option | Setup | Mac per release | Phone taps/update | Auto-detects new version | Key failure mode | Privacy |
|---|---|---|---|---|---|---|
| **GitHub Releases + Obtainium (+ Shizuku)** | ~15 min | 1 command (`gh release create`) | **0** (2–3 without Shizuku) | Yes (poll + notification) | Fine-grained-PAT 403 bug — use classic PAT; Shizuku dies on reboot | Private repo on GitHub; PAT on phone |
| Tailscale serve + Obtainium HTML (+ Shizuku) | ~30–60 min | 1 script (cp + index) | **0** (2–3 without Shizuku) | Yes | Headless file-serve needs open-source tailscaled; one-VPN-at-a-time on Android | Best — never leaves own infra |
| Personal F-Droid repo + F-Droid 1.19+ | half a day | cp + `fdroid update` + sync | **0**, no Shizuku needed | Yes | fdroidserver least tested on macOS; F-Droid must be installer-of-record | Self-owned |
| Telegram bot (`sendDocument`) | ~10 min | 1 curl | ~4 | Push message | 50 MB hosted-API cap (not binding at 2.9 MB) | APK transits Telegram cloud |
| Firebase App Distribution | ~1 h | 1 CLI/Gradle command | 3–4 | Email + tester-app alert | No silent updates; Google sign-in ceremony for one person | Every build uploaded to Google |
| Google Drive (rclone) | ~15–30 min | 1 rclone command | ~5 **+ manual checking** | **No** — own uploads never notify | Malware false-positives block APK downloads | Google-hosted |
| Cloudflare named tunnel + Obtainium HTML | ~1 h + domain | 1 script | 2–3 | Yes | Public endpoint unless Access service tokens wired into headers | TLS terminates at Cloudflare |
| ngrok free | ~15 min | 1 script | 2–3 | Yes | 1 GB/month transfer cap; interstitial needs bypass header | Public endpoint |
| Gmail / email | — | — | — | — | **Dead end: .apk blocked outright** | — |

## The four candidate channels, evaluated

### Telegram bot — works, floor of ~4 taps

A bot sends any file type via `sendDocument`; there is no APK-specific restriction and no policy problem with a private bot messaging its owner. The hosted Bot API caps uploads at **50 MB** (the 2.9 MB camera APK fits with room to grow; past 50 MB you would self-host `telegram-bot-api`, which raises the limit to 2000 MB). Fully curl-scriptable:

```bash
curl -sS -F chat_id="$CHAT_ID" -F document=@"dist/camera-0.5.0.apk" \
  -F caption="camera v0.5.0" \
  "https://api.telegram.org/bot$BOT_TOKEN/sendDocument"
```

One-time setup: create the bot via @BotFather, **/start it from your own account** (bots cannot initiate conversations), read `chat_id` from `getUpdates`. Phone side, Telegram is merely the hand-off app — tapping the document fires an intent to the system package installer, with Telegram as the "install unknown apps" grantee. Steady state: notification → open document → Update → Done, **about 4 taps**. It is the best *pure-notification* channel — a real push the moment the build lands — but it can never go below the system installer's confirm dialog, so it loses to Obtainium the moment silent updates matter. Verdict: fine as a secondary "build ready" ping; not the primary rail.

### Gmail — verified dead end

`.apk` is explicitly on [Gmail's blocked file types list](https://support.google.com/mail/answer/6590), and the block covers compressed forms, files inside archives, **and password-protected archives with archived content**. Inbound mail carrying a blocked attachment is bounced to the sender — so receiving is blocked, not just sending. The only known bypass is extension renaming, which would demand a phone-side file-manager rename before every install. Outlook.com blocks `.apk` too (since April 2020). Eliminate email entirely.

### Google Drive — scriptable but blind

Upload is a solved problem (`rclone copy dist/camera-<v>.apk gdrive:apk/`; create your own OAuth client_id to avoid rclone's throttled shared one; the old `gdrive` CLI is abandoned). Drive stores and serves APKs, with two real failure modes: the scanner sometimes **false-positives an APK as malware and blocks mobile download entirely** (reported for exactly this use case), and files over 100 MB get an unscanned-file interstitial. The disqualifier is detection: Drive notifications fire only for files *someone else* shares with you — **your own upload to your own account triggers nothing**, you cannot share with yourself, and Obtainium has no Drive source (its HTML source cannot scrape Drive's auth/confirm-token download URLs). Polling a Drive folder by hand, ~5 taps per update. Verdict: strictly dominated by every other working option.

### A custom downloader/installer app — Android already caps what it could do

What a non-privileged installer app can legally do, verified against the [PackageInstaller.SessionParams docs](https://developer.android.com/reference/android/content/pm/PackageInstaller.SessionParams):

- With the `REQUEST_INSTALL_PACKAGES` ("install unknown apps") per-app grant, it can create a `PackageInstaller` session, stream in the APK, and `commit()` — which surfaces the system confirm dialog. That is the unavoidable floor for any **first** install.
- `setRequireUserAction(USER_ACTION_NOT_REQUIRED)` waives the dialog **only when all of the following hold**: the installer declares the (normal-level) `UPDATE_PACKAGES_WITHOUT_USER_ACTION` permission; the installer is the **update owner or installer-of-record** of the app being updated (i.e. this is an update, not a first install — Android 14's `ENFORCE_UPDATE_OWNERSHIP` lets the first installer claim exclusive silent-update rights); and the app being updated targets a recent SDK — **API 34+ on Android 16 ("Baklava")**, a bar the docs say will keep advancing. The camera app targets SDK 36, so it qualifies.
- Fully silent *first* installs without root exist only for device owners (MDM provisioning — requires a factory-fresh device, unusable on a personal daily driver) or via Shizuku's shell privilege.

The punchline: **a custom installer app would end up reimplementing exactly Obtainium**, which already does session-based installs, installer-of-record silent updates, Shizuku integration, and source watching. Building one buys nothing. Verdict: use Obtainium.

## Samsung and Android realities (2026, One UI on Android 16)

- **Auto Blocker ships ON** (default since One UI 6.1.1, July 2024, carried through One UI 7/8). While on, it "fully blocks the installation of apps from unauthorized sources, even if those sources were granted the REQUEST_INSTALL_PACKAGES permission" — only Play Store and Galaxy Store count as authorized, the per-app "install unknown apps" toggle is grayed out globally, and **there is no per-app exception**, only the global toggle (biometric-gated) under Settings → Security and privacy → Auto Blocker. Standard mode also blocks ADB commands **over USB cable**. "Maximum restrictions" adds scanning and device-admin blocks but no additional install barrier. One UI 8.5 is testing a **30-minute pause** that auto-re-enables — the right tool for occasional sideloading, but for a continuous auto-update pipeline Auto Blocker simply has to stay off. Whether a Shizuku-mediated `pm install` over *wireless* debugging slips past Auto Blocker is **unverified** — no authoritative source either way — so plan on Auto Blocker being off regardless.
- **Android 15/16 hardening** does not block this workflow but is worth knowing: Android 15 expands restricted settings for sideloaded apps (accessibility, notification access, etc. need a manual per-app unlock); Android 16's Advanced Protection Mode, **off by default**, disables sideloading entirely when enabled — leave it off on this device.
- **Google's developer verification** begins 2026-09-30 in four pilot countries and reaches certified devices globally in 2027. Self-built apps installed via ADB/Shizuku-class paths remain exempt, and "Limited Distribution Accounts" (up to 20 devices, no government ID) are planned — a horizon item, not a current blocker.
- **Shizuku** grants shell-level privilege (silent `pm install`) to authorized apps and activates on-device via wireless-debugging pairing, no PC needed. Its documented limitation: **the process dies on every reboot and must be manually restarted** — there is no non-root autostart. Obtainium's Shizuku-backed background updates are **async and best-effort: a failed background update produces no notification**, so a stale version can linger silently; the mitigation is that Obtainium still shows pending updates when opened.
- **Signing and versioning**: an update installs only if signed with the **same key** and carrying a **higher versionCode**; a key mismatch fails with `INSTALL_FAILED_UPDATE_INCOMPATIBLE` and forces an uninstall (losing app data). The camera app signs releases with the **machine-local debug keystore**, so every release must be built on the Mac Mini — a build from any other machine cannot update over the phone's installed copy. That constraint is compatible with this workflow (the Mac Mini is the only builder), but generating a dedicated release keystore and backing it up would remove the single-machine fragility; the repo already bumps versionCode/versionName with each `dist/` APK, which is exactly what Obtainium-driven updates require.

## Automation sketch

### One-time, Mac

```bash
# distribution repo: assets only, source stays local
gh repo create camera-releases --private

# classic PAT for the phone: Settings → Developer settings → Tokens (classic),
# scope: repo. (Fine-grained tokens hit Obtainium bug #2694 — don't.)
```

### One-time, phone (Galaxy S26 Ultra)

1. Settings → Security and privacy → **Auto Blocker → off** (biometric confirm). Verify Advanced Protection Mode is off.
2. Install **Obtainium** (download the APK from its GitHub releases in the browser; one-time "install unknown apps" grant to the browser).
3. In Obtainium: Settings → source-specific settings → GitHub → paste the classic PAT. Add app → `https://github.com/<you>/camera-releases`.
4. First install of the camera app through Obtainium — one tap (a first install can never be silent; silent applies to updates only).
5. For zero-tap updates: install **Shizuku**, pair via Developer options → Wireless debugging, start it; in Obtainium enable Shizuku as installer and background updates (globally and for the app). Grant Obtainium the Shizuku permission on first use.
6. After every reboot: open Shizuku, tap start. (Skip Shizuku entirely if this ritual costs more than the 2–3 taps per update it saves.)

### Per release, Mac (`camera/tools/release.sh`)

```bash
#!/usr/bin/env zsh
set -euo pipefail
export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
cd ~/workspace/camera

# version comes from the repo's single source of truth
version=$(rg -o 'versionName = "([^"]+)"' -r '$1' app/build.gradle.kts)
apk="dist/camera-${version}.apk"

# guard: refuse to re-release — bump versionCode/versionName first
if gh release view "v${version}" --repo "$(gh api user -q .login)/camera-releases" >/dev/null 2>&1; then
  echo "v${version} already released; bump versionCode/versionName first" >&2
  exit 1
fi

./gradlew assembleRelease
cp app/build/outputs/apk/release/app-release.apk "$apk"
git add "$apk" app/build.gradle.kts
git commit -m "release: v${version}"

gh release create "v${version}" "$apk" \
  --repo "$(gh api user -q .login)/camera-releases" \
  --title "camera v${version}" --notes "$(git log -1 --format=%s)"
```

Steady state: bump versionCode/versionName, run `tools/release.sh`, and the phone updates itself within Obtainium's polling interval (default 6 h; pull-to-refresh in Obtainium forces an immediate check). Optionally append the Telegram curl one-liner to the script as a push notification that the release shipped — Telegram as the bell, GitHub + Obtainium as the pipe.

## Sources

- Telegram Bot API limits and FAQ: [core.telegram.org/bots/faq](https://core.telegram.org/bots/faq); self-hosted server limits: [github.com/tdlib/telegram-bot-api](https://github.com/tdlib/telegram-bot-api)
- Gmail blocked file types (including archives and password-protected archives): [support.google.com/mail/answer/6590](https://support.google.com/mail/answer/6590)
- rclone Drive backend: [rclone.org/drive](https://rclone.org/drive/); Drive notification triggers: [support.google.com/drive/answer/6318501](https://support.google.com/drive/answer/6318501)
- PackageInstaller silent-update conditions: [developer.android.com — PackageInstaller.SessionParams](https://developer.android.com/reference/android/content/pm/PackageInstaller.SessionParams); update ownership: [source.android.com — app ownership](https://source.android.com/docs/setup/create/app-ownership)
- Samsung Auto Blocker: [Samsung support ANS10003636](https://www.samsung.com/us/support/answer/ANS10003636/); [Knox security whitepaper](https://docs.samsungknox.com/admin/fundamentals/whitepaper/samsung-knox-mobile-security/system-security/samsung-auto-blocker/); default-on since One UI 6.1.1: [Android Authority, 2024-07](https://www.androidauthority.com/enable-sideloading-one-ui-6-1-1-3463446/); One UI 8.5 pause: [Sammy Fans, 2025-09](https://www.sammyfans.com/2025/09/22/samsung-might-let-you-pause-auto-blocker-soon/)
- Shizuku setup and reboot limitation: [shizuku.rikka.app/guide/setup](https://shizuku.rikka.app/guide/setup/)
- Obtainium sources, private-repo tokens, silent-install criteria: [wiki.obtainium.imranr.dev](https://wiki.obtainium.imranr.dev/); private-asset fix: [issue #857](https://github.com/ImranR98/Obtainium/issues/857); fine-grained-token bug: [issue #2694](https://github.com/ImranR98/Obtainium/issues/2694)
- GitHub release limits: [docs.github.com — About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)
- Tailscale: [macOS variants](https://tailscale.com/docs/concepts/macos-variants), [serve](https://tailscale.com/docs/features/tailscale-serve), [pricing](https://tailscale.com/pricing)
- F-Droid unattended updates (client 1.19+): [f-droid.org TWIF 2024-02-01](https://f-droid.org/2024/02/01/twif.html); fdroidserver install: [f-droid.org docs](https://f-droid.org/docs/Installing_the_Server_and_Repo_Tools/)
- Firebase App Distribution CLI: [firebase.google.com — distribute-cli](https://firebase.google.com/docs/app-distribution/android/distribute-cli)
- Cloudflare quick tunnels: [developers.cloudflare.com](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/); ngrok free-plan limits: [ngrok.com/docs/pricing-limits/free-plan-limits](https://ngrok.com/docs/pricing-limits/free-plan-limits)
- Android developer verification timeline: [android-developers.googleblog.com, 2026-03](https://android-developers.googleblog.com/2026/03/android-developer-verification.html)
