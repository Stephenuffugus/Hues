# GAME_CARD — Hues (Hue Match)

## At a glance
- **One-line hook:** Match a target color by feel before the clock runs out.
- **Genre / vibe:** Arcade color-perception puzzle; minimalist dark, tactile.
- **Core loop:** Drag a hue strip + a saturation/value pad to match the target swatch within tolerance before the per-round timer expires, lock it in, see your match%.
- **Status:** working
- **Live URL:** https://stephenuffugus.github.io/Hues/
- **Repo:** Stephenuffugus/Hues

## Tech
- **Stack:** Vanilla single-file HTML + CSS + JS, all inline in `index.html`. No framework, no dependencies.
- **Build step:** none.
- **Entry point:** `index.html`.
- **Controls:** Touch / pointer drag (hue strip + SV pad) and taps. Installable PWA (service worker + manifest), works offline.

## Existing economy
- **In-game score or currency:** Per-round points (CIEDE2000 accuracy + bonuses), Endless best, Daily streak, and **coins** (localStorage) spent in a cosmetic Border Shop. Untouched by Sunbeam.
- **What sunbeams mapped onto:** the two existing reward moments — every round completion, and the first Daily completion of the day. Coins and sunbeams sit side by side.

## Single-domain check
- **Would it work served at `lucidwinds.com/hues/`?** Yes. There are **no root-relative paths** in the app — every internal reference (`manifest.json`, `sw.js`, `icon-192.png`, the SW `register("sw.js")`, manifest `start_url`/`scope` of `./`) is relative, so it works unchanged under a subpath.
- External absolute URLs are only: Google Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`) and the Sunbeam SDK (`lucidwinds.com`). None break under a subpath.

## Sunbeam wiring shipped
- **gameId:** `hues`
- **SDK:** `<script src="https://lucidwinds.com/sunbeam-sdk.js?v=2">` in `<head>`, then `Sunbeam.init({gameId:"hues"})`. Every call is guarded with `window.Sunbeam && …` (so offline play, when the remote SDK can't load, never throws) and `.catch(function(){})`.
- **Earn events:**

| Trigger (in code) | amount | source label |
|---|---|---|
| Round complete — `lockGuess()`, fires once per round in every mode (Daily, Endless, Versus) | 4 | `hues:win` |
| First Daily completion of the day — `dailyResult()` `firstToday` block | 4 | `hues:daily` |

- **Session yield:** ~4 per round. A 5-round Daily ≈ 20 (+4 daily = 24); a casual Endless run ≈ 20–40. Within the 15–40 target; well under the 300/min, 5000/day caps.
- **Verified:** real headless Chromium session, console clean, `await Sunbeam.balance()` rose to `pending = 12` after 3 scored rounds (4 each); `localStorage['sws_pending_sunbeams']` recorded `lastSource: "hues:win"`. Zero console/page errors.
