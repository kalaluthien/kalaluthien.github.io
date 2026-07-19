---
share: true
title: UI Interaction Laws
categories:
  - Wiki
tags:
  - hci
  - ui-design
  - fitts-law
  - usability
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-fitts-law-and-modern-ui-building-blocks]]"
aliases:
  - steering law
  - Accot-Zhai steering law
  - thumb zone
---

# UI Interaction Laws

The family of quantitative laws that predict how long UI interactions take, and
which pattern each one governs. [Fitts's Law](/wiki/fitts-law/) (pointing at a discrete target) is
the anchor; this page covers its siblings and the pattern-level mapping.

## Which law governs what

| Movement | Law | Formula | Growth |
|---|---|---|---|
| Point at a spot | [Fitts's Law](/wiki/fitts-law/) | $MT=a+b\log_2(D/W+1)$ | log in $D/W$ |
| Drag through a corridor | steering law | $MT=a+b\,(A/W)$ | **linear** in $A/W$ |
| No pointer at all | keyboard / palette | — | bypasses both |

## Steering law (Accot & Zhai, CHI 1997)

"Beyond Fitts' Law: Models for Trajectory-Based HCI." When selection requires
keeping the pointer **inside a bounded path** (tunnel) rather than landing on a
point, time is **linear** in length/width: $MT = a + b\,(A/W)$ — narrow
corridors are punishingly slow, and you pay *continuously* (no aim-once-commit).
Governs **cascading dropdowns, hover mega-menus, sliders, scrollbars**.

### Diagonal-slip problem and the safe triangle

The natural path from a parent menu item to a submenu item is a **diagonal**,
but it crosses sibling rows; the instant the pointer leaves the parent's row the
submenu closes. The **safe triangle** (a.k.a. hover triangle / menu-aim; Ben
Kamens' 2013 breakdown of **Amazon's mega-dropdown**, `jQuery-menu-aim`) watches
motion *direction*: while the cursor stays inside the triangle spanned by its
position and the submenu's two far corners, treat it as "aiming in" and keep the
submenu open — a **prediction** that restores forgiveness. Beats the naive fix
(a timed close *delay*, which feels sluggish) by fixing the geometry.

## Command palettes — the bypass

A **command palette** (`Cmd-K`/`Ctrl-Shift-P`: VS Code, Slack, Linear, Raycast)
has **no target to aim at**, so neither Fitts's nor the steering law applies —
zero travel, zero steering. The ceiling for expert, high-frequency actions:
**the fastest target is no target.** Keyboard shortcuts and type-ahead search
are the same idea in smaller doses. Gestures/swipes similarly trade
point-at-target for a *direction* task (edge-swipes need no precise start), at
the cost of discoverability.

## Thumb zone (ergonomic overlay, NOT a targeting law)

Distinct from Fitts's geometry: the **thumb zone** / **reachability** is about
what a held thumb can reach without strain. Steven Hoober's field study (~1,300
users): ~49% hold phones one-handed, ~75% of interactions are thumb-driven. The
**bottom** of a phone is a green (easy) zone; top corners are red (need a grip
shift). A **bottom tab bar** wins twice — it is a Fitts **edge-target**
(infinite $W$ upward) *and* in the green zone. A **top nav bar** or a
**top-corner hamburger** loses the ergonomic half even though it is an equally
good edge-target; hidden nav also adds an acquisition, which is why phone apps
moved navigation out of the hamburger and into bottom tabs.

## Builder's checklist

Size before proximity · honor 44 pt / 48 dp / 24 px minimums (pad the hit
target) · pin high-value low-precision actions to edges/corners with **no
one-pixel gap** · phone nav in the bottom thumb zone · bring the menu to the
cursor ($D\approx 0$) · keep any steering corridor short & wide + add a safe
triangle (or flatten it) · surface frequent / bury rare · give experts a
keyboard bypass · reduce the *number* of targets.

## Related

- [Fitts's Law](/wiki/fitts-law/) — the pointing law, its formula, corollaries, and building
  blocks in detail.
