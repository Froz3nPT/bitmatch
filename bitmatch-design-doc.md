# Bitmatch — Design & Project State

A speed-focused match-3 puzzle game with bosses, combos, and (eventually) chibi anime/JRPG-style pixel art.

This document is the source of truth for the project's design intent, current state, and roadmap. Read this before making changes.

---

## Vibe & Direction

- **Inspiration:** Zookeeper (the speed/cascade rhythm), Candy Crush (persistent special tiles), Pokémon Shuffle (loadout + creature collection meta).
- **Visual goal:** Original "chibi JRPG" pixel-art critters with big eyes — anime/game *style*, not actual recognisable IP. No Pikachu, no Kirby, no Cloud. Currently using emoji as placeholders.
- **Audio:** Bouncy candy-crush-ish procedural music via Tone.js. Light SFX. To be replaced with real bundled audio when we move to Capacitor.
- **Platform:** Mobile-first, portrait, Android primary target. PWA today, real APK via PWABuilder, eventual Capacitor wrap for Play Store.

## Naming

- **Game name:** Bitmatch
- **Repo:** `bitmatch` (public, github.com/Froz3nPT/bitmatch)
- **Theme color:** `#6d5ba6` (purple gradient midpoint)

---

## Current Architecture

**Single file:** `index.html` contains the entire game (HTML, CSS, JS, ~2500 lines). Loads Tone.js from CDN.

**Companion files:**
- `manifest.json` — PWA manifest, `display: fullscreen`
- `sw.js` — service worker, cache-first for own assets
- `icon-192.png`, `icon-512.png`, `icon-maskable-512.png` — placeholder icons

**This single-file architecture is intentional for the MVP but is hitting its limits.** First task in Claude Code is refactoring it into modules.

### Proposed module split (for refactor)

```
src/
  index.html              (shell + script tags)
  css/
    styles.css            (all styles, currently inline)
  js/
    main.js               (boot, screen routing)
    game/
      board.js            (grid state, tile lifecycle, drop/spawn)
      matches.js          (findMatches, expandSpecial)
      resolve.js          (processClearWithUpgrades, processClearSet, cascades)
      input.js            (drag/tap handlers)
      stage.js            (startStage, endStage, tick, win/fail logic)
    meta/
      progress.js         (load/save, achievements, stars)
      loadout.js          (renderLoadout, computeActivePerks)
      themes.js           (THEMES constant, theme switching)
      bosses.js           (BOSS_PERKS, boss UI, boss mechanics)
      stages-data.js      (STAGES array)
      achievements.js     (ACHIEVEMENTS array, unlock logic)
    fx/
      audio.js            (Tone.js setup, sfx(), music)
      haptics.js          (vibrate())
      visuals.js          (showFloat, showCombo, victory, toasts)
    ui/
      screens.js          (showScreen, screen rendering)
      stages-screen.js    (renderStages)
      collection.js       (renderCollection)
      achievements-ui.js  (renderAchievements)
```

Bundle with **Vite** (fast, simple, builds to single optimized HTML for Pages). No framework — vanilla JS is fine and avoids a ton of complexity.

---

## Core Mechanics (DO NOT BREAK)

### Match-3 with persistent specials (Candy Crush style)
- **Match 4 in a row** → striped tile (clears row or column when next matched)
- **Match 5+ in a line** → rainbow color bomb (clears all tiles of one type when matched)
- **Specials chain** — striped triggering striped triggering rainbow, etc.
- **Special swap interactions:**
  - Striped + anything → striped activates
  - Rainbow + regular tile → clears all of regular's type
  - Rainbow + Rainbow → clears entire board ("MEGA")

### Drag & tap input
- Drag any tile onto any other tile to swap (no adjacency requirement)
- Drag threshold 6px before becoming a drag (tap-to-select still works as fallback)
- Invalid swaps revert with red flash, count as `invalidSwaps` stat

### Combo & Rush system
- **Combo only builds from PLAYER MOVES**, never from cascades. This was a deliberate fix; do not regress.
- Combo window: 1500ms between successful player matches
- Combo caps for scoring at 5x (multiplier in score formula)
- Combo caps numerically at 10x (for achievements)
- **Rush** triggers at combo ≥ 6, lasts 7 seconds, does NOT refresh on re-trigger
- During Rush: drop animations are faster (280ms vs 520ms), small score bonus per tile

### Cascades & rhythm (Zookeeper-style)
- Tiles in a match clear with **40ms stagger** (top-left → bottom-right sweep)
- Pop animation: scale up to 1.4x, rotate 8°, flash white, shrink to nothing (420ms total)
- Drop after stagger completes
- Cascade delay: 720ms (normal) / 460ms (rush) AFTER drops finish — explicit "settle moment" between cascades
- This rhythm is critical to the feel. Do not flatten.

