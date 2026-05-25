> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# Sparkade Game Development — Skill Index

You are building a game for the **Sparkade** platform.

Sparkade is a mobile-first, portrait-orientation scrollable game feed. Games run inside an iframe, are delivered as a **single `.js` file**, and communicate with the platform through `window` functions injected by the shell.

These skills teach **how to integrate with the Sparkade platform and write correct Phaser code for its environment**. They do not prescribe a genre, art style, or game design — those are yours to decide based on the brief. Build whatever the brief asks for: a one-tap arcade game, a puzzle, a clicker, an endless runner, a shooter — anything that fits a single file and a phone screen.

---

## Platform constraints — these are mandatory

- **Single file** — the entire game is one `.js` file. No imports, no bundler, no external files.
- **Phaser v4** — loaded by the shell from `https://assets.sparkade.io/lib/phaser.min.js`, available as the global `Phaser`. The shell also loads the HiDPI plugin. Do not load either yourself.
- **Canvas** — 390×844 logical pixels, portrait. The shell scales it to fit the device.
- **HiDPI** — every pixel value must go through the `px()` helper. This is not optional; without it the game is blurry on every modern phone. See `hidpi.md`.
- **No external assets** — all textures are generated procedurally with `this.make.graphics()` + `generateTexture()`. No image URLs, no audio files.
- **Mobile-first** — touch is the primary input. On-screen controls must be visible elements, not invisible tap zones. Support keyboard as a secondary path for desktop testing.
- **Background** — `#080808`, to match the shell.
- **Score** — optional. Games with scoring submit a non-negative integer via `window.gameOver(score)`. Games without scoring call `window.gameOver()` with no argument.

---

## The skills

These five files are reference material. They are already available to you in full — read them as documentation, not as URLs to fetch.

**`hidpi.md`** — The mandatory HiDPI plugin. The `devicePixelRender()` initialisation, the `px()` helper, and the exact rule for what to wrap and what not to wrap. Read this first — every pixel value in the game depends on it.

**`phaser-setup.md`** — The Sparkade Phaser configuration, world bounds, scene organisation, the delta-time requirement, performance guidance, and which browser APIs are available in the iframe.

**`sparkade-integration.md`** — The complete platform bridge: every `window.*` function and callback the shell provides, what is mandatory vs optional, the save system, and IAP. This is the part that makes a game a *Sparkade* game.

**`patterns.md`** — Phaser v4 API patterns that are correct for this environment, with the specific failure modes they avoid (the particle API change, deferring `gameOver()` out of physics callbacks, physics group types, pooling, audio timing). Use these for correctness; the examples are illustrative, not a prescription of what to build.

**`ui.md`** — In-game UI built with Phaser Graphics/Text (no HTML/DOM). Technical correctness rules for readable, tappable, correctly-layered UI. Colours, fonts, and layout are creative choices; the technical rules are not.

---

## Deliverable

One file: **`game.js`** — a complete, runnable Phaser v4 game. Single file, no imports, no placeholders, no TODOs.