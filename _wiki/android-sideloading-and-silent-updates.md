---
share: true
title: Android sideloading and silent updates
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-16
updated: 2026-07-16
sources:
  - "[[2026-07-16-automated-apk-delivery-from-mac-to-phone]]"
aliases:
  - samsung-auto-blocker
  - shizuku
---

# Android sideloading and silent updates

What Android (and Samsung's One UI on top of it) permits when installing APKs
outside an app store, as of Android 16 / One UI 8 era. Distribution channels
and tooling live in [Personal APK distribution](/wiki/personal-apk-distribution/).

## The permission model

- **"Install unknown apps"** (`REQUEST_INSTALL_PACKAGES`) is a per-app grant:
  each app that hands APKs to the installer (browser, Telegram, Obtainium,
  file manager) needs its own one-time grant.
- A granted app drives the **PackageInstaller session API**: create session,
  stream APK bytes, `commit()` — which surfaces the system confirm dialog.
  That dialog is the unavoidable floor for any **first** install without
  root/MDM.

## Silent updates — the exact conditions

`setRequireUserAction(USER_ACTION_NOT_REQUIRED)` skips the confirm dialog
only when **all** hold
([SessionParams docs](https://developer.android.com/reference/android/content/pm/PackageInstaller.SessionParams)):

1. The installer declares `UPDATE_PACKAGES_WITHOUT_USER_ACTION`
   (normal-level, auto-granted).
2. The installer is the **update owner or installer-of-record** of the app
   being updated — i.e. this is an update, never a first install. Android 14's
   `ENFORCE_UPDATE_OWNERSHIP` lets the first installer claim exclusive
   silent-update rights; any other installer then triggers a dialog.
3. The app being updated targets a recent SDK: **API 34+ on Android 16
   ("Baklava")**; the docs state this bar keeps advancing with each release.

Consequence: silent updates belong to whoever installed the app first. Pick
the updater app before the first install.

## Fully silent first installs

- **Device owner / MDM**: truly silent, but provisioning requires a
  factory-fresh device with no accounts — not viable on a personal daily
  driver.
- **Root**: trivial, out of scope by preference.
- **Shizuku**: exposes ADB shell privilege (silent `pm install`) to
  authorized apps; activates on-device via wireless-debugging pairing, no PC.
  Hard limitation: **the process dies on every reboot and must be manually
  restarted** — no non-root autostart exists
  ([setup guide](https://shizuku.rikka.app/guide/setup/)).

## Samsung Auto Blocker (One UI 6+)

- **On by default since One UI 6.1.1 (July 2024)**, carried through One UI
  7/8 — current Galaxy flagships ship with it enabled. The setting survives
  Smart Switch restores.
- While on, it "fully blocks the installation of apps from unauthorized
  sources, even if those sources were granted the REQUEST_INSTALL_PACKAGES
  permission" — only Play Store and Galaxy Store are authorized; the
  install-unknown-apps toggle is grayed out globally
  ([Samsung ANS10003636](https://www.samsung.com/us/support/answer/ANS10003636/),
  [Knox whitepaper](https://docs.samsungknox.com/admin/fundamentals/whitepaper/samsung-knox-mobile-security/system-security/samsung-auto-blocker/)).
- **No per-app exception exists** — the only control is the global toggle
  (biometric-gated). Standard mode also blocks ADB commands **over USB
  cable**. "Maximum restrictions" adds scanning and device-admin blocks, not
  a further install barrier.
- One UI 8.5 is testing a **30-minute pause** (auto re-enables) — good for
  occasional sideloading, useless for a continuous update pipeline.
- Open question (no authoritative source either way): whether a
  Shizuku-mediated `pm install` over *wireless* debugging bypasses Auto
  Blocker. Assume it does not; keep Auto Blocker off for any sideload
  pipeline.

## Adjacent hardening to know about

- **Android 15**: restricted settings widen — sideloaded apps need a manual
  per-app unlock before granting accessibility, notification access, device
  admin, overlay, or usage access.
- **Android 16 Advanced Protection Mode** (off by default): when on, disables
  install-unknown-apps entirely.
- **Google developer verification**: pilots 2026-09-30 (BR/ID/SG/TH),
  certified devices globally in 2027. ADB-class installs stay exempt;
  "Limited Distribution Accounts" (20 devices, no government ID) planned for
  hobbyists.

## Signing and versioning invariants

- An update needs the **same signing key** and a **higher `versionCode`**.
  Mismatch → `INSTALL_FAILED_UPDATE_INCOMPATIBLE`; only remedy is uninstall
  (which wipes app data).
- The Android **debug keystore is machine-local** (`~/.android/debug.keystore`):
  a debug-signed app can only be updated by builds from the same machine. For
  multi-machine or long-lived apps, generate a dedicated release keystore and
  back it up.