### Auto-end on target hit
- Stage ends *immediately* when score crosses target (not when timer runs out)
- Victory animation: green board glow + spinning float
- Stars based on time/moves remaining at win:
  - ≥50% remaining = 3 stars
  - ≥20% = 2 stars
  - <20% = 1 star
  - 0 if failed

---

## Stages (20 total)

| # | Types | Time | Target | Special |
|---|-------|------|--------|---------|
| 1 | 3 | 55s | 1500 | tutorial baseline |
| 2 | 3 | 55s | 2800 | |
| 3 | 3 | 50s | 4500 | |
| 4 | 3 | 45s | 6500 | |
| 5 | 3 | 50s | 8500 | **⚔️ TEMPO boss** — 18 moves limit |
| 6 | 4 | 55s | 4500 | |
| 7 | 4 | 55s | 7000 | |
| 8 | 4 | 50s | 9500 | |
| 9 | 4 | 45s | 12500 | |
| 10 | 4 | 60s | 16000 | **🌀 TWISTER boss** — shuffle every 14s |
| 11 | 4 | 55s | 10000 | |
| 12 | 4 | 55s | 13500 | |
| 13 | 5 | 50s | 17500 | |
| 14 | 5 | 45s | 22000 | |
| 15 | 5 | 60s | 27500 | **❄️ GLACIER boss** — freeze tile every 10s |
| 16 | 5 | 55s | 16000 | |
| 17 | 5 | 55s | 21000 | |
| 18 | 5 | 50s | 27000 | |
| 19 | 5 | 45s | 34000 | |
| 20 | 6 | 65s | 43000 | **👾 APEX boss** — 32 moves + shuffle every 18s |

**Difficulty principle:** 4 types is the standard cap. 5 only for harder mid/late stages. 6 reserved for the final fight.

---

## Themes

5 themes defined in `THEMES` constant. Each has 6 base tiles + 1 boss tile.

| Theme | Unlock | Base Tiles | Boss |
|-------|--------|------------|------|
| Animals | start | 🦁🐵🐸🐼🦊🐨 | 🐲 Dragon |
| Fruits | stage 5 | 🍎🍌🍇🍓🥝🍊 | 🍍 Pineapple King |
| Space | stage 10 | ⭐🌙☀️🪐🌎👽 | 🛸 UFO Overlord |
| Faces | stage 15 | 😀😎😍🥳🤩😂 | 🤡 Mad Jester |
| Food | stage 20 | 🍕🍔🍩🌮🍜🥐 | 🌶️ Inferno Pepper |

**Future:** emoji is placeholder. The plan is original chibi pixel-art assets per tile. Loading them as PNG sprites in CSS `background-image` should be the cleanest swap (avoids touching game logic).

---

## Boss Perks (Pokémon Shuffle inspired)

When a defeated boss tile is in your active loadout, you get a passive buff:

| Boss | Perk |
|------|------|
| 🐲 Dragon | +2% score |
| 🍍 Pineapple King | +5% score |
| 🛸 UFO Overlord | +3s start time |
| 🤡 Mad Jester | +3% score |
| 🌶️ Inferno Pepper | +5s + 3% score |

Stack multiplicatively for score, additively for time.

---

## Loadout System

- **Up to 6 critters** picked from the player's full unlocked roster (any theme + defeated bosses)
- Used in order per stage — stage with `types: 3` uses the first 3 slots
- If loadout is empty or insufficient, falls back to the currently selected theme's base tiles
- Boss perks only active when their tile is in the slice that's actually in play
- Persists across sessions

---

## Achievements (20 total)

Stored as Set of IDs in progress. Toast notification + sound + vibration on unlock. See `ACHIEVEMENTS` constant for full list.

Categories:
- Stage completion (first_blood, halfway, completionist)
- Star collection (perfectionist, star_collector, grandmaster)
- Combo skill (combo_maestro, combo_legend)
- Rush usage (rush_hour, rush_master)
- Special tiles (rainbow_maker, striped_up, chain_reaction)
- Style (pacifist, speedrunner, big_match, world_tour)
- Volume (critter_hunter, critter_slayer, boss_slayer)

---

## Audio (Tone.js procedural)

**Music:** 4-loop polyphonic — bass walk, plucky arpeggio, sustained pad, kick. ~118 BPM, C major I-vi-IV-V progression. Plays on first user gesture (browser audio unlock requirement).

