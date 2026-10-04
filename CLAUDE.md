# Kingdom Clash — Engine Notes

Party game for youth-group game nights. One self-contained `index.html` (no build step,
no dependencies), deployed via GitHub Pages by pushing to `main` on
`stevenabi6912-prog/Kingdom-Clash`. Operator runs the game on one window; a presenter
window (`?present` URL param) mirrors everything for the audience/projector.

This file doubles as the handoff reference for building sibling games on this engine.

## Architecture

- **Single file.** All CSS, HTML, and JS in `index.html`. Assets (png/jpg/mp3) sit in
  the repo root. Some filenames contain spaces (`face off.mp3`, `pick again.mp3`) —
  browsers URL-encode them fine, but match names exactly.
- **Dual-window sync.** Operator broadcasts full game state (`broadcastState()`) on
  every change via THREE redundant channels: BroadcastChannel (`kingdom-clash-v1`),
  a localStorage key (`kingdomClash_liveState_v1`) consumed by `storage` events in
  other windows, and a 500ms polling fallback on the presenter. All three exist
  because BroadcastChannel between popup windows is unreliable in some Safari setups.
  Receivers guard with `applyingRemoteState` to prevent rebroadcast loops.
- **Game state:** `board` (array of `{label, revealed}`), `scores`/`shields`
  (always keyed A–D), `activeTeam`, `teamCount` (2–4), `started`, `winnerShown`.
  Settings persist in localStorage under `kingdomClashSettings_v12_*`; board-related
  settings are staged and only take effect via the "Apply & Start Fresh Board" button.
- **Outcomes:** tiles carry string labels (`"500"`, `"BANKRUPT"`, `"x2"`,
  `"PICK_AGAIN"`, `"STEAL"`, `"FACE_OFF"`, `"SHIELD"`). `applyOutcome(label, tileIdx)`
  is async and may await host input via picker overlays that return Promises
  (`pickStealTarget` → team key or null for skip; `pickFaceOffWinner` → team key,
  includes a 20s countdown with heartbeat + synthesized tick). A new tile type =
  deck entry in `buildDeck` + `formatReveal` branch + `applyOutcome` branch +
  settings count dropdown.
- **New board = fresh game.** `newBoard()` zeroes scores, shields, active team, undo.
  Nothing carries over (deliberate host preference).

## Hard-won gotchas (do not relearn these)

1. **`.hidden { display:none !important }`** — without `!important`, any later rule
   with equal specificity (e.g. `.presenter-hud { display:flex }`) silently wins the
   cascade and "hidden" elements render. This bug cost a full debugging session.
2. **No magic-number layout.** Body is a flex column; `.stage` is `flex:1 1 auto;
   min-height:0`; grid fills what's left. Earlier `height: 100svh - 150px` style
   calcs overflowed the viewport whenever the panel grew. Tile squareness comes from
   container queries (`container-type:size` on `.grid-wrap`, `cqw/cqh` math on `.grid`).
3. **rAF doesn't fire in hidden tabs.** Any rAF-driven animation (score count-up)
   must snap to the final value when `document.hidden` and carry a failsafe timeout —
   otherwise a backgrounded operator shows stale scores forever. Decorative WAAPI
   flyers skip entirely when hidden and carry `setTimeout` cleanup for throttled tabs.
4. **Mid-flip tiles have ~0 width.** The flip animation is `rotateY`; at the moment
   outcomes run, `getBoundingClientRect().width` is ≈0. Only bail on fully-empty rects;
   the rect's center is still correct.
5. **Percentage `max-height` fails in auto-sized grid tracks** (circular dependency,
   silently ignored). Use viewport units (`svh`) for popup image caps.
6. **`renderGrid` memoizes a board signature** and skips the `innerHTML` rebuild when
   the board hasn't changed. Broadcasts arrive in rapid bursts; without the memo,
   every score/popup broadcast wiped the grid and cut flip/ripple animations off
   after ~30ms (audience never saw them).
7. **Audio needs a user gesture.** `ensureAudioAwake()` resolves only after real
   interaction — programmatic `.click()` on Start hangs. In automated tests, stub
   `window.ensureAudioAwake = async () => {}` and set state directly. All SFX play
   on the OPERATOR device only (known limitation, host accepted it).
8. **GitHub Pages caches for 10 minutes** (`max-age=600`) plus browser cache — always
   hard-refresh both windows after a deploy before concluding something is broken.
9. **Randomized looping animations need NEGATIVE `animation-delay`** (start mid-cycle)
   plus per-element random durations, or they read as a marching pattern / dead board.
10. **Winner timing:** `checkForWinner` waits `popupAutoCloseMs + 700` so the final
    tile's outcome popup plays out before the winner popup replaces it.

## Conventions

- Popups: `showPopup(kind, title, sub, color, imageKey, autoClose)` — broadcasts via
  `currentPopup`; presenter re-renders only when `kind|title` changes, updates sub
  text / countdown in place otherwise. `popupAutoCloseMs = 2500`.
- Fluid type everywhere: `clamp(min, vw-based, max)`. Presenter HUD steps font sizes
  down via `.teams-3` / `.teams-4` classes.
- Team boxes stack name (small label) over score (big gold number) — side-by-side
  layouts caused mid-word name wrapping at 4 teams.
- Testing: serve a copy from a scratch dir with a tiny Node static server
  (worktree dirs are sandbox-blocked for spawned processes), drive the page via
  browser JS eval, presenter as same-origin iframe. Mute audio in tests. Remember
  BroadcastChannel cross-contaminates between iframes of the same origin.
- SFX pipeline: files registered in `sfxFiles`/`relSfx`, preloaded into WebAudio
  buffers + HTMLAudio fallback pool. Countdown tick is synthesized (square-wave
  oscillator), no file. Convert Epidemic `.wav` drops with
  `ffmpeg -i in.wav -codec:a libmp3lame -qscale:a 4 out.mp3`.
