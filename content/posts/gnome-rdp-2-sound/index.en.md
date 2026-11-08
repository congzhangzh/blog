---
title: "A GNOME session over RDP, made usable (2): sound that crackled when the screen moved"
date: 2026-11-08
draft: false
summary: "The sound popped only while something scrolled. The video encoder was not too slow for the sound: the sound was waiting for the encoder, twice, through a lock and through a queue. Two small changes took the sound delay from 159 ms to 7 ms and stopped the lost samples."
tags: ["rdp", "freerdp", "gnome", "pipewire", "sound", "gnome-rdp-series"]
---

*Part 2 of a series. [Part 1](../gnome-rdp-1-overview/) has the setup.*

The first complaint was simple: **the sound crackles and pops, but only sometimes**. A video
in the browser played fine while nothing moved on the screen, and broke up as soon as a page
scrolled.

## Looking without touching

The live server showed one thread busy at 69 % of a core (the client thread, which encodes
the video), the AV2 encoder's worker threads at about 40 % each, and the PipeWire sound
thread almost idle. An idle sound thread in a session with crackling sound is a clue: it
was not doing too much work, it was *waiting*.

## A test that hears it

In a container: a headless GNOME session, a window that scrolls, and a 30-second 1 kHz tone
played through PipeWire. The test client logs every sound packet ("wave") with its arrival
time and the server's capture time stamp. From that we get two numbers that matter to an
ear:

- **waves**: how many sound packets arrived (about 840 in 30 s when nothing is lost);
- **delay**: arrival minus capture time, as p95 / p99 / max.

With the AV2 encoder and a scrolling window, the base build lost samples: 782 waves instead
of about 840, and PipeWire handed us 28 % fewer samples in the first five seconds than it
captured.

## Two places where the sound waited for the video

**1. A lock.** Each time Mutter delivered a frame, the backend took the server's
*clients list* lock and then waited until every client had encoded that frame. The sound
thread needs the same lock to hand each chunk of sound to the clients. So while an AV2
frame was being encoded (65 ms on average, up to 941 ms for a key frame), the sound thread
stood still, and PipeWire dropped what it could not deliver. FreeRDP's own Windows backend
already has a comment warning about exactly this trap.

*Fix:* wait for the encode without holding the clients list lock.

**2. A queue.** Sound packets went to each client as messages for the client's thread, the
same thread that encodes the video, and that thread only looked at its messages after the
frame was done. Even with nothing lost, the sound arrived in bursts.

*Fix:* each client gets a sound thread of its own that writes the sound packets itself. Two
threads writing to one connection must not mix their packets, so one lock now guards every
write to the connection's channel queue.

## Before and after

AV2, a scrolling window, one series of runs:

| build | waves (of ~840) | captured in the first 5 s | delay p95 / p99 / max |
|---|---|---|---|
| before | 782 (−7 %) | 173 056 (−28 %) | 93 / 126 / 159 ms |
| + the lock fix | 840 | 240 640 | 78 / 447 / 811 ms |
| + the sound thread | 841 | 240 640 | **3 / 5 / 7 ms** |
| both, with AV1 | 839 | 240 640 | 2 / 4 / 10 ms |

The lock fix alone stops the loss but makes the delay *worse*: now nothing is lost, so
everything waits in the queue behind the encoder. Only both together fix what you hear.

## The bug we added, and found in real use

A day of real use turned up a defect in the new sound thread: when a connection died, the
sound thread kept writing into it (an "SSL bad length" error every 40 ms) while holding the
channel lock, and the video thread stalled behind it for over a second. The fix: after a
failed write, the sound thread stops and leaves the teardown to the client thread. We could
not make a container client die in the same way (a killed test client is always noticed by
the client thread first), so this one rests on the code and the real session's log.

Both changes passed the unit tests, a 17-cycle connect/kill/disconnect loop under
AddressSanitizer, and then a working day.
