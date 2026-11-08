---
title: "A GNOME session over RDP, made usable (6): fix it here, or report it upstream?"
date: 2026-11-08
draft: false
summary: "Two of our bugs were in GNOME, not in FreeRDP. For stale screen areas we shipped a workaround and wrote a report. For an invisible pointer we wrote the workaround, tested it, and then did not ship it. Why the same kind of bug got two different answers."
tags: ["rdp", "freerdp", "gnome", "mutter", "upstream", "gnome-rdp-series"]
---

*Part 6 of a series. [Part 1](../gnome-rdp-1-overview/) has the setup.*

Our rule is *upstream first*: a fix belongs in the project where the bug is. But a user who
cannot see their pointer today does not care where the bug is, and a GNOME release takes
months to reach a distribution. So for every bug in someone else's code there is a choice:
work around it in our code now, report it and wait, or both.

This series had two such bugs, both in Mutter's screencast, and we answered them
differently.

## The invisible pointer

After the server restarted, the pointer in the client was invisible. Clicks still worked,
and the pointer came back "by itself" some time later.

The server's debug log showed what Mutter sent. The cursor metadata (`SPA_META_Cursor`) in
the first buffer carried Mutter's *empty* cursor picture: format 0, size 0x0. For the next
21 seconds, while the pointer moved and clicked, every buffer carried only positions, never
a picture. A minute later a new connection got a normal 24x24 arrow at once.

Reading Mutter's source explains the pattern (our reading, not proven with a debugger):

- the stream writes the cursor picture only when a flag, `cursor_bitmap_invalid`, is set:
  when the stream starts, and on the cursor tracker's `cursor-changed` signal;
- the picture comes from the cursor *renderer*, which has no cursor while the pointer counts
  as invisible, so the stream writes the empty one and clears the flag;
- when the pointer shows again with the same cursor as before, the tracker sees no change
  (`update_current_cursor` compares the objects) and emits nothing, so the flag stays clear
  and the real picture never comes.

The client keeps the empty picture until the cursor *shape* changes, over a link or a text
field, which is why it seemed to come back by itself. GNOME Remote Desktop handles an empty
picture the same way, as "hidden until the next picture", so it can be caught by the same
thing.

## The workaround we wrote, and did not ship

The workaround is twenty lines: after an empty picture, if the pointer *moves* and Mutter
still sends no picture, send a normal arrow until Mutter sends a real shape. It built clean,
passed the unit tests and the format and spelling checks, and changed nothing in six runs of
a restart test.

Those six runs are the problem: **the container never produced the empty picture.** A
freshly started GNOME sent a real arrow every time. The bug showed up only in a session that
had been running for a long time, in a state we could not name. So the test showed the
workaround is harmless, not that it helps.

And it has a cost we can name: an empty cursor is also what an application gets when it
hides the pointer on purpose, a game or a video player for example. From the metadata alone
we cannot tell "hidden by the application" from "hidden by this bug". The workaround would
show an arrow over a game that asked for none.

So we kept the branch and did not ship it. The report for GNOME is written: the steps, what
we saw, the code we read, and that a fresh session does not show it. The likely fix there is
one line in the right place: invalidate the cursor picture when the pointer becomes visible
again.

## The stale areas, where we did both

The damage bug in [part 3](../gnome-rdp-3-ghosting/) is in the same component, and there we
shipped the workaround. The difference was in the facts, not in the principle:

| | stale areas (part 3) | invisible pointer (part 6) |
|---|---|---|
| how often | every session, many times an hour | rare, after a restart, in some sessions |
| reproduced in a clean setup | yes: 1 of 208 and 6 of 325 frames | no: 0 of 6 runs |
| proof that the workaround works | measured: 6 stale frames before, 0 after | none, only that it does no harm |
| what the workaround can get wrong | nothing: it copies pixels that really changed | it can show an arrow that should be hidden |
| cost | 1.3 ms a frame, measured | almost none |

When a workaround is cheap, can only make the picture more correct, and is measured to
help, we ship it and report the bug. When we cannot show that it helps, and it can be wrong
in a way users would notice, we report and wait, and keep the workaround on a branch in
case the answer is slow.

## What we keep from this

- **Check the clean environment first.** The workaround passed every test, and none of
  those tests reached the case it was for. A test that never hits the bug proves nothing
  about the fix.
- **Name what a workaround can get wrong.** "It costs nothing" is not the same as "it is
  never wrong".
- **A shelved branch is not a waste.** It is ready for the day the bug report gets no
  answer, and it documents the bug better than prose.
- **The bug report is part of the work.** Both GNOME reports wait to be filed, in plain
  words, with numbers and without code, so that the people who know Mutter can fix it in
  the right place.
