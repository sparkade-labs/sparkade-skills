# Phaser 3 Setup — Sparkade Core Skill

Canonical Phaser 3.90.0 configuration and scene structure for Sparkade games.

---

## Game Config

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
  physics: {
    default: 'arcade',
    arcade: { gravity: { y: 0 }, debug: false },
  },
  scene: [BootScene, MenuScene, GameScene, GameOverScene],
};

new Phaser.Game(config);
```

- Always `390×844` — this is the Sparkade portrait canvas standard.
- `Phaser.Scale.FIT` + `CENTER_BOTH` handles all device sizes correctly.
- Use `gravity: { y: 0 }` (top-down default). Override per-scene if needed.
- Scene order matters — Boot first, then Menu, Game, GameOver.

---

## Scene Structure

Every game uses exactly four scenes:

### BootScene
Generates all procedural textures. Never skipped.

```js
class BootScene extends Phaser.Scene {
  constructor() { super('Boot'); }

  create() {
    // Generate all textures here
    this._makeTexture('player', 20, 20, (g) => {
      g.fillStyle(0xF55018);
      g.fillCircle(10, 10, 10);
    });
    // ... more textures

    this.scene.start('Menu');
  }

  _makeTexture(key, w, h, drawFn) {
    const g = this.make.graphics({ x: 0, y: 0, add: false });
    drawFn(g);
    g.generateTexture(key, w, h);
    g.destroy();
  }
}
```

### MenuScene
Shows game title, instructions, and a "TAP TO START" prompt.  
Calls `window.gameStarted()` when the player starts a run — NOT before.

### GameScene
The main game loop. Calls `window.gameOver(score)` when the run ends.

### GameOverScene
Displays run summary (score, depth reached, stats).  
Shows sparks earned and personal best via `window.onScoreResult`.  
Has a "PLAY AGAIN" button that restarts GameScene, and an "EXIT" button that calls `window.gameExit()`.

---

## Scene Lifecycle Rules

- Use `create()` for setup, never `constructor()`.
- Use `this.scene.start('X')` to transition — it destroys the current scene cleanly.
- Pass data between scenes via the second argument: `this.scene.start('GameOver', { score, depth })`.
- Receive it in the next scene: `create(data) { this.score = data.score; }`.
- Never keep references across scene transitions — they will be stale.

---

## Update Loop

```js
update(time, delta) {
  // delta is milliseconds since last frame
  // Use delta for all movement: velocity is pixels/second
  // e.g. this.player.x += speed * (delta / 1000);
  // Phaser arcade physics handles this automatically via setVelocity
}
```

Always use `delta`-based movement or Phaser physics velocities. Never hardcode per-frame pixel values.

---

## Procedural Texture Patterns

All visual assets are generated in BootScene. Common patterns:

```js
// Rounded rectangle
g.fillStyle(0x334155);
g.fillRoundedRect(0, 0, w, h, 6);
g.strokeRoundedRect(0, 0, w, h, 6);

// Circle with outline
g.fillStyle(0xF55018);
g.fillCircle(cx, cy, r);
g.lineStyle(2, 0xffffff, 0.4);
g.strokeCircle(cx, cy, r);

// Diamond (enemy shape)
g.fillStyle(0x7c3aed);
g.fillTriangle(cx, 0, w, cy, cx, h);
g.fillTriangle(cx, 0, 0, cy, cx, h);
```

---

## Performance Rules

- Pool bullets with `this.physics.add.group({ maxSize: 50, runChildUpdate: false })`.
- Pool particles — use Phaser's built-in `ParticleEmitter`, not custom objects.
- Destroy off-screen objects: check `!this.cameras.main.worldView.contains(obj.x, obj.y)`.
- Keep active enemy count ≤ 15 on screen at once.
- Never create new objects in `update()` — use pools.
