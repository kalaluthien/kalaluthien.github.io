---
share: true
title: iOS/iPad apps on Apple Silicon Macs
categories:
  - Wiki
tags:
  - macos
  - ios-app-lifecycle
  - betterdisplay
  - virtual-display
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-running-an-ipad-game-off-screen-on-a-mac]]"
aliases:
  - iPad apps on Mac
  - iOS apps on Mac
  - BetterDisplay off-screen
---

# iOS/iPad apps on Apple Silicon Macs

An iPhone/iPad App Store build installed on an Apple Silicon Mac keeps the **iOS
scene lifecycle**, not the desktop app model. This one fact drives every
behavior below: a scene is *foreground-active*, *foreground-inactive*,
*background*, or *suspended*, and a plain app runs code **only while its window
is visible on a connected display**.

## The visibility rule, not focus

- **Window mapped on a connected display** → scene stays *foreground-active* and
  code keeps running, **even if another app holds keyboard focus**.
- **Minimize to Dock** or **Hide the app** → scene enters *background* → the
  system grants a few seconds to save, then *suspends* it. The process stays in
  memory but executes nothing, so timers, automation, and a game's simulation
  stall.

The axis is **visibility, not focus**. So an iOS-on-Mac app cannot run "in the
background" behind other windows the way a desktop app does — but it also does
not need to be the frontmost application. Only apps declaring a background mode
(location, audio, VoIP, a few others) keep running when hidden; a game client
does not qualify.

## Off-screen parking via a virtual display

Moving a window to another display is **not** the same as minimizing it: the
window stays mapped, the scene stays active, and code keeps running. So the
technique is to keep the window visible on a screen you never look at:

1. Create a virtual display that is **not mirrored** onto the main screen.
2. Place it off to the side in **System Settings → Displays → Arrange…** — a
   region no physical monitor covers.
3. Move the window there when idle, back to the main monitor to use it.

The window is always "on screen" from the system's point of view, so the app
never suspends; it is simply on a screen no one can see.

## BetterDisplay setup

[BetterDisplay](https://github.com/waydabber/BetterDisplay) supplies the virtual
display (`brew install --cask betterdisplay`, or the releases page). Add one
virtual screen and set:

- **Connect this virtual screen: on** — load-bearing. A *disconnected* virtual
  screen is not a display surface, so a window "on" it counts as hidden and the
  app suspends. The parking display must stay connected; disconnecting it to
  "save resources" re-triggers suspension.
- **Aspect ratio 16:9** so the parked window keeps its proportions; name is a
  free-form identifier. Keep resolutions ≤8K (higher can be unstable).

Then drag the virtual screen beside the main display in **Arrange…** and do
**not** mirror it. Exact coordinates do not matter if the window-mover reads the
live layout through CoreGraphics.

## Moving another app's window needs Accessibility

Programmatically moving another app's window goes through macOS **System
Events**, which requires **Accessibility permission** for the terminal (System
Settings → Privacy & Security → Accessibility). Without it the move fails with an
"assistive access" error (`-1719`) — a system gate that cannot be scripted
around. BetterDisplay's own CLI manages *displays*, not *windows*, so it is not a
substitute for the window move.

## Caveat

The suspend-on-background behavior is the observed behavior for a plain game
client plus the general iOS-on-Mac lifecycle; an app that declares a background
mode could differ.
