# ADR 0002: a request for help names a real shortage, and takes itself down

- Status: accepted
- Context: a pod riff at crit 10 asked for "a quiet way to ask each other for help without chat". Plenty has no text chat on purpose ([`DECISIONS.md`](../DECISIONS.md)), so strangers in one room coordinate through emotes, named places and the activity log. None of those can say "Ada's factory across the island is out of ore". The only person who can read that is someone who walks over and inspects it.

## The question

How does a player get a stranger to fix a stalled machine without typing anything, and without the request turning into a second, unmoderated channel?

## Decision

A **request** (`ask` in the protocol) is a player pointing at a machine and saying "this is stuck". It has no words of its own. Its only text is the machine's own status, which the economy already computes and the HUD and the "!" marker already show.

- **What can be asked about.** A finished building whose status is one of the economy's idle states (`idle`, `unserved` or `throttled`, which `isIdle` already defines). The asker has to be within load range (`F` range), so a request comes from someone who stood there and saw it. `applyCommand` checks all of this against the server's state. A browser can't send a shortage, only a building's coordinates, so it can't request a shortage that doesn't exist.
- **What it says.** The building, its owner, who asked, and the status string as the economy worded it ("idle: no metal ore (needs 3, has 0)"). The roster and the map label read the building's *current* status, so the counts stay true while the request is up ("has 2" after a partial load).
- **Who sees it.** Everyone in the room. A small flag in the asker's colour, and one label above the building, are drawn after every object so trees don't hide them. There's one line under the asker's name in the roster, and one line in the activity log. No sound, no popup, and no motion of its own.
- **One at a time, per player.** The simpler rule: a player has at most one request up. Asking somewhere else moves it, and `R` beside the same machine (or the button in the inspect panel) takes it down. A machine also carries at most one request, so a second asker is told who already asked.
- **How it clears.** Every tick the server checks each request against its building and takes it down when:
  - the building isn't idle any more (*resolved*), which the log says;
  - the building is stuck on a *different* shortage (*changed*). The log says the asked-for thing arrived and names the new gap, and anyone can ask again for that. Counts changing inside the same shortage don't count as different;
  - the building is gone;
  - two minutes of play have passed (`ASK_TICKS`, 1200 ticks; *expired*). Ticks, not wall time, so a paused empty room doesn't expire anything while nobody's there ([ADR 0001](0001-empty-rooms-pause.md));
  - the asker takes it down, or the world ends.

  No client timer decides any of these. The countdown in the inspect panel is computed from the server's `until`.
- **It touches nothing.** A request is stored in `RoomState.asks` alongside claims. It moves no stock, no goods, no buffers and no seed bank. The spec compares the full ledger, stockpile, goods, seed bank and every building's buffers before and after raising, clearing and expiring a request.
- **Persistence and reconnect.** Requests are part of the saved room and the snapshot, so a refresh or a rejoin sees them. They travel in the `meta` part of a delta like claims. `PROTOCOL_VERSION` and `SCHEMA_VERSION` are now 3; a schema-2 room loads with none up.
- **The action log.** `action=ask` names the asker, the building, its owner and the status. `action=ask_cleared` records why it came down (`resolved`, `changed`, `expired`, `withdrawn`, `moved`, `gone`).

## Why

The game's argument is that a room consumes its world together. A stalled factory is exactly where a group stops being together: the owner is busy elsewhere, and a stranger walking past has no way to know it matters. Chat would fix that and bring everything chat brings. A request carries only what the simulation already knows, so it can't say anything false or anything unkind. Because it comes down by itself, it can't go stale or pile up into a notification feed.

## Alternatives weighed

1. **Free-text requests, or picking from canned phrases.** This is chat by another name, and canned phrases still invent a second vocabulary for shortages the game already describes.
2. **Show every idle machine to everyone automatically.** The "!" marker and the economy panel's bottleneck line already do this, quietly. A request adds the one thing they can't: a person saying *this one matters to me*, which is what turns a passer-by into a helper.
3. **Clear on a client timer, or only by hand.** Requests would stay up after the fix and teach players to ignore them.
4. **One request per building, unlimited per player.** One player could flag the whole island. Per player keeps a request a deliberate act.

## Consequences and costs

- It's easy to miss a request on a big world when the building is off screen. The roster line is the room-wide view. There's deliberately no arrow, sound or feed.
- The two-minute timeout is a judgement, not a measurement: long enough to walk across Earth and load a factory, short enough that an abandoned request doesn't linger.
- A request names a status, not a remedy. "No power: not cabled to a generator" asks for a cable, which the helper still has to work out. The status strings were already written to say what fixes them, and the roster drops the how-to tail only to stay short.
