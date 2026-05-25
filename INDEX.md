> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# Sparkade Game Development — Skill Index

You are building a game for the **Sparkade** platform.

Sparkade is a mobile-first, portrait-orientation scrollable game feed. Games run inside iframes, are delivered as a **single `.js` file**, and communicate with the platform through `window` functions injected by the shell.

---

## Platform constraints — read these first

- **Single file** — the entire game is one `.js` file. No imports, no bundler, no external files.
- **Phaser v4** — loaded by the shell from `https://assets.sparkade.io/lib/phaser.min.js`. Available as the global `Phaser`. Do not load it yourself.
- **Canvas dimensions** — 390×844 logical pixels, portrait orientation.
- **No external assets** — all textures must be generated procedurally in `BootScene` using `this.make.graphics()` + `generateTexture()`. No image URLs.
- **Mobile-first** — touch is the primary input. On-screen controls must be visible UI elements, not invisible tap zones.
- **Background** — always `#080808` to match the shell.
- **Score** — a non-negative integer submitted via `window.gameOver(score)`.

---

## Skills — fetch all that apply

### Always fetch

These four are required for every Sparkade game.

| Skill | URL |
|-------|-----|
| HiDPI | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/hidpi.md |
| Phaser v4 Setup | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/phaser-setup.md |
| Phaser v4 Patterns | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/patterns.md |
| Sparkade Integration | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/sparkade-integration.md |

**hidpi.md** — mandatory. The `phaser-hidpi` plugin fixes blurry rendering on all modern phones. Covers the `devicePixelRender()` initialisation, the `px()` helper function, what to wrap and what not to wrap. Every pixel value in the game must use `px()`.

**phaser-setup.md** — canonical Phaser config for Sparkade, world bounds, scene architecture, delta-time requirement, performance constraints, available browser APIs.

**patterns.md** — copy-paste Phaser v4 patterns for every common implementation. Covers: game config, depth layers, texture generation, scene data passing, joystick, action button, particles, physics groups, group iteration, gameOver safe call, hit flash, HUD text, timer events, selection overlay, audio, object pooling, world boundary, floating text, scene lifecycle, hitstop, score counter tween. Do not implement these from memory.

**sparkade-integration.md** — the complete platform bridge. Every `window.*` function and callback the shell provides, what is mandatory vs optional, the save system, IAP, and lifecycle.

### Fetch if your game needs complex UI

| Skill | URL |
|-------|-----|
| In-Game UI | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/ui.md |

Fetch when the game has an upgrade/selection screen, shop UI, modal dialogs, a boss HP bar, or toast notifications. Skip for games that only need a score counter and basic HUD — those patterns are in `patterns.md`.

---

## Fetch order

Fetch all chosen skills **simultaneously in one parallel batch**. Do not fetch sequentially. Write code only after all fetches are complete.

---

## Deliverable

One file: **`game.js`** — the complete Phaser v4 game. Single file, no imports, no placeholders, no TODOs.
