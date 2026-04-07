> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.  
> Redistribution, commercial use, and modification are prohibited.

# Sparkade Game Development — Skill Index

You are building a game for the **Sparkade** platform.  
Sparkade is a mobile-first, TikTok-style scrollable game feed.  
Games run inside iframes, are written as a **single `.js` file**, and communicate with the platform via `postMessage`.

---

## How to use this index

1. Read this file in full.
2. Decide which skills you need using the tables below.
3. Fetch all chosen skills **simultaneously in one parallel batch** — do not fetch sequentially.
4. Write code only after all fetches are complete.

You will never need to fetch a genre skill to discover what sub-skills exist — every URL is listed here.

---

## Always fetch — every game

These three skills are required for every Sparkade game regardless of genre or complexity.

| Skill | URL |
|-------|-----|
| Phaser 3 Setup | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/phaser-setup.md |
| Phaser 3.90 Patterns | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/patterns.md |
| Sparkade Integration | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/sparkade-integration.md |

**phaser-setup.md** — canonical config, sprite quality standard, UI brightness rules, delta-time scoring, performance constraints.  
**patterns.md** — copy-paste Phaser 3.90 API patterns. Joystick, particles, scene management, audio, HUD, object pooling, hitstop, score animation. Do not implement any of these from memory.  
**sparkade-integration.md** — platform bridge protocol, save system, IAP. Fixed infrastructure, no creative decisions.

---

## Fetch if your game needs complex UI

**ui.md** covers panels, modals, upgrade cards, health bars, toast notifications, rarity badges, progress bars, and stat rows.

Fetch it when your game has **any** of the following:
- An upgrade or item selection screen
- A shop or IAP interface
- A boss HP bar or secondary progress bar
- Modal dialogs (confirmations, level complete, etc.)
- Toast notifications during gameplay

Skip it when your game only has a score counter, a health bar, and basic HUD text — those patterns are already in `patterns.md` (Patterns 11 and 12).

| Skill | URL |
|-------|-----|
| In-Game UI | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/ui.md |

---

## Fetch if your brief calls for exceptional polish

**polish.md** covers the design rationale behind juice, feel, and moment-to-moment satisfaction. The *implementation* of every polish technique (hitstop, score animation, floating text, hit flash, death sequence) is already in `patterns.md`. Fetch this skill only when the brief specifically calls for high visual polish or when you want the design reasoning behind those techniques.

| Skill | URL |
|-------|-----|
| Polish & Juice | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/polish.md |

---

## Genre skills — pick one, fetch its sub-skills directly

Pick the genre that best matches the brief. All sub-skill URLs are listed here — you do not need to fetch the genre SKILL.md first to discover them.

### Roguelite

Top-down arena combat with procedural rooms, enemy variety, and per-run upgrade choices. Runs last 3–6 minutes.

| Skill | URL | Fetch when |
|-------|-----|------------|
| Roguelite (genre overview) | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/SKILL.md | Always — for any roguelite |
| Run Structure | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/run-structure.md | Always — for any roguelite |
| Combat | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/combat.md | Always — for any roguelite |
| Upgrades | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/upgrades.md | Always — for any roguelite |
| Procedural Generation | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/procedural.md | Always — for any roguelite |

### Runner

Endless auto-scroller. Player character moves forward automatically. Tap to jump over obstacles, optionally duck under them. Runs last 60–180 seconds.

| Skill | URL | Fetch when |
|-------|-----|------------|
| Runner (genre overview) | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/SKILL.md | Always — for any runner |
| Scrolling & World | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/scrolling.md | Always — for any runner |
| Player Feel | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/player.md | Always — for any runner |
| Obstacles & Difficulty | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/obstacles.md | Always — for any runner |

---

## Worked examples

### Example A — Roguelite

Brief: *"A dark fantasy dungeon crawler where you fight skeletons and collect cursed upgrades."*

Skills to fetch (all simultaneously):
- `phaser-setup.md`
- `patterns.md`
- `sparkade-integration.md`
- `ui.md` ← needed for upgrade selection screen and boss HP bar
- `polish.md` ← brief implies high visual quality ("dark fantasy")
- `genres/roguelite/SKILL.md`
- `genres/roguelite/run-structure.md`
- `genres/roguelite/combat.md`
- `genres/roguelite/upgrades.md`
- `genres/roguelite/procedural.md`

Total: 10 fetches, issued in parallel, resolved before a single line of code is written.

---

### Example B — Runner

Brief: *"A neon city runner where you jump over obstacles and dodge cars."*

Skills to fetch (all simultaneously):
- `phaser-setup.md`
- `patterns.md`
- `sparkade-integration.md`
- `genres/runner/SKILL.md`
- `genres/runner/scrolling.md`
- `genres/runner/player.md`
- `genres/runner/obstacles.md`

Skip: `ui.md` (no upgrade screens, no modals — score and HUD are covered by `patterns.md`).  
Skip: `polish.md` (brief doesn't call for exceptional polish focus).

Total: 7 fetches, issued in parallel.

---

## Deliverable — one file only

Every game submission requires exactly one file:

### `game.js`
The complete Phaser 3 game. Single file, no imports, no placeholders, no TODOs.

---

## Platform constraints

- **Single file** — the entire game is one `.js` file. No imports, no bundler.
- **Mobile-first** — portrait orientation, 390×844 logical canvas. Touch is the primary input.
- **Phaser 3.90.0** — loaded from `https://assets.sparkade.games/lib/phaser/3.90.0/phaser.min.js`. Do not reference any other CDN.
- **No external assets** — all graphics must be drawn procedurally with Phaser Graphics or generated as textures in `BootScene`. No image URLs.
- **Score is a single integer** — submitted via `window.gameOver(score)` at run end.
- **Background colour** — always `#080808` to match the platform shell.
