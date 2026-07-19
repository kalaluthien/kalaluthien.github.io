---
share: true
title: "Running an iPad Game Off-Screen on a Mac: Why It Needs a Visible Virtual Display"
categories:
  - Report
  - Mac
tags:
  - macos
  - betterdisplay
  - ios-app-lifecycle
  - virtual-display
author: claude
---

> 마비노기 모바일 for Mac does not exist and the iPad app is used on the Mac
> Mini. Background execution without focus is not supported for this app; it
> follows the iPhone rules. Write this up, plus the BetterDisplay settings.

## Short answer

마비노기 모바일 (Mabinogi Mobile) has no native Mac build. On an Apple Silicon
Mac you install the iPad App Store build, and it runs under the iPhone/iPad
lifecycle rather than the desktop one. Under that lifecycle an app runs code
only while its window is **visible on a connected display**. Minimizing or
hiding the window moves the app to the background, where the system suspends it
and the game's timers and automation stall.

The requirement is visibility, **not** keyboard focus. So you cannot run the
game "in the background" the way a desktop app runs behind other windows, but
you also do not have to keep looking at it. Put its window on a second display
you never look at — a BetterDisplay virtual monitor placed off to the side. The
window stays visible (so the app keeps running) while it sits out of sight and
out of focus. That off-screen parking spot is the whole point of the tooling in
`mabinogi-mobile-automation`.

## Why there is no real background option

An iPhone or iPad app installed on Apple Silicon keeps the iOS scene lifecycle:
a scene is *foreground-active*, *foreground-inactive*, *background*, or
*suspended*. A plain app runs code only in the foreground states. When it goes
to the background the system gives it a few seconds to save, then suspends it —
the process stays in memory but executes nothing. Only apps that declare a
background mode (location, audio, VoIP, and a few others) keep running, and a
game client does not qualify.

On the Mac this maps onto ordinary window actions:

- **Minimize to Dock** or **Hide the app** → the scene enters the background →
  suspended. The game freezes.
- **Window stays mapped on a connected display** → the scene stays
  foreground-active, even if another app holds keyboard focus. The game keeps
  running.

This is why "focus" is the wrong axis. The app does not need to be the frontmost
application; it needs its window to remain on a real, connected display surface.
A second monitor satisfies that — and a virtual monitor is a second monitor the
hardware does not have to exist for.

## The workaround: visible, but out of the way

Moving a window to another display is not the same as minimizing it. The window
stays mapped, the scene stays active, and code keeps running. So the strategy
is:

1. Create a virtual display that is not mirrored onto the main screen.
2. Place it off to the side in the display arrangement — a region no physical
   monitor covers.
3. Move the game window there when idle, and back to the main monitor to play.

The window is always "on screen" from the system's point of view, so the game
never suspends; it is simply on a screen you cannot see. `background` parks it
there without taking focus; `foreground` brings it back and focuses it.

## BetterDisplay: the parking spot

[BetterDisplay](https://github.com/waydabber/BetterDisplay) creates the virtual
display. Install it either way:

- **Homebrew:** `brew install --cask betterdisplay`
- **Web:** download the app from the
  [releases page](https://github.com/waydabber/BetterDisplay/releases) (or
  betterdisplay.pro) and drag it to `/Applications`.

Add one virtual screen and set:

- **Connect this virtual screen:** on. It must be connected — a disconnected
  virtual screen is not a display surface, so a window "on" it would count as
  hidden and the app would suspend.
- **Virtual screen name:** `Virtual 16:9` (any name; it is only an identifier).
- **Aspect ratio:** `16 x 9`, so the parked window keeps the game's proportions.
- **Enable resolutions over 8K**, **Use custom resolution list**, **Rotated
  orientation:** off. Defaults are enough, and virtual screens above 8K can be
  unstable.

Then, in **System Settings → Displays → Arrange…**, drag `Virtual 16:9` to the
right of the main display and do **not** mirror it. Exact coordinates do not
matter; the mover reads the live display layout through CoreGraphics, so it
adapts when you rearrange monitors or change scaling. What matters is that the
parking display is a separate, non-mirrored, connected screen.

## One more requirement: Accessibility permission

Moving another app's window is done by macOS System Events, which needs
Accessibility permission for the terminal (System Settings → Privacy & Security
→ Accessibility). Without it the move fails with an "assistive access" error
(`-1719`). This is a system gate; it cannot be scripted around. Do not try to
move the window with BetterDisplay's own CLI — that tool manages displays, not
windows.

## Caveats

- The suspend-on-background behavior above is the observed behavior for this
  client and the general iOS-on-Mac lifecycle; a specific app that declares a
  background mode could behave differently.
- The parking display must stay **connected**. A virtual screen you disconnect
  to "save resources" turns the parked window into a hidden one, which
  re-triggers suspension.
