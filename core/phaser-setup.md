# Phaser 3 Setup — Sparkade Core Skill

Standards and principles for all Sparkade games. These are not suggestions — they are the minimum bar for a shippable game.

---

## Configuration Requirements

Every game must use these exact settings. They are non-negotiable.

```js
const W = 390, H = 844;

const config = {
  type: Phaser.AUTO,
  width: W,
  height: H,
  backgroundColor: '#080808',
  parent: 'game-container',
  scale: {
    mode: Phaser.Scale.FIT,
    autoCenter: Phaser.Scale.CENTER_BOTH,
  },
  render: {
    pixelArt: false,
    antialias: true,
    antialiasGL: true,
    roundPixels: false,
  },
  physics: {
    default: 'arcade',
    arcade: { gravity: { y: 0 }, debug: false },
  },
  scene: [BootScene, MenuScene, GameScene, GameOverScene],
};

const game = new Phaser.Game(config);

// HIGH-DPI FIX — mandatory. Without this, every sprite is blurry on modern phones.
game.events.once('ready', () => {
  const canvas = game.canvas;
  const dpr    = window.devicePixelRatio || 1;
  const w      = canvas.clientWidth  || W;
  const h      = canvas.clientHeight || H;
  canvas.width  = w * dpr;
  canvas.height = h * dpr;
  const ctx = canvas.getContext('2d');
  if (ctx) ctx.scale(dpr, dpr);
});
```

**Why `antialias: true`:** Sparkade games are rendered on high-DPI screens. Pixel art mode produces jagged, amateur-looking sprites. Smooth rendering is always correct here.

**Why the DPR fix:** Phaser renders at logical resolution by default. On a phone with `devicePixelRatio: 3`, every texture is rendered at 1/3 the native resolution and upscaled — producing visible blur. This fix forces native resolution rendering.

---

## World Bounds — Always Enforce

Gameplay must never leave the canvas. Enemies, players, and projectiles that escape the screen break the game instantly.

```js
// In GameScene.create()
this.physics.world.setBounds(0, 0, W, H);
```

Every physics body that should stay on screen must have `setCollideWorldBounds(true)`. The player always does. Enemies should. Projectiles are destroyed at range limits instead.

Draw a visible boundary so players perceive the edges before hitting them. The style is your creative choice — it should fit the game's visual theme.

---

## Scene Architecture

Four scenes. Always the same four scenes.

- **BootScene** — generates all procedural textures. No gameplay. Transitions immediately to Menu.
- **MenuScene** — title, brief instructions, start prompt. Calls `window.gameStarted()` when the player commits to starting.
- **GameScene** — all gameplay. Calls `window.gameOver(score)` when the run ends.
- **GameOverScene** — run summary, sparks earned, play again / exit.

Pass data between scenes via the second argument to `scene.start()`. Never use global variables for scene state.

---

## Sprite Quality Standard

This is where most AI-generated games fail. A single flat shape is not a sprite — it is a placeholder. Every character, enemy, projectile, and significant pickup must be composed of multiple layered shapes.

**The minimum layers for any character sprite:**
1. A soft shadow or glow underneath (low alpha, slightly offset)
2. The main body shape
3. A highlight region (lighter tone of the main colour, top-left biased)
4. A small specular dot (near-white, very small, inside the highlight)
5. A dark outline (low alpha, stroked around the main body)

**Why this matters:** These five layers are what make a shape read as a 3D object under light. Without them, sprites look flat and amateur regardless of how good the game mechanics are. With them, even simple circles and rectangles look polished and intentional.

Generate all textures in `BootScene` using `this.make.graphics()` + `generateTexture()`. Design each sprite to read clearly at its actual game size — if it's a 28×28 enemy, design at 28×28, not larger.

Each enemy type must be visually distinct at a glance. If a player can't tell enemy types apart in the first 10 seconds, the designs have failed. Use different shapes, not just different colours.

---

## UI Brightness — A Common Failure Mode

Dark, muddy UI is the most frequent quality issue in AI-generated Sparkade games. The cause is usually inherited dark colour values from the `#080808` game background being used for UI text too.

**The rules:**
- Primary text (score, titles, button labels) — always bright. `#ffffff` or near-white.
- Secondary text (labels, descriptors) — never dimmer than `#888888`.
- UI panel backgrounds — noticeably lighter than `#080808`. Use `#111111` to `#1a1a1a` so panels have contrast against the game background.
- Borders on UI elements — always visible. A border at `rgba(255,255,255,0.08)` is effectively invisible. Minimum `0.15` opacity.
- Floating text over gameplay — always use `stroke` + `strokeThickness` so it reads against any background colour.
- The primary action button is always the most visually prominent element on the screen.

---

## Performance Standards

- Use object pools for bullets and frequently-spawned objects. Never `new` an object in `update()`.
- Use Phaser's built-in `ParticleEmitter` for particles.
- Cap active enemy count at 15.
- Bullets despawn at range limits, not by off-screen check.
- Always use `delta`-based movement or Phaser physics velocities. Never hardcode per-frame pixel values.

---

## Available Browser APIs

The game runs inside an iframe on the Sparkade platform. The following browser APIs are fully available and confirmed working — do not hesitate to use them:

**Web Audio API** — use `window.AudioContext` (or `window.webkitAudioContext`) for all sound effects. No libraries needed. Generate everything procedurally with oscillators, filters, and gain nodes.

**One mandatory rule:** never instantiate `AudioContext` on page load. Browsers suspend audio contexts created before a user interaction. Create it on the first `pointerdown` or equivalent input event — Phaser's input system satisfies this naturally.

```js
// Correct — create on first interaction
this.input.once('pointerdown', () => {
  this._audioCtx = new (window.AudioContext || window.webkitAudioContext)();
});
```

All other standard browser APIs (Canvas 2D, requestAnimationFrame, localStorage, etc.) are equally available. The iframe `allow="autoplay"` attribute is already set by the platform shell.
