# Hue Match — project guide (CLAUDE.md)

Perceptual color-matching game. Drag a hue strip + a saturation/value pad to match
a target color before a per-round timer expires. Scoring is **CIEDE2000** (real
Lab-space ΔE). Single self-contained `index.html` (HTML+CSS+JS inline), no build
step, no dependencies, installable PWA.

## Files
| File | Purpose |
|---|---|
| `index.html` | The entire game — HTML, CSS, and JS inline. Entry point. |
| `sw.js` | Service worker. Cache-first for same-origin. Bump `CACHE` when cached files change. |
| `manifest.json` | PWA manifest (standalone, portrait, dark theme). |
| `icon.svg` / `icon-192.png` / `icon-512.png` | App icons (hue ring + center swatch). PNGs are generated; see "Regenerating icons". |
| `README.md` | One-liner. |
| `CLAUDE.md` | This file — architecture + tunables. Keep it updated as things change. |

## Run locally
```
python3 -m http.server 8000   # then open http://localhost:8000/
```
The service worker only activates over **https/localhost** (not `file://`). That's
expected — don't try to "fix" it for file://.

## Architecture (all inside `index.html`)
- **`Store`** — localStorage wrapper with in-memory fallback so it never throws.
  **Always go through `Store`; never call `localStorage` directly.** Keys live in `K`.
- **Color pipeline** — `hsv2rgb` → `rgb2lab` → `deltaE2000` (Sharma-verified CIEDE2000).
  `dEofHSV(a,b)` is the convenience wrapper. **Do not replace CIEDE2000 with RGB/HSV
  distance** — it's what makes the game feel right.
- **Player-facing closeness** — everything funnels through `matchPct(dE)` and
  `closeness(dE)`. Never show raw ΔE. Keep all closeness going through these two.
- **`mulberry32(seed)`** — seeded RNG. Daily/Versus are seeded (same puzzle for
  everyone via the date seed); Endless uses `Math.random`.
- **`makeRound(rng,level,diff)`** — builds target/start colors + `time`/`tol` from `CONFIG.curves`.
- **Game state** — single global `G`. Screens: `menu`/`game`/`result`/`shop` via `show()`.
- **Between rounds** — never auto-advances. `showBreakdown()` slides up a sheet with
  this round's point breakdown + a scrollable review of every round so far. The next
  round's timer starts only when the player taps **Next round**. **Do not reintroduce
  auto-advance.**
- **Economy** — coins earned every round (win or lose) → cosmetic Border Shop (`BORDERS`).
- **Missions** — 3 seeded daily goals (`MISSION_POOL` → `pickMissions`), auto-pay coins.
- **Feel** — live countdown number + bar, danger vignette, WebAudio `beep()` + `buzz()` haptics.

## CONFIG — all tunables live in one block at the top of `<script>`
Edit values in `CONFIG` to balance-test; nothing below it redefines these numbers.

