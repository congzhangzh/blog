---
title: "A GNOME session over RDP, made usable (1): what broke in daily use"
date: 2026-11-08
draft: false
summary: "We moved our own daily desktop onto a FreeRDP shadow server inside a headless GNOME session. The first two days turned up crackling sound, stale screen areas, lag on slow links, one big monitor instead of two, and an invisible pointer. This series is how each was found and fixed, with the numbers."
tags: ["rdp", "freerdp", "gnome", "mutter", "pipewire", "wayland", "gnome-rdp-series"]
---

A remote desktop is only finished when you can live in it. Demos hide what an eight-hour
working day finds at once: sound that pops while you scroll, a clock in the top bar that is a
minute behind, a session that gets slower and slower on hotel Wi-Fi.

So we did the obvious thing: we moved our own daily desktop onto the stack and wrote down
everything that annoyed us. This series is the result, one post per problem.

## The setup

```
 laptop (SDL3 client)                        Linux server
┌──────────────────────┐   RDP / TLS    ┌───────────────────────────────────────────┐
│ sdl-freerdp          │◀──────────────▶│ session manager (sesman)                  │
│  /multimon, sound    │                │  └─ headless GNOME session per user       │
└──────────────────────┘                │      └─ freerdp-shadow-cli, Mutter backend│
                                        │          ScreenCast + RemoteDesktop (EIS) │
                                        │          PipeWire: picture + sound        │
                                        └───────────────────────────────────────────┘
```

- **The server** is FreeRDP's shadow server with a new backend for GNOME's compositor,
  Mutter. It asks Mutter for a *virtual monitor* (the ScreenCast `RecordVirtual` call) and
  gets the picture, the damage regions and the cursor through PipeWire. Input goes back
  through libei. The backend itself is in review upstream
  ([#14006](https://github.com/FreeRDP/FreeRDP/pull/14006)).
- **The session manager** starts a headless GNOME session for each user who logs in, so
  there is no physical screen at all: GNOME draws only for us.
- **The client** is FreeRDP's SDL3 client, on Linux, macOS and Windows.

## What broke, and where it is fixed

| # | What we saw | Cause | Post |
|---|---|---|---|
| 2 | sound crackles while the screen scrolls | the capture thread waited for every video encode while holding a lock the sound needed | [Sound](../gnome-rdp-2-sound/) |
| 3 | an old window title, a late clock | Mutter's damage regions sometimes leave changed areas out | [Ghosting](../gnome-rdp-3-ghosting/) |
| 4 | lag that grows without end on a slow link | no flow control: frames were sent faster than the link could carry them | [Flow control](../gnome-rdp-4-flow-control/) |
| 5 | one huge monitor instead of two | the server made one virtual monitor of the client's bounding box | [Multi-monitor](../gnome-rdp-5-multi-monitor/) |
| 6 | the pointer is invisible, clicks still work | Mutter sends an empty cursor picture and never the real one again | [Fix it or report it?](../gnome-rdp-6-fix-or-report/) |

## How we worked

Every problem went through the same steps:

1. **Read the live session, without touching it.** Thread CPU times, the server's own logs,
   the memory map. The session is someone's working desktop; restarting it is a decision,
   not a debugging step.
2. **Reproduce it in a container.** A headless GNOME 50 session, PipeWire, a test client and
   a scripted workload (a scrolling window, a 1 kHz tone, a window that opens and closes
   every 2 seconds). Each run is a script with a version number, and a broken script gets a
   new version beside the old one, so every number in these posts can be traced to a run.
3. **Measure before and after, in the same series.** The host's load changes during the
   day, so we only compare runs taken one after the other.
4. **Run it under AddressSanitizer** and the unit tests before anything is installed.
5. **Install side by side**, never over the running version, so going back is one line in a
   systemd drop-in.

The fixes are in our builds of FreeRDP, published by
[rdp-release-kit](https://github.com/congzhangzh/rdp-release-kit). Two of the problems are
in GNOME, not in FreeRDP; the last post is about deciding what to do when the bug is
someone else's.
