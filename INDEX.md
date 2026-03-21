> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.  
> Redistribution, commercial use, and modification are prohibited.

# Sparkade Game Development — Skill Index

You are building a game for the **Sparkade** platform.  
Sparkade is a mobile-first, TikTok-style scrollable game feed.  
Games run inside iframes, are written as a **single `.js` file**, and communicate with the platform via `postMessage`.

---

## Step 1 — Always fetch these two core skills first

| Skill | URL |
|-------|-----|
| Phaser 3 Setup | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/phaser-setup.md |
| Sparkade Integration | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/sparkade-integration.md |

Read both in full before writing any code. They contain the canonical boilerplate, the platform message protocol, mobile canvas sizing, and touch input setup. Do not skip them.

---

## Step 2 — Fetch the genre skill

Pick the genre that best matches the game brief and fetch that skill only. Do not fetch multiple genre skills unless explicitly instructed.

| Genre | Description | Skill URL |
|-------|-------------|-----------|
| Roguelite | Top-down arena combat with procedural rooms, enemy variety, and per-run upgrade choices. Runs last 3–6 minutes. | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/SKILL.md |

The genre skill will instruct you to fetch additional sub-skills. Follow those instructions.

---

## Step 3 — Optionally fetch the polish skill

Fetch this after the genre skill if the game brief calls for high visual polish:

| Skill | URL |
|-------|-----|
| Polish & Juice | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/polish.md |

---

## Deliverables — always produce both files

Every game submission requires exactly two files:

### 1. `game.js`
The complete Phaser 3 game. Single file, no imports, no placeholders, no TODOs.

### 2. `manifest.json`
Metadata consumed by the Sparkade platform. Always produce this alongside `game.js`.

```json
{
  "slug":        "my-game-slug",
  "title":       "My Game Title",
  "category":    "Roguelite",
  "description": "A one to two sentence description of the game. Write this as a player-facing blurb.",
  "controls":    "Left side: move  ·  Right side: shoot",
  "version":     "1.0.0",
  "is_active":   true
}
```

**Field rules:**
- `slug` — lowercase, hyphens only, no spaces. Derived from the title. e.g. `"deep-sea-diver"`
- `title` — title case, max ~24 characters
- `category` — must match the genre exactly e.g. `"Roguelite"`, `"Endless Runner"`, `"Platformer"`
- `description` — 1–2 sentences, player-facing, no spoilers about mechanics
- `controls` — single line, ultra-concise. Format: `"Left: move  ·  Right: shoot"`
- `version` — always `"1.0.0"` for new games
- `is_active` — always `true`

---

## Platform constraints (memorise these)

- **Single file** — the entire game is one `.js` file. No imports, no bundler.
- **Mobile-first** — portrait orientation, 390×844 logical canvas. Touch is the primary input.
- **Phaser 3.90.0** — loaded from `https://assets.sparkade.games/lib/phaser/3.90.0/phaser.min.js`. Do not reference any other CDN.
- **No external assets** — all graphics must be drawn procedurally with Phaser Graphics or generated as textures in `preload`. No image URLs.
- **Score is a single integer** — submitted via `window.gameOver(score)` at run end.
- **Background colour** — always `#080808` to match the platform shell.