**SFX:** swap, match (pitch rises with size), combo (rises with combo number), rush, special, bossHit, victory, fail, invalid, achievement.

**Toggleable** via 🔊/🔇 button in title bar. State persisted.

**Future:** Replace with bundled real audio when we move to Capacitor. Tone.js procedural is "fine but obviously procedural."

---

## Haptics (`navigator.vibrate`)

- 8ms tap on valid swap
- Scaling pulse on combo 2-10
- `[40,40,80]` on rush start
- Patterns intensify with special activations
- Boss unlock: `[80,40,80,40,160]`
- Victory: `[50,50,50,50,200]`
- Toggleable via 📳/📴 button. State persisted.
- Silently no-ops on iOS (Apple never implemented `navigator.vibrate`).

---

## Persistence (localStorage)

Keys:
- `bitmatch_save_v1` — full progress object
- `bitmatch_settings_v1` — audio/haptics toggles

**Sets are serialized as arrays** (JSON limitation), rehydrated on load. The `_v1` suffix is intentional — if schema changes, write a migration, don't bump silently.

Saves on:
- Stage end (pass or fail)
- Setting toggles
- Theme/loadout changes
- `visibilitychange` to hidden
- `beforeunload`

Silently no-ops if `localStorage` unavailable (e.g., Claude artifact sandbox). Works everywhere else.

---

## PWA & Mobile Wrap

**Current state:**
- Manifest with `display: fullscreen`, `display_override: ['fullscreen', 'standalone']`
- Service worker with cache-first for own assets, network for everything else
- Cache key: `bitmatch-v1` — bump version when content changes meaningfully
- Hosted on GitHub Pages: `https://froz3npt.github.io/bitmatch/`
- APK generated via PWABuilder, signed with a keystore the user holds

**Known issues:**
- Address bar still showing in some launch contexts (still investigating — may be PWABuilder TWA wrapper limitation)
- Procedural music doesn't survive screen-off (browser limitation)

---

## What's Done

- ✅ Core match-3 with drag/tap input
- ✅ Persistent specials (striped, color bomb)
- ✅ Combo + Rush system (player-moves only)
- ✅ 20 stages with 4 boss fights
- ✅ Boss HP bars + boss display
- ✅ Boss tile unlocks (kill boss → tile joins your roster)
- ✅ Loadout system with perks
- ✅ 20 achievements with toast notifications
- ✅ Critter-Dex (collection grid)
- ✅ 5 themes with progressive unlock
- ✅ Auto-win on target reached
- ✅ Stars based on time/moves remaining
- ✅ Music + SFX (procedural Tone.js)
- ✅ Vibration patterns
- ✅ localStorage persistence (progress + settings)
- ✅ Manifest + service worker (PWA installable)
- ✅ Hosted on GitHub Pages
- ✅ APK via PWABuilder

---

## What's Next (Roadmap)

### Immediate (priority order)

1. **Refactor single HTML into proper modules** — Vite project, file split per the proposed structure above. No behavior changes; just architecture. Verify game still works identically.
2. **Investigate fullscreen issue on Android** — user reports address bar still appearing despite `display: fullscreen`. May need TWA-specific tweaks in PWABuilder, or it may be a Chrome Custom Tab launch context issue.
3. **Tune cascade feel** — user wants more Zookeeper-like pacing. Current settings: 40ms stagger, 420ms pop, 720/460ms cascade delay. Adjust if needed.

### Soon

4. **Asset pipeline for chibi pixel art.** Plan: each theme tile becomes a PNG sprite. Store in `assets/themes/<theme-name>/<tile-id>.png`. Update tile rendering to use `background-image` instead of emoji text. Keep emoji as fallback if asset missing.
5. **Daily challenge mode** (deferred earlier) — once art is in.
6. **Pre-stage power-ups** (deferred) — Pokémon Shuffle style, non-game-destructive small advantages.

### Eventually

7. **Capacitor wrap** for proper native APK with:
   - Real bundled music files (replace Tone.js procedural)
   - Native haptics via `@capacitor/haptics`
   - Persistent storage via `@capacitor/preferences`
   - Splash screen
8. **Play Store listing** ($25 one-time Google fee) — needs screenshots, description, content rating, privacy policy.

---

## Communication Style Notes (for Claude Code)

The project owner (Eduardo) prefers:
- Direct, concise responses with no flattery or jargon
- Honest tradeoff analysis, not just yes-and
- "Here's what I'd recommend and why" over "what would you like?"
- Doesn't want copies of the game's UI text in chat — just code and decisions
- Worked in PM/PO roles, has Java/SQL/frontend background — comfortable with technical detail
