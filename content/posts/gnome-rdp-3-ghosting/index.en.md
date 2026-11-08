---
title: "A GNOME session over RDP, made usable (3): ghosting, when the compositor's damage is wrong"
date: 2026-11-08
draft: false
summary: "An old window title stayed on screen; the clock in the top bar was minutes late. Not the encoder, not the network: Mutter's damage regions sometimes leave changed areas out. A 64x64 tile check repairs what is missed, for about 1.3 ms a frame."
tags: ["rdp", "freerdp", "gnome", "mutter", "pipewire", "screencast", "gnome-rdp-series"]
---

*Part 3 of a series. [Part 1](../gnome-rdp-1-overview/) has the setup.*

The second complaint: **ghosting**. A window was closed and its title bar stayed. The clock
in the top bar showed an old time, late by different amounts. A video played without a
problem, and so did a terminal.

## Ruling things out

A video encoder can leave ghosts while it recovers quality after a big change, so we
measured that first: frames that switch from one window to another, encoded with AV1 and
AV2, compared with the settled picture (PSNR). AV2 was within 1 dB after at most three
frames (184–244 ms); AV1 at once. Nothing that lasts for minutes. The encoders have no
look-ahead either (AV2 `g_lag_in_frames = 0`, AV1 in realtime mode).

What was left was the input to the encoder: the regions of the screen that Mutter said had
changed.

## Damage, and how to check it

Mutter's screencast attaches *damage* to every buffer (`SPA_META_VideoDamage`): a list of
rectangles that changed since the last buffer. A consumer that keeps its own copy of the
screen applies only those rectangles, which is the whole point: copying and encoding a
2560x1440 screen 30 times a second is expensive when only a clock changed.

We added a detector behind the damage copy: after applying a buffer's damage, compare the
whole buffer with our copy. Any difference left is a change that the damage did not report.

In a container, with a window that changes its title every 0.3 s and a pop-up that opens and
closes every 2 s:

- 1 of 208 buffers had 41 140 pixels changed outside its only damage region: a 526x326 box,
  exactly the pop-up with its shadow.
- In another run, 6 buffers in a row missed the same 1017x617 box. No buffer in between
  reported it.

In a real session the boxes were the clock and status icons in the top bar, top-bar icons and
strips near the bottom of the screen: about 1300 buffers in 25 minutes. That is the ghosting:
the area changed, nobody said so, and our copy kept the old pixels until something else
repainted there.

## The safety net

Trusting the damage is wrong some of the time, and copying the whole screen every frame is
too slow, so the fix is in between: after the damage copy, compare the buffer with our copy
in 64x64 tiles and copy the tiles that still differ.

- **Cost** at 2560x1440: 1.30 ms a frame when nothing is left over, 1.42 ms with three tiles
  to repair; about 4 % of one core at 30 frames a second. It is limited by memory bandwidth,
  so a per-pixel compare is not slower (1.22 ms), but tiles give us boxes to log.
- **Effect** in the same test: before, 6 of 325 frames had a stale area, the same box six
  checks in a row; after, 0 of 298, with 44 tiles repaired along the way.
- It is on by default and can be switched off with
  `FREERDP_SHADOW_MUTTER_DAMAGE_CHECK=0`. The server logs how many tiles Mutter left out,
  every few seconds, so a fixed Mutter will show up as zeros.

The first words after installing it were the ones we wanted: *"it works now, the clock
updates."* Our xrdp Wayland backend, which reads the same Mutter streams, has the same kind
of check.

## Telling GNOME

A workaround in every consumer is not the real fix: Mutter should report every change. The
report for GNOME is written: the steps above, the numbers, and where in
`meta-stream-source.c` we looked. Part 6 is about when we ship a workaround like this one and
when we only report.
