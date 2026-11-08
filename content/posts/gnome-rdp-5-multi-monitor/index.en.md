---
title: "A GNOME session over RDP, made usable (5): two monitors, not one wide one"
date: 2026-11-08
draft: false
summary: "With two client monitors, GNOME saw one 5500-pixel-wide screen: one top bar across both, maximized windows spanning the gap. Now each client monitor gets a GNOME virtual monitor of its own, at its place, with the client's primary as GNOME's. And a timing bug that only showed when the primary moved between connects."
tags: ["rdp", "freerdp", "gnome", "mutter", "multi-monitor", "gnome-rdp-series"]
---

*Part 5 of a series. [Part 1](../gnome-rdp-1-overview/) has the setup.*

A laptop with an external screen sends two monitors to the server: in our case a
2940x1846 built-in display and a 2560x1440 external one beside it. The server made **one**
virtual monitor of their bounding box, 5500x1846. GNOME saw a single very wide screen: one
top bar across both monitors, and a maximized window that filled both, broken by the gap
between the physical screens.

## One virtual monitor per client monitor

Mutter can create several virtual monitors, each with its own ScreenCast stream. So the
server now does what our xrdp Wayland backend already did:

1. **A stream per client monitor.** Each one has its own size, damage, safety net
   ([part 3](../gnome-rdp-3-ghosting/)) and cursor, and is copied into its place in the
   one desktop the RDP client sees.
2. **GNOME's layout from the client's.** After the streams are up, the server places
   GNOME's monitors with `org.gnome.Mutter.DisplayConfig.ApplyMonitorsConfig` (temporary,
   so nothing is saved), at the client's positions, and closes gaps of up to 4 pixels.
3. **The client's primary is GNOME's primary,** and it is always GNOME's first monitor, so
   the top bar and the windows of the session stay on the screen the user looks at.
4. **Input to the right monitor.** libei gives one absolute-pointer region per virtual
   monitor; each pointer event goes to the region it falls in.
5. **The layout can change.** A new connect, or a Display Control message from the client,
   adds, removes or resizes monitors in the running session.

FreeRDP's server API had no way to pass all of this to a backend: its Display Control
handler kept only the primary monitor's size. A new optional callback, `ClientLayout`, gets
the whole layout; backends without it keep the old behaviour.

## Checked in a container

The test client runs on an X server split into two RandR monitors, with `/multimon`; GNOME
runs headless with no monitor of its own, as under the session manager:

- GNOME gets two monitors at the client's places, `[[0,0,1600,900],[1600,0,1280,800]]`,
  and the client's primary is GNOME's primary;
- a maximized window is 1600x868: one monitor minus the top bar;
- a one-monitor client afterwards gets one monitor again;
- the same under AddressSanitizer, with no report.

One rough edge is left: the first big jump of the pointer onto the second monitor sometimes
stops at the edge between them (3 of 6 runs). The next move always gets through, 12 of 12.
Our events are right, with the right device and region, so the pointer is held back inside
GNOME, and we have not found where.

## The bug that real use found

After a day on two real monitors: *"the MacBook's own screen always becomes the primary,
but my external one is"*. The server's log showed GNOME following the client every time;
it was the client that called the built-in screen primary in some connects. (With the SDL
client, `/monitors:` puts the monitor you list first as primary.)

But the same log showed something worse. When the primary moves from one monitor to the
other between connects, the two virtual monitors **swap sizes**. The server sent GNOME the
new positions as soon as each stream had a format, and a resized stream still had its old
format, so GNOME's monitors were placed with the sizes they were leaving. In that session it
happened to come out right. In a test with the same shape (two monitors of different sizes,
the primary swapped on every connect) the old build put the monitors in the wrong places in
one of three connects, and the fixed build, which waits until every stream has its new size,
in none. One run each: a timing race needs more runs to be sure, and we say so.

The server now also logs the layout as the client sent it, one line per monitor, so the next
question like this one takes a minute to answer.
