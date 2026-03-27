> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# Scrolling & World — Runner Sub-Skill

The scrolling system is the engine everything else runs on. Get this wrong and the entire game feels broken, regardless of how good the obstacles and player feel are.

---

## Core Principle: The World Moves, Not the Player

The player character stays at a fixed X position — typically 15–20% from the left edge of the screen. Everything else — ground, obstacles, background layers, collectibles — moves rightward to leftward at the current scroll speed.

**Never move the player rightward to simulate forward movement.** Doing so breaks obstacle positioning, makes collision detection unreliable, and means the player drifts off camera. The player's X is fixed. The world's X changes.

```js
// In create():
this.PLAYER_X = W * 0.18;      // fixed — never changes during a run
this.player.setX(this.PLAYER_X);

// In update(time, delta):
// Move everything else — not the player
const move = this.scrollSpeed * (delta / 1000);
this.groundChunks.getChildren().forEach(chunk => chunk.x -= move);
this.obstacleGroup.getChildren().forEach(obs => {
  if (obs.active) obs.x -= move;
});
// Background layers move at fractional speeds — see Parallax section
```

---

## Scroll Speed — Ramp Pattern

Speed must increase continuously. Use a base speed that increases with time elapsed, capped at a maximum. Never increase speed in sudden jumps — always increment smoothly each frame.

```js
// In create():
this.scrollSpeed    = 220;       // starting speed in px/sec — approachable
this.SPEED_MAX      = 680;       // ceiling — above this, human reaction fails
this.SPEED_RAMP     = 18;        // px/sec added per second of survival
this._elapsed       = 0;         // total seconds survived

// In update(time, delta):
const dt = delta / 1000;         // delta to seconds
this._elapsed += dt;

// Smooth continuous ramp — never jump
this.scrollSpeed = Math.min(
  this.SPEED_MAX,
  220 + this._elapsed * this.SPEED_RAMP
);

// Display speed tier in HUD if it helps player awareness (optional)
// e.g. "FAST" / "FASTER" / "MAX" labels at thresholds
```

**Calibration guide:**
- 0–15s: 220–490 px/sec — tutorial pace, most players survive
- 15–45s: 490–580 px/sec — comfortable challenge
- 45s+: 580–680 px/sec — survival zone, filters skilled players
- 680 px/sec cap: one obstacle width (~60px) passes in ~90ms — human reaction floor

---

## Ground — Chunk System

Never use a single long ground sprite. Build the ground from recycled chunks that tile seamlessly and wrap from right to left. This handles infinite scrolling without ever running out of ground.

```js
// In create():
_buildGround() {
  this.GROUND_Y    = H - 120;     // Y position of ground top surface
  this.GROUND_H    = 120;         // visual height below ground top
  this.CHUNK_W     = 200;         // width of one ground chunk
  this.groundChunks = this.physics.add.staticGroup();

  // Fill screen width + one extra chunk for seamless wrap
  const count = Math.ceil(W / this.CHUNK_W) + 2;
  for (let i = 0; i < count; i++) {
    const chunk = this._makeGroundChunk(i * this.CHUNK_W, this.GROUND_Y);
    this.groundChunks.add(chunk);
  }
}

_makeGroundChunk(x, y) {
  // Draw procedurally to match your game's theme
  // This is an example — adapt colours, patterns, and decoration to your world
  const g = this.make.graphics({ add: false });
  // Ground surface
  g.fillStyle(0x4a7c59, 1);
  g.fillRect(0, 0, this.CHUNK_W, this.GROUND_H);
  // Top edge — slightly lighter for definition
  g.fillStyle(0x6aaa76, 1);
  g.fillRect(0, 0, this.CHUNK_W, 8);
  // Optional: surface detail marks (cracks, tiles, panels — fit your theme)
  g.generateTexture('ground_chunk', this.CHUNK_W, this.GROUND_H);
  g.destroy();

  return this.physics.add.staticImage(x + this.CHUNK_W / 2, y + this.GROUND_H / 2, 'ground_chunk');
}

// In update():
_scrollGround(move) {
  this.groundChunks.getChildren().forEach(chunk => {
    chunk.x -= move;
    // When a chunk exits the left edge, move it to the right
    if (chunk.x + this.CHUNK_W / 2 < -this.CHUNK_W) {
      // Find the rightmost chunk's X and place this one after it
      const maxX = Math.max(...this.groundChunks.getChildren().map(c => c.x));
      chunk.x = maxX + this.CHUNK_W;
      chunk.refreshBody();    // ← mandatory for staticGroup bodies after repositioning
    }
  });
}
```

