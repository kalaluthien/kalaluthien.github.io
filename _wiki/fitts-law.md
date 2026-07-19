---
share: true
title: "Fitts's Law"
categories:
  - Wiki
tags:
  - fitts-law
  - hci
  - ui-design
  - usability
author: claude
math: true
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-fitts-law-and-modern-ui-building-blocks]]"
aliases:
  - Fitts law
  - Fitts's law
---

# Fitts's Law

The model of **pointing**: the time to move a pointer (mouse cursor, fingertip)
to a target grows with the distance to it and shrinks as the target gets larger,
and only **logarithmically** in the ratio — so target **size buys more than
proximity does**. The single most-used law in UI layout. The sibling laws and
the pattern-level mapping live in [UI Interaction Laws](/wiki/ui-interaction-laws/).

## Formula

Let $D$ = distance to target center (also written $A$, amplitude), $W$ = target
width *along the axis of motion*, $MT$ = movement time, $a$/$b$ = empirical
constants (device + user + setup).

- **Fitts's original 1954 index of difficulty:** $ID = \log_2(2D/W)$ bits.
- **MacKenzie's 1992 "Shannon" reformulation** (modern standard, ISO 9241-9):
  $ID = \log_2(D/W + 1)$ bits — fits data better and never goes negative (the
  original can go negative for close, large targets).
- Movement time is linear in difficulty: $MT = a + b\cdot ID$.
- **$a$** = fixed overhead (reaction + press), target-independent. **$b$** =
  seconds per bit, the device's cost of difficulty.

> ⚠️ Naming caution: the form $\log_2(2D/W)$ is often *called* "Shannon" but is
> actually Fitts's **original** index. The genuine Shannon (MacKenzie 1992) form
> is $\log_2(D/W + 1)$. They agree when $D \gg W$.

### Throughput

$TP = ID_e / MT$ in **bits/s**, where the *effective* width
$W_e = 4.133\,\sigma$ is re-measured from the actual scatter of users' endpoints
($\sigma$ = std. dev. of hits). This "adjust for the errors they really made"
step lets throughput fairly compare devices. Mouse ≈ 3.7–4.9 bits/s; direct
finger touch often higher (~6–11 bits/s).

### Refinements

- **2-D targets:** use the dimension along the line of approach, or the smaller
  of width/height (`min` model, MacKenzie & Buxton 1992). Wide-thin buttons are
  easy horizontally, fiddly vertically.
- **Small-target error floor:** below some size, aiming noise dominates and
  error rates climb — the empirical basis for minimum-size guidelines.

## Corollaries designers use

- **Bigger + closer = faster;** prefer size over proximity (the log).
- **Screen edges and corners are effectively infinite targets:** the pointer
  *stops* at the edge, so $W\to\infty$ along that axis and $ID\to 0$,
  $MT\to a$. Corners are infinite on *both* axes — the **prime pixels** /
  **magic corners**, the four fastest points on screen. Powers the **macOS menu
  bar** (top edge) and **Windows Start** (corner); Tognazzini reports the
  edge-pinned Mac menu bar is ~5× faster than a Windows in-window menu bar.
- **Magic pixel / one-pixel tax:** the benefit needs the target to *truly* reach
  the edge; a single non-clickable pixel of gap costs ~20–30% (Tognazzini).
- **Context menu → $D\approx 0$:** opens at the cursor, so travel vanishes and
  $MT\to a$. Fastest non-keyboard acquisition; cost is discoverability.
- **Fittsize:** reduce the *number* of acquisitions, not only size/distance.

## Building blocks as Fitts artifacts

- **Minimum tap targets** (the small-target floor as policy, ~9 mm physical):
  Apple **44 pt**, Material **48 dp**, WCAG 2.5.8 **24 CSS px** (AA, with a
  spacing exception), WCAG 2.5.5 **44 CSS px** (AAA). Hit target ≠ visual size —
  pad a small icon out to the minimum.
- **FAB:** large (56 dp), fixed position (muscle memory), near the thumb zone.
- **Segmented control:** contiguous targets, no gaps → tiny $D$, no dead space.
- **Dock magnification:** dynamic $W$; debated, since a growing icon can shift
  under the pointer and violate the fixed-target assumption.
- **Toolbar vs overflow ("⋯"/kebab):** visible = one acquisition; hidden = an
  extra one, same tax as the hamburger. Surface frequent, bury rare.

## Related

- [UI Interaction Laws](/wiki/ui-interaction-laws/) — the steering law, the thumb zone, the
  command-palette bypass, and which law governs which pattern.
