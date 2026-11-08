---
title: "A GNOME session over RDP, made usable (4): flow control, or why a slow link lagged 13 seconds"
date: 2026-11-08
draft: false
summary: "On a slow link the delay grew without end: 2.4 s, then 13 s, within half a minute. The encoder kept its frame rate whatever the link could carry. At most 8 frames in flight, acknowledged by the client, keeps it at a quarter of a second."
tags: ["rdp", "freerdp", "gnome", "rdpgfx", "network", "gnome-rdp-series"]
---

*Part 4 of a series. [Part 1](../gnome-rdp-1-overview/) has the setup.*

While testing the sound fix ([part 2](../gnome-rdp-2-sound/)) we slowed the container's
network down with `tc netem` to see what a weak link does. The answer was alarming: at
1 Mbit/s with AV1, the sound delay was 14 seconds at the 95th percentile, and 99 % of the
sound packets were more than 100 ms late.

## Nothing told the encoder to slow down

The graphics pipeline of RDP (RDPGFX) has acknowledgements built in: the client sends a
*frame acknowledge* for every frame it has decoded. A server can use them to know how many
frames are still on the way, and stop producing new ones when too many are.

FreeRDP's shadow server only used them in its X11 backend. The Mutter backend encoded every
frame Mutter delivered, at full rate, and wrote it to the socket. On a fast link that is
fine. On a slow link the frames pile up in the TCP send buffer, and everything behind them,
including sound and the pointer, waits in the same queue. The queue never shrinks, because
new frames keep arriving faster than the link drains them.

## At most 8 frames in flight

Our xrdp setup had the answer already: `XRDP_GFX_FRAMES_IN_FLIGHT=8`. So the shadow server
now counts frames sent but not yet acknowledged, per client, and skips encoding while
8 are in flight. Skipped frames are not lost: the damage keeps adding up, and the next frame
sent covers all of it.

Two details mattered:

- **A client that never acknowledges** (some clients don't) would stop the picture forever.
  For such a client the server sends a frame every 2 seconds anyway. Our first version also
  did that for a client that had acknowledged before and then paused, which let it reach 12
  frames in flight; the final version applies the 2-second rule only to a client that never
  acknowledged.
- **The limit is a setting:** `FREERDP_SHADOW_GFX_FRAMES_IN_FLIGHT` (8 by default, 0 turns
  it off).

## Before and after

AV1 at 1 Mbit/s, the median sound delay in the first and the last 5 seconds of a 30-second
run, with the number of sound packets that arrived:

| build | delay at the start | delay at the end | waves |
|---|---|---|---|
| before | 2 419 ms | 13 270 ms | 553 |
| 8 in flight | 298 ms | 250 ms | 835 |

The delay no longer grows. On a fast link nothing changed (839–840 waves, delay p99 3–6 ms
either way), and at 2 Mbit/s neither build built up a backlog.

## A pause that is too long

We also paused the client for 10 seconds to check that the picture comes back. It did, in
every build except one run under AddressSanitizer, where the connection was dropped after
9.4 s. That is by design: FreeRDP sets a TCP user timeout of 9 seconds, and a client that
reads nothing for that long is treated as gone. A 6-second pause recovers everywhere.