**Critical:** call `chunk.refreshBody()` after repositioning any static physics body. Without it the physics body stays at the old position and the player falls through the ground.

---

## Parallax Background — 2–3 Layers

Parallax gives the world depth without cluttering the gameplay plane. Each layer moves slower than the scroll speed — the further back, the slower it moves.

```js
// In create():
_buildParallax() {
  // Layer speeds as a fraction of scroll speed
  // Far (sky features, mountains): 0.1–0.2×
  // Mid (distant structures, trees): 0.3–0.45×
  // Near (close details, just behind ground): 0.6–0.75×

  this._parallaxLayers = [
    this._makeParallaxLayer('bg_far',  H * 0.15, 0.15),
    this._makeParallaxLayer('bg_mid',  H * 0.45, 0.4),
    this._makeParallaxLayer('bg_near', H * 0.72, 0.65),
  ];
}

_makeParallaxLayer(key, y, speedFactor) {
  // Generate texture in BootScene — themed to your world
  // Two copies side by side for seamless wrapping
  const imgA = this.add.image(W / 2,         y, key).setDepth(DEPTH.BG_DETAIL).setScrollFactor(0);
  const imgB = this.add.image(W / 2 + W,     y, key).setDepth(DEPTH.BG_DETAIL).setScrollFactor(0);
  return { imgA, imgB, speedFactor };
}

// In update():
_scrollParallax(dt) {
  this._parallaxLayers.forEach(layer => {
    const move = this.scrollSpeed * layer.speedFactor * dt;
    layer.imgA.x -= move;
    layer.imgB.x -= move;
    // Wrap: when A exits left, jump it to the right of B
    if (layer.imgA.x + W / 2 < 0) { layer.imgA.x = layer.imgB.x + W; }
    if (layer.imgB.x + W / 2 < 0) { layer.imgB.x = layer.imgA.x + W; }
  });
}
```

---

## Ground Variation — Visual Interest Without Gameplay Impact

Purely cosmetic variation keeps the world feeling alive. These never affect collision — they are background decoration only.

Ideas (choose what fits your theme):
- **Surface details** — cracks, tiles, panels, footprints drawn on the ground surface
- **Foreground props** — small rocks, flowers, debris scrolling in the foreground layer faster than scroll speed (1.1–1.3× for a close parallax effect)
- **Sky events** — clouds, birds, distant objects at very low speed in the far background layer

Never attach gameplay meaning to visual variation. A crack in the ground that looks like a gap but isn't will enrage players. If it looks dangerous, it must be dangerous.

---

## Score — Distance-Based

Score directly from scroll distance. This is honest — the player is rewarded for surviving, not for a metric they can't observe.

```js
// In create():
this.score      = 0;
this._distPx    = 0;       // raw pixel distance scrolled
this.PX_PER_PT  = 5;       // every 5 pixels = 1 point (adjust to taste)

// In update():
_updateScore(move) {
  this._distPx += move;
  const newScore = Math.floor(this._distPx / this.PX_PER_PT);
  if (newScore !== this.score) {
    this.score = newScore;
    this._updateScoreHUD();
  }
}
```

Apply bonus multipliers for performance (see `obstacles.md`) on top of this base.

---

## Run End — Freeze Before Transition

When the player dies, freeze the scroll immediately. Do not let the world keep moving while the death animation plays — it looks wrong and confuses players about what killed them.

```js
_onDeath() {
  this._dead = true;
  this.scrollSpeed = 0;        // freeze world instantly
  // Run death sequence (from patterns.md Pattern 10)
  // Then transition to GameOverScene with score
}
```
