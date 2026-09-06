// ==UserScript==
// @name         Web osu!mania Auto-Player
// @namespace    webosumania-autoplay
// @version      3.4
// @description  Auto-plays maps on webosumania.com — hooks directly into the game engine
// @author       AutoPlayer
// @match        https://webosumania.com/*
// @match        https://web-osu-mania.pages.dev/*
// @match        https://hectickiwi.github.io/Web-Osu-Mania/*
// @run-at       document-idle
// @grant        none
// ==/UserScript==

/**
 * ╔═══════════════════════════════════════════════════════════════╗
 * ║           Web osu!mania Auto-Player v3.4                     ║
 * ╠═══════════════════════════════════════════════════════════════╣
 * ║                                                               ║
 * ║  INSTALL:                                                     ║
 * ║    Tampermonkey → New Script → paste this → Save              ║
 * ║    OR paste into browser console (F12)                        ║
 * ║                                                               ║
 * ║  USAGE:                                                       ║
 * ║    1. Open webosumania.com → pick any map → click Play        ║
 * ║    2. Press F2 to toggle auto-play ON                         ║
 * ║    3. Press any GAME KEY (not F2) to start the song           ║
 * ║                                                               ║
 * ║  F2 = toggle ON/OFF                                           ║
 * ║                                                               ║
 * ║  Console tweaks:                                              ║
 * ║    autoplayConfig.HIT_OFFSET = -10  (hit 10ms early)          ║
 * ║    autoplayConfig.HUMANIZE = 5      (±5ms random jitter)      ║
 * ║                                                               ║
 * ║  ─── v3.4 changelog ────────────────────────────────────────  ║
 * ║  Updated to match new game engine (index-pointer model).      ║
 * ║  Uses game.currentColumnIndices[col] instead of sprites[0].   ║
 * ║  Hold body is now a separate sprite; released via release().  ║
 * ║                                                               ║
 * ╚═══════════════════════════════════════════════════════════════╝
 */

