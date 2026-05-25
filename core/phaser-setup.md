# Phaser v4 Setup — Sparkade Core Skill

Platform requirements for every Sparkade game. These are not suggestions — they are correctness requirements imposed by the platform environment.

---

## Configuration

Every game must use this exact initialisation pattern. It uses the `phaser-hidpi` plugin — see `hidpi.md` for the full reference.

```js
const LOGICAL_W = 390;
const LOGICAL_H = 844;

const { game: hidpiConfig, px } = devicePixelRender({ width: LOGICAL_W, height: LOGICAL_H });

// W and H are physical pixels — use these for all layout
const W = px(LOGICAL_W);
const H = px(LOGICAL_H);

new Phaser.Game(Object.assign(hidpiConfig, {
  type:            Phaser.AUTO,
  backgroundColor: '#080808',
  parent:          'game-container',
  scale: {
    mode:       Phaser.Scale.FIT,
    autoCenter: Phaser.Scale.CENTER_BOTH,
  },
  render: {
    pixelArt:    false,
    antialias:   true,
    antialiasGL: true,
    roundPixels: false,
  },
  physics: {
    default: 'arcade',
    arcade:  { gravity: { y: 0 }, debug: false },
  },
  scene: [BootScene, MenuScene, GameScene, GameOverScene],
}));
```

**`parent: 'game-container'`** — the shell's HTML has `<div id="game-container">`. Phaser must mount here.

**`backgroundColor: '#080808'`** — must match the platform shell background.

**`devicePixelRender` is a global** provided by the shell alongside Phaser. Do not set `resolution:` manually — the plugin handles it. Do not load either script yourself.

**`W` and `H` are physical pixels.** After `px()` multiplies by `devicePixelRatio`, `W` and `H` are no longer 390 and 844 on HiDPI devices. All layout must use these constants. All inline pixel values must be wrapped with `px()`. See `hidpi.md`.

---

## World Bounds

```js
// In GameScene.create():
this.physics.world.setBounds(0, 0, W, H);
```

Every physics body that must stay on screen needs `setCollideWorldBounds(true)`. Short-lived objects (projectiles) should be despawned at range limits instead.

---

## Scene Architecture

Four scenes. This structure is the standard — deviating from it produces lifecycle bugs with the platform bridge calls.

- **BootScene** — generates all procedural textures. No gameplay. Transitions immediately to MenuScene.
- **MenuScene** — title and start prompt. Calls `window.gameStarted()` when the player commits to starting.
- **GameScene** — all gameplay. Calls `window.gameOver(score)` when the run ends.
- **GameOverScene** — run summary. Has "play again" and "exit" actions.

Pass data between scenes via the second argument to `scene.start()`. Do not use global variables for scene state.

---

## Delta-Time — Mandatory for All Time-Based Logic

Never increment score or change game state directly in `update()` without using `delta`. `update()` runs every frame — at 120 fps it fires twice as often as at 60 fps. Any logic not gated through `delta` runs at different rates on different devices.

```js
// In create():
this._scoreTick = 0;

// In update(time, delta):
this._scoreTick += delta;           // delta = ms since last frame
if (this._scoreTick >= 1000) {
  this._scoreTick -= 1000;          // subtract, don't reset — absorbs leftover ms
  this.score += 1;
}
```

The same rule applies to speed ramps, spawn timers, difficulty scaling — anything that changes over time must use `delta`, not frame count.

---

## Performance Constraints

- Cap active entity count at 15.
- Use object pools for projectiles and frequently-spawned objects. Never `new` inside `update()`.
- Use Phaser's built-in `ParticleEmitter` for particles.
- Use `delta`-based movement or Phaser physics velocities. Never hardcode per-frame pixel values.
- Short-lived objects: despawn at range limit, not by off-screen check.

---

## Available Browser APIs

The game runs inside an iframe on the Sparkade platform. These are confirmed available:

**Web Audio API** — use `window.AudioContext` (or `window.webkitAudioContext`). Generate sounds procedurally. Never instantiate `AudioContext` before a user interaction — browsers suspend early contexts. Create it on the first `pointerdown`.

```js
this.input.once('pointerdown', () => {
  this._audioCtx = new (window.AudioContext || window.webkitAudioContext)();
});
```

`localStorage`, `Canvas 2D`, and standard Web APIs are available. The iframe has `allow="autoplay"` set by the platform.