| Path | Current | Meaning |
|---|---|---|
| `pctScale` | `72` | `match% = round(100·exp(-dE/pctScale))`. Higher = more generous %. |
| `closeness` | `{bullseye:1.5, veryClose:4, close:10}` | ΔE thresholds for the closeness label. |
| `curves.casual` | `{t0:11,td:0.16,tmin:4.5,c0:34,cd:0.9,cmin:11}` | time=`max(tmin,t0-L·td)`, tol=`max(cmin,c0-L·cd)`. |
| `curves.normal` | `{t0:9,td:0.20,tmin:3.6,c0:30,cd:1.2,cmin:8}` | " |
| `curves.hard` | `{t0:7,td:0.28,tmin:2.7,c0:26,cd:1.6,cmin:6}` | " |
| `gen.target` | `{sMin:0.42,sRange:0.58,vMin:0.36,vRange:0.58}` | HSV ranges for the target color. |
| `gen.start` | `{sMin:0.30,sRange:0.50,vMin:0.40,vRange:0.40}` | HSV ranges for the starting guess. |
| `score.baseWeight` | `700` | `base = round(acc²·baseWeight)`, `acc = max(0,1-dE/tol)`. |
| `score.lockBonus` | `40` | flat reward for locking in (not timing out). |
| `score.precWeight` | `240` | `precision = round((1-dE/fine)·precWeight)`. |
| `score.speedWeight` | `300` | `speed = round((timeLeft/time)·speedWeight)`, **accuracy-gated** on `dE≤fine`. |
| `score.fineFactor` / `fineFloor` | `0.28` / `3` | `fine = max(fineFloor, tol·fineFactor)` — gate for precision+speed bonuses. |
| `score.timeoutMult` | `0.7` | timeout: `base·=timeoutMult`, all bonuses forfeited (−30%). |
| `tiers` | `{gold:3, green:8}` | daily share squares: dE≤gold 🟩, ≤green 🟨, else ⬛. |
| `perfectDE` | `1.5` | dE≤ this = a "perfect" (mission credit + chime + glow). |
| `coinsPerPoint` | `120` | coins/round = `floor(roundTotal/coinsPerPoint)`. |
| `lives` | `3` | endless starting lives + life cap. |
| `stage` | `{every:5, bonusPer:15, lifeReward:1}` | every Nth level: `+bonusPer·stage` coins, `+lifeReward` life (capped at `lives`). |
| `dailyBonus` | `{base:20, perStreak:3}` | first daily clear of the day: `base + streak·perStreak` coins. |
| `roundsPerSet` | `5` | rounds in Daily & Versus. |
| `borderPrices` | `hairline:0, bevel:80, frost:180, brutalist:240, gold:350, neon:420, deco:550, prism:900` | shop prices (coins). |
| `missionRewards` | `score4000:40, score7000:70, level10:45, perfect3:35, daily:30, rounds20:25, coins60:30` | mission payouts (coins). |
| `timer` | `{warnAt:0.5, badAt:0.22, dangerAt:0.33}` | fractions of time left for colour/danger feedback. |

## Modes
- **Daily** — 5 seeded rounds, streak, shareable squares result. First clear/day pays `dailyBonus`.
- **Endless** — 3 lives, difficulty ramps with level, a Stage clears every 5 levels
  (+1 life capped at 3, bonus coins), live BEST-chase.
- **Versus** — local pass-and-play, same seeded rounds for both players.

## Scoring (per round)
`base` (accuracy²) + `lockBonus` + `precision` (within `fine`) + `speed`
(accuracy-gated). Timeout → base×0.7 and forfeits all bonuses.

## Hard constraints — do not
- Don't replace CIEDE2000 with RGB/HSV distance.
- Don't break the single-file structure of the game logic without a clear reason.
- Don't call `localStorage` directly — always go through `Store`.
- Don't reintroduce auto-advance between rounds; the player initiates the next round.
- Keep all player-facing closeness going through `matchPct`/`closeness`.
- The service worker only activates over https/localhost (expected).

## Housekeeping
- Bump `CACHE` in `sw.js` whenever you change cached files (currently `huematch-v4`).
- Update this file as you change things.

## Regenerating icons
PNGs were generated by a small pure-JS encoder (no ImageMagick needed). The design is
a full-spectrum hue ring + center swatch on `#0a0a0c`. The generator script is kept at
`/tmp/genicon.js` during dev; re-run with `node` if the icon art changes, or rasterize
`icon.svg` with any tool that supports `<foreignObject>` conic-gradient masks.

## Known follow-ups / not yet done
- **Sunbeam economy (Sky Wolf Studios):** additive cross-game currency SDK to be wired
  at existing reward moments — pending integration rundown from the studio. Additive only.
- **Global daily leaderboard + cross-device sync + tamper-resistant coins** — needs a
  backend (Firebase). Daily is already same-puzzle-for-everyone via the date seed.
- **Self-host the 3 Google Fonts** for true offline fidelity (currently CDN + system fallback).
- **Balance:** Casual + stage life-refill trends toward effectively-endless; accuracy-gated
  speed bonus gets hard to earn once `fine` hits its floor of 3 mid/late run. See git history.