(function () {
    "use strict";

    const CONFIG = {
        TOGGLE_KEY: "F2",
        POLL_MS: 1,
        HIT_OFFSET: 0,
        // HUMANIZE behavior (ms std deviation):
        //   0       = perfect robot (all Marvelous/320)
        //   5       = very accurate human (~95% 320, ~5% 300)
        //   10      = good player (mix of 320 and 300, tiny 200s)
        //   18      = decent player (mostly 300, some 320, occasional 200)
        //   25+     = sloppy (full distribution including 100s)
        HUMANIZE: 0,
        TAP_HOLD_MS: 40,
    };

    // ─── HIT WINDOW REFERENCE (at OD8, typical) ────────────────
    // 320/MAX: ±16.1ms  | 300: ±40ms  | 200: ±73ms  | 100: ±103ms  | 50: ±127ms


    let enabled = false;
    let gameRef = null;
    let loopId = null;

    // Per-column state tracking
    //   { tapTimer, holdReleaseAt, holdReleaseArmed, lastIndex }
    let colState = [];

    // ─── OVERLAY ───────────────────────────────────────────────
    function ensureOverlay() {
        if (document.getElementById("ap3")) return;
        const el = document.createElement("div");
        el.id = "ap3";
        el.innerHTML = `<div style="
      position:fixed;top:10px;right:10px;z-index:999999;
      background:rgba(0,0,0,.92);color:#fff;
      font:600 13px/1.4 'Segoe UI',system-ui,sans-serif;
      padding:8px 14px;border-radius:8px;
      border:1px solid rgba(255,255,255,.12);
      pointer-events:none;user-select:none;
      backdrop-filter:blur(8px);
    ">
      <span id="ap3d" style="display:inline-block;width:8px;height:8px;border-radius:50%;background:#f44;vertical-align:middle;margin-right:6px"></span>
      <span id="ap3l">AUTO OFF</span>
      <span style="opacity:.4;margin-left:6px">[F2]</span>
      <div id="ap3i" style="font-weight:400;font-size:11px;opacity:.55;margin-top:2px">Waiting for game...</div>
    </div>`;
        document.body.appendChild(el);
    }

    function setOvr(on, info) {
        const d = document.getElementById("ap3d");
        const l = document.getElementById("ap3l");
        const i = document.getElementById("ap3i");
        if (!d) return;
        d.style.background = on ? "#4f4" : "#f44";
        l.textContent = on ? "AUTO ON" : "AUTO OFF";
        if (info !== undefined) i.textContent = info;
    }

    // ─── FIND GAME via React Fiber ─────────────────────────────
    function findGame() {
        if (!window.__PIXI_APP__) return null;
        const canvas = document.querySelector("canvas");
        if (!canvas) return null;

        let el = canvas;
        while (el) {
            for (const key of Object.keys(el)) {
                if (key.startsWith("__reactFiber$") || key.startsWith("__reactInternalInstance$")) {
                    const g = walkFiber(el[key], 0);
                    if (g) return g;
                }
            }
            el = el.parentElement;
        }

        for (const id of ["__next", "root", "app"]) {
            const root = document.getElementById(id);
            if (!root) continue;
            for (const key of Object.keys(root)) {
                if (key.startsWith("__reactFiber$") || key.startsWith("__reactInternalInstance$")) {
                    const g = walkFiber(root[key], 0);
                    if (g) return g;
                }
            }
        }
        return null;
    }

    function walkFiber(fiber, depth) {
        if (!fiber || depth > 80) return null;
        let hook = fiber.memoizedState;
        while (hook) {
            const s = hook.memoizedState;
            if (isGame(s)) return s;
            if (s && s.current && isGame(s.current)) return s.current;
            if (hook.queue && isGame(hook.queue.lastRenderedState)) return hook.queue.lastRenderedState;
            hook = hook.next;
        }
        return walkFiber(fiber.child, depth + 1) || walkFiber(fiber.sibling, depth + 1);
    }

    function isGame(o) {
        return (
            o && typeof o === "object" &&
            o.inputSystem && o.scoreSystem &&
            o.columns && Array.isArray(o.columns) &&
            o.hitObjects && Array.isArray(o.hitObjects) &&
            o.song && typeof o.timeElapsed === "number" &&
            // NEW ENGINE: has currentColumnIndices array
            Array.isArray(o.currentColumnIndices)
        );
    }

    // ─── TIMING ────────────────────────────────────────────────
    function getTime() {
        if (!gameRef || !gameRef.song) return -1;
        // game.timeElapsed is set to Math.round(song.seek()*1000) each PLAY frame.
        // Prefer song.seek() for sub-ms precision.
        try {
            const s = gameRef.song.seek();
            if (typeof s === "number" && s >= 0) return s * 1000;
        } catch (e) {}
        if (typeof gameRef.timeElapsed === "number") return gameRef.timeElapsed;
        return -1;
    }

    // ─── HUMANIZE: Gaussian (normal) distribution ──────────────
    function gaussianRandom() {
        const u1 = Math.random();
        const u2 = Math.random();
        return Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2);
    }

    function rollOffset() {
        if (CONFIG.HUMANIZE <= 0) return 0;
        const sigma = CONFIG.HUMANIZE;
        let offset = gaussianRandom() * sigma;
        const maxErr = sigma * 3;
        if (offset > maxErr) offset = maxErr;
        if (offset < -maxErr) offset = -maxErr;
        // Slight bias toward late — humans typically react late, not early
        offset += sigma * 0.15;
        return offset;
    }

    // ─── AUTO-PLAY TICK ────────────────────────────────────────
    //
    // NEW ENGINE MODEL (v3.4+):
    //
    //   game.columns[col]                = fixed array of Tap|Hold sprites for this column
    //   game.currentColumnIndices[col]   = index of the CURRENT active hit object
    //                                       (advances on hit / release / miss)
    //
    //   Front sprite = game.columns[col][game.currentColumnIndices[col]]
    //     sprite.data.type    = "tap" | "hold"
    //     sprite.data.time    = note start time (ms)
    //     sprite.data.endTime = note end time (ms; equals time for plain taps)
    //     sprite.data.isHoldHead = true on the tap head of a hold note
    //
    // A hold note = TWO consecutive entries in the column:
    //   1. Tap  (data.type="tap",  data.isHoldHead=true,  data.endTime = holdEnd)
    //   2. Hold (data.type="hold",                        data.endTime = holdEnd)
    //
    // Flow for a hold note:
    //   - inputSystem.hit(col)     → scores the Tap head, index advances to Hold body
    //   - keep the key pressed
    //   - inputSystem.release(col) at endTime → Hold.release() scores it, index advances
    //
    function tick() {
        if (!enabled || !gameRef || gameRef.state !== "PLAY") return;

        // Don't use a stale game ref (happens during retry before scanner catches up)
        if (gameRef.app !== window.__PIXI_APP__) return;

        const t = getTime();
        if (t < 0) return;
        const adj = t + CONFIG.HIT_OFFSET;

        const kc = gameRef.difficulty ? gameRef.difficulty.keyCount : 4;

        while (colState.length < kc) {
            colState.push({ tapTimer: null, holdReleaseAt: -1, holdReleaseArmed: false, lastIndex: -1 });
        }

        for (let col = 0; col < kc; col++) {
            const cs = colState[col];

            const column = gameRef.columns[col];
            if (!column) continue;

            const idx = gameRef.currentColumnIndices[col];
            if (typeof idx !== "number") continue;

            // If the engine advanced past a note (hit/miss/release), reset per-note state
            if (cs.lastIndex !== idx) {
                cs.lastIndex = idx;
                cs.holdReleaseAt = -1;
                cs.holdReleaseArmed = false;
            }

            const front = column[idx];
            if (!front || !front.data) continue;

            const d = front.data;

            // ── Case 1: current sprite is the BODY of a hold ──
            if (d.type === "hold") {
                // We should already be holding the key from when we hit the head.
                // Schedule the release for its endTime (with per-note jitter).
                if (!cs.holdReleaseArmed) {
                    cs.holdReleaseArmed = true;
                    cs.holdReleaseAt = d.endTime + rollOffset();
                }
                if (adj >= cs.holdReleaseAt) {
                    try { gameRef.inputSystem.release(col); } catch (e) {}
                    // Engine will advance currentColumnIndices[col]; next tick resets cs.
                }
                continue;
            }

            // ── Case 2: current sprite is a TAP (possibly hold head) ──
            if (d.type !== "tap") continue;

            // Per-note random offset (assigned once, persists on the sprite)
            if (front.__apOffset === undefined) {
                front.__apOffset = rollOffset();
            }
            const targetTime = d.time + front.__apOffset;
            if (adj < targetTime) continue;

            // ── Hit this note ──
            // Cancel any pending quick-release from a previous tap in this column
            if (cs.tapTimer) {
                clearTimeout(cs.tapTimer);
                cs.tapTimer = null;
            }

            // If the column is still marked pressed (from a stale prior tap
            // whose release timer hasn't fired), release first so hit() re-fires.
            // release() on a Tap sprite is a no-op scoring-wise (Tap.release()
            // is empty) and does NOT advance the index.
            if (gameRef.inputSystem.pressedColumns &&
                gameRef.inputSystem.pressedColumns[col]) {
                try { gameRef.inputSystem.release(col); } catch (e) {}
            }

            // Press the key
            try { gameRef.inputSystem.hit(col); } catch (e) {}

            // Was this a hold head? If so, the engine has now advanced the
            // index onto the Hold body; the "d.type==='hold'" branch above
            // will handle the timed release. Don't schedule a tap release.
            const isHoldHead = d.isHoldHead === true && d.endTime > d.time;
            if (isHoldHead) continue;

            // Regular tap — release after a short delay
            const c = col;
            cs.tapTimer = setTimeout(() => {
                if (gameRef && gameRef.inputSystem) {
                    try { gameRef.inputSystem.release(c); } catch (e) {}
                }
                if (colState[c]) colState[c].tapTimer = null;
            }, CONFIG.TAP_HOLD_MS);
        }
    }

    // ─── GAME DETECTION ────────────────────────────────────────
    function startScanning() {
        setInterval(() => {
            // Same-game guard: if PIXI app is unchanged, we already have the right ref
            if (gameRef && window.__PIXI_APP__) {
                if (gameRef.app === window.__PIXI_APP__) return;

                // PIXI app changed → game was recreated (retry / new song)
                console.log("[AutoPlay] 🔄 Game recreated (retry/new song), re-scanning...");
                resetColState();
                gameRef = null;
            }

            if (!window.__PIXI_APP__) {
                if (gameRef) {
                    resetColState();
                    gameRef = null;
                    setOvr(enabled, "Waiting for game...");
                }
                return;
            }

            const g = findGame();
            if (g) {
                gameRef = g;
                resetColState();
                const n = g.hitObjects ? g.hitObjects.length : 0;
                const k = g.difficulty ? g.difficulty.keyCount : "?";
                console.log(`[AutoPlay] ✅ Game: ${n} notes, ${k}K`);
                setOvr(enabled, `${n} notes | ${k}K`);
            }
        }, 500);
    }

    function resetColState() {
        colState.forEach(cs => {
            if (cs && cs.tapTimer) {
                clearTimeout(cs.tapTimer);
                cs.tapTimer = null;
            }
        });
        colState = [];
    }

    // ─── TOGGLE ────────────────────────────────────────────────
    function toggle() {
        enabled = !enabled;

        if (enabled) {
            console.log("[AutoPlay] ✅ ON");
            resetColState();
            if (!loopId) loopId = setInterval(tick, CONFIG.POLL_MS);

            if (!gameRef) {
                const g = findGame();
                if (g) {
                    gameRef = g;
                    const n = g.hitObjects.length;
                    const k = g.difficulty.keyCount;
                    setOvr(true, `${n} notes | ${k}K`);
                } else {
                    setOvr(true, "Waiting for game...");
                }
            } else {
                setOvr(true, `${gameRef.hitObjects.length} notes | ${gameRef.difficulty.keyCount}K`);
            }
        } else {
            console.log("[AutoPlay] ❌ OFF");
            if (loopId) { clearInterval(loopId); loopId = null; }

            if (gameRef && gameRef.inputSystem && gameRef.difficulty) {
                for (let c = 0; c < gameRef.difficulty.keyCount; c++) {
                    try { gameRef.inputSystem.release(c); } catch (e) {}
                }
            }
            resetColState();
            setOvr(false, gameRef ? "Paused" : "Waiting for game...");
        }
    }

    // ─── KEY LISTENER ──────────────────────────────────────────
    document.addEventListener("keydown", (e) => {
        if (e.key === CONFIG.TOGGLE_KEY || e.code === CONFIG.TOGGLE_KEY) {
            e.preventDefault();
            e.stopPropagation();
            toggle();
        }
    }, true);

    // ─── INIT ──────────────────────────────────────────────────
    window.autoplayConfig = CONFIG;

    function init() {
        console.log(
            "%c[osu!mania AutoPlayer v3.4]%c Press F2 to toggle.",
            "color:#4f4;font-weight:bold;font-size:14px",
            "color:#ccc;font-size:14px"
        );
        ensureOverlay();
        startScanning();
    }

    if (document.readyState === "complete" || document.readyState === "interactive") {
        init();
    } else {
        document.addEventListener("DOMContentLoaded", init);
    }
})();
