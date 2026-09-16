# Notes

What I found while playing, and what I asked the agent to change, in the order it happened.

## 1. Initial build

Asked the agent to fork `esst-prog2/snake` and build the game described in the README: classic
Snake in the browser, starting when `index.html` is opened, no server, no build step.

The agent built a single-file `index.html`: canvas-based grid movement, arrow keys/WASD steering,
growing snake, food, score and a persisted best score, and a game-over screen with a restart
button.

## 2. Playtest: controls felt laggy

Playing it, turning felt delayed — especially near the edges of the board, where I needed to make
two or three quick key presses in a row to avoid a wall, and it felt like presses were getting
dropped.

Asked the agent to reduce the delay between keyboard input and movement, and increase the frame
rate.

The agent found the actual cause: direction changes were stored in a single slot that a second
key press before the next tick would silently overwrite. It replaced that with a small queue (up
to 2 pending turns) so quick double-taps both register, and dropped the movement tick from 110ms
to 85ms. This fixed it — confirmed satisfactory.

## 3. Requested: don't auto-start

Asked the agent to not start the game immediately on load, and instead require pressing space to
start, with on-screen text making that clear.

The agent added a "ready" state with a "Press SPACE to start" prompt drawn on the canvas, and
space also toggled pause during play.

## 4. Playtest: space did nothing when opened from my folder

Opening the file directly from `Documents/DEV/snake` (rather than the agent launching it for me),
pressing space did nothing at all — the game never started.

Told the agent. It guessed this was a keyboard-focus issue (a freshly opened tab not always
getting OS-level focus) and added a click-on-canvas fallback plus a `window.focus()` call.

## 5. Playtest: still not starting — asked for a Start button

That didn't fix it either. Asked the agent to add a distinct, clickable Start button instead of
relying on space/click-to-focus.

The agent added an amber/orange Start button below the canvas that relabels itself
Start → Pause → Resume depending on game state.

## 6. Playtest: button didn't work either — asked for a countdown instead

The button also appeared not to work. Asked the agent to drop the manual start mechanism
entirely and instead auto-start after a visualized 3-second countdown when the page opens.

The agent implemented an animated 3-2-1 countdown drawn on the canvas, auto-starting play when it
reached zero.

## 7. Decided this had gone the wrong direction — reverted

The countdown (and the escalating fixes before it) made things worse rather than better. Asked
the agent to revert all the way back to the version from step 2 (the responsiveness fix), which
had already been confirmed working: game starts immediately, space toggles pause, no start
screen.

## 8. Found the real bug: canvas never drew anything

After reverting, opening `file:///.../snake/index.html` showed the page layout (score labels,
hint text, the canvas's dark background/border) but the snake and food never actually appeared —
nothing was drawn inside the canvas at all, even after a hard refresh.

Told the agent exactly what I was seeing (layout present, canvas empty, hard refresh made no
difference). That ruled out a stale cache and pointed at the script failing before it ever drew
anything. The agent identified the real cause: the script read `localStorage` for the best score
near the top of the code, and some browsers throw instead of returning `null` when `localStorage`
is blocked for `file://` pages (strict privacy mode, private browsing, certain extensions) — which
silently killed the entire script before `reset()`/the draw loop ever ran. This explains why *all*
the earlier fixes (click, focus, button, countdown) never actually worked: the whole script was
crashing on load regardless of the start mechanism.

The agent wrapped both `localStorage` calls in try/catch so a storage block degrades gracefully
(best score just won't persist) instead of taking down the game. Confirmed fixed — the game now
works from that folder.

## 9. Re-requested the Start button / space-to-start

With the real bug fixed, asked the agent to re-add a Start button and/or press-space-to-start.

The agent re-added the "ready" state, the on-canvas "Press SPACE or click Start" prompt, and the
amber Start/Pause/Resume button in one clean pass. Confirmed working.

## Final state

Game starts in a "ready" state; space bar or the Start button begins play and toggles pause;
input is queued so quick double-turns register; best score persists via `localStorage` where the
browser allows it and degrades gracefully where it doesn't.
