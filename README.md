# webosumania-autoplay

A userscript that auto-plays maps on [webosumania.com](https://webosumania.com/) (the web port of osu!mania by HecticKiwi). Hooks straight into the game engine through the React fiber, so it's not simulating keyboard events — it calls the input system directly.

Currently on v3.4.

## What it does

- Press F2 in-game to arm auto-play.
- Press any game key to start the song.
- It hits every note. Perfectly, by default. Or with realistic human-ish timing error if you dial in `HUMANIZE`.
- Hold notes handled (releases at the right endTime, per-note jitter).
- Small overlay in the top-right tells you state and note count.

## Install

Tampermonkey → Create a new script → paste the contents of `Web osu!mania Auto-Player-3.4.txt` → Save. That's it. Refresh the site.

Also works if you just paste the whole thing into the browser console (F12) once per page load, but you'll lose it on refresh.

## Config

Poke at these from the console while a map is loaded:

```js
autoplayConfig.HIT_OFFSET = -10   // hit 10ms early (or +10 for late)
autoplayConfig.HUMANIZE   = 5     // ms of std deviation on hit timing
autoplayConfig.TAP_HOLD_MS = 40   // how long each tap is held
autoplayConfig.TOGGLE_KEY = "F2"  // rebind if F2 conflicts with something
```

`HUMANIZE` uses a Gaussian distribution clamped to ±3σ with a small late bias, because in my experience real players push slightly late, not early. Rough guide:

| value | what it looks like |
|-------|-----|
| 0  | Robot. Full MAX/320. |
| 5  | Very accurate human. Mostly 320s, occasional 300. |
| 10 | Good player. Mix of 320 and 300, a few 200s. |
| 18 | Decent player. Mostly 300, occasional 200/100. |
| 25+| Sloppy. Full spread. |

## Why it exists

I wanted an autoplay for local practice and to record showcase clips without shakily typing 200 keys a second. The upstream game doesn't ship one, and every "web osu mania auto" thing I found online was either simulating fake keyboard events (which get flagged as key-repeat and eaten by the input handler) or scraping the DOM (which the game doesn't render — it's a PIXI canvas).

So this one goes through the fiber and calls `game.inputSystem.hit(col)` / `.release(col)` directly. Same code path the real keyboard handler uses.

## How it works, briefly

1. Walk the React fiber tree from the canvas element up looking for the Game instance. The check is duck-typed: has `inputSystem`, `scoreSystem`, `columns`, `hitObjects`, `song`, `timeElapsed`, and (as of v3.4) `currentColumnIndices`.
2. Once found, run a 1ms polling loop.
3. Each tick, for every column, look at `game.columns[col][game.currentColumnIndices[col]]` — the current active note.
4. If it's a `tap`, hit at `data.time`. If the tap has `isHoldHead` and a real `endTime`, don't release; the engine will advance the index to the Hold body.
5. If it's a `hold` body, schedule a `release()` for `data.endTime`.
6. When `window.__PIXI_APP__` changes (retry, next song, back to menu), drop the ref and rescan.

## Broken? Probably an engine update

Every time HecticKiwi refactors the game internals this script needs adjusting. v3.3 broke because the engine changed columns from a live sprite-queue-with-`.shouldRemove` model to a fixed array + `currentColumnIndices` pointer. v3.4 is the fix for that.

Things that would break it again:
- Renaming `inputSystem` / `hit` / `release` on the Game class.
- Removing `window.__PIXI_APP__`.
- Changing the shape of `TapData` / `HoldData`.
- Reorganising the fiber tree deeply enough that the walker can't find the state hook.

The `isGame()` check is the fastest place to look if it stops detecting the game.

## Honest disclaimers

- **Made with AI help.** Wrote it with an LLM as a pair programmer and some knowledge of the codebase from reading it. I know the parts that matter; I don't pretend I typed every semicolon.
- **Local only.** Don't upload runs to anywhere that has leaderboards or considers this cheating. It's for your own machine.
- **No warranty.** It will break on the next engine refactor and I'll patch it when I care to. PRs welcome if you get there first.
- **Not affiliated** with HecticKiwi, osu!, or ppy in any way.

## License

MIT. Do whatever.
