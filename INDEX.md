> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.  
> Redistribution, commercial use, and modification are prohibited.

# Sparkade Game Development — Skill Index

You are building a game for the **Sparkade** platform.  
Sparkade is a mobile-first, TikTok-style scrollable game feed.  
Games run inside iframes, are written as a **single `.js` file**, and communicate with the platform via `postMessage`.

---

## Step 1 — Always fetch these core skills first

Fetch all four. Read them in this order before writing any code.

| Skill | URL |
|-------|-----|
| Phaser 3 Setup | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/phaser-setup.md |
| Phaser 3.90 Patterns | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/patterns.md |
| Sparkade Integration | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/sparkade-integration.md |
| In-Game UI | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/ui.md |

**phaser-setup.md** — canonical config, sprite quality standard, UI brightness rules, delta-time scoring, performance constraints.  
**patterns.md** — copy-paste Phaser 3.90 API patterns for every common implementation. Joystick, particles, scene management, audio, HUD, object pooling, hitstop, score animation. Do not implement any of these from memory — use the patterns.  
**sparkade-integration.md** — platform bridge protocol, save system, IAP. Fixed infrastructure, no creative decisions.  
**ui.md** — every in-game UI pattern: panels, buttons, modals, health bars, toast notifications, pause menu, exit button, touch target sizing. Use these patterns for all UI — do not invent UI from scratch.

---

## Step 2 — Fetch the genre skill

Pick the genre that best matches the game brief and fetch that skill only. Do not fetch multiple genre skills unless explicitly instructed.

| Genre | Description | Skill URL |
|-------|-------------|-----------|
| Roguelite | Top-down arena combat with procedural rooms, enemy variety, and per-run upgrade choices. Runs last 3–6 minutes. | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/SKILL.md |
| Runner | Endless auto-scroller. Player character moves forward automatically. Tap to jump over obstacles, optionally duck under them. Runs last 60–180 seconds. | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/SKILL.md |

The genre skill will instruct you to fetch additional sub-skills. Follow those instructions.

---

## Step 3 — Fetch additional skills if requested by the genre skill

The genre skill will instruct you to fetch further skills when relevant. Follow those instructions.

| Skill | URL | When to fetch |
|-------|-----|---------------|
| Polish & Juice | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/polish.md | When genre skill requests it, or when the brief specifically calls for high visual polish. Note: the core implementation patterns for polish (hitstop, score animation, floating text, hit flash, death sequence) are already in `patterns.md`. This skill provides the design rationale. |

---

## Deliverable — one file only

Every game submission requires exactly one file:

### `game.js`
The complete Phaser 3 game. Single file, no imports, no placeholders, no TODOs. No other files are required or expected.

---

## Platform constraints (memorise these)

- **Single file** — the entire game is one `.js` file. No imports, no bundler.
- **Mobile-first** — portrait orientation, 390×844 logical canvas. Touch is the primary input.
- **Phaser 3.90.0** — loaded from `https://assets.sparkade.games/lib/phaser/3.90.0/phaser.min.js`. Do not reference any other CDN.
- **No external assets** — all graphics must be drawn procedurally with Phaser Graphics or generated as textures in `preload`. No image URLs.
- **Score is a single integer** — submitted via `window.gameOver(score)` at run end.
- **Background colour** — always `#080808` to match the platform shell.
