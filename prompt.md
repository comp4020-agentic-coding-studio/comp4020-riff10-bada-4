# Riff: a quiet way to ask for help

You are working on **Plenty**, a cosy multiplayer harvesting-and-automation
game. Read `README.md`, `CLAUDE.md`, `docs/ARCHITECTURE.md`, `docs/DESIGN.md`,
`docs/DECISIONS.md`, the ADRs in `docs/adr/`, the specs in `spec/`, and the
current implementation (`shared/sim.ts`, `server/`, `client/`) before changing
anything. Every invariant in `CLAUDE.md` still applies without exception,
especially: server-only authority, ledger conservation
(`initial + restored = remaining + extracted + destroyed`), no new source of
`restored` stock besides the Green Lobby, claims never blocking communal
walking/harvesting, fixed-order economy allocation, same-origin writes only,
and `spec/invariants.test.ts` staying green.

Plenty already has no text chat by design (see `docs/DECISIONS.md`). Keep it
that way — do not add chat, DMs, or free text input of any kind.

## Goal

Give players a quiet way to ask each other for help without chat.

Buildings already compute and display their own shortages (e.g. "needs 6 wire
(have 2)"). Let a player standing near such a building raise a small, honest
**request beacon** for it, visible to every connected player, so a stranger
across the island can notice a factory is starved of ore and go fix it.

## Required behaviour

- A player near a building with an unmet input requirement can raise a
  request for it with a clear, discoverable control (a key binding consistent
  with the existing scheme, and/or an on-screen affordance shown at the same
  moment the shortage text is already shown).
- The request names the building, its owner, and what it needs, using the
  same shortage language the game already computes — do not invent a new
  shortage model.
- The request is visible to every connected player, not just the requester:
  as a small marker near the building on the map, and/or a short line in a
  presence-adjacent UI element (not a new dashboard). Keep it quiet and
  small, consistent with the game's current restrained HUD.
- The request must be truthful and self-clearing:
  - it disappears automatically once the shortage it names is actually
    resolved (checked against real server state, not a client timer);
  - it also expires after a reasonable timeout if left unresolved;
  - a player can only raise one active request at a time (per building, or
    per player — pick the simpler rule and state it in the ADR).
- A request carries no new resources and must not touch the ledger, the
  stockpile, or any conservation invariant. It is pure coordination state,
  exactly like an emote or a log line, not a game mechanic with its own
  economy.
- Requests must survive reconnect: a player who refreshes or rejoins sees the
  currently-active requests in the room, not a blank slate.
- Only real, currently-true shortages can be requested — a player cannot
  fabricate a request for a building that isn't actually short of anything,
  and the server must be the one to validate this (`applyCommand`), never the
  client alone.

## Engineering and quality

- Before implementing, read how shortages are currently computed and
  displayed (likely in `shared/sim.ts` and the building inspect/ghost-preview
  code in `client/`), how emotes are modeled and broadcast end-to-end
  (command → `applyCommand`/`tick` → delta → render), and how the roster
  panel surfaces presence, so the new feature reuses these existing
  mechanisms rather than inventing parallel ones.
- Add or amend an ADR under `docs/adr/` recording this decision: what a
  request is, what triggers it, how it clears, and why (tie it back to the
  game's "no chat" stance and to coordinating strangers who cannot talk).
- Bump `PROTOCOL_VERSION` if the wire protocol changes incompatibly, and
  `SCHEMA_VERSION` with a migration in `server/migrate.ts` if saved state
  changes incompatibly.
- Add tests in `spec/sim.test.ts` and/or `spec/server.test.ts` for:
  - a request can only be raised for a building genuinely short of an input,
    and the server rejects a fabricated one;
  - a request becomes visible to a second connected client without a reload;
  - a request clears itself once the shortage is actually resolved;
  - a request times out if left unresolved;
  - a reconnecting client receives the room's currently-active requests;
  - the ledger and stockpile are unaffected by raising, clearing, or
    expiring a request;
  - `spec/invariants.test.ts` remains green.
- Show new tests failing before the fix where practical. Run the complete
  test suite, typecheck, and `pnpm check` (needs a running server:
  `pnpm build && pnpm start`, or `APP_URL`).

## Visual requirements

- Keep the HUD calm and small — no new dashboard, activity feed, or
  notification tray. A request is a quiet marker and/or a short line of text,
  not a popup or modal.
- Respect the existing visual language and colour-per-player scheme; do not
  introduce loud colours, sounds, or game-like alert effects.
- Keyboard-operable, with a visible focus/affordance state, sufficient
  contrast, and no motion that ignores `prefers-reduced-motion`.
- Test with at least two simultaneous browser sessions so the live,
  cross-client behaviour is actually exercised, not just inferred from one
  client.

## Finish

Make small logical commits, keep `main` deployable throughout, push the
completed work, and confirm the deployed app works with two real browser
sessions. As your final commit, delete `prompt.md`, per `CLAUDE.md`.

Report:

- live URL;
- commit SHAs;
- tests run (and which were shown failing before the fix);
- screenshots or a short description of the behaviour from two sessions;
- any limitation found.
