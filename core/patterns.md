> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# Phaser v4 Patterns — Sparkade Core Skill

These are copy-paste patterns for the Phaser v4 APIs used in every Sparkade game. Each pattern solves a specific, documented failure mode. Do not deviate from these — every variation listed here has been tested to fail in the Sparkade iframe environment.

Read this file after `phaser-setup.md` and `hidpi.md`.

**`px()` is always in scope.** All pixel measurements in these patterns — coordinates, sizes, radii, font sizes, line widths, strokeThickness, physics velocities, tween pixel targets — must be wrapped with `px()`. `W` and `H` are already pre-multiplied and must not be wrapped. See `hidpi.md` for the complete rule. The patterns below omit `px()` wrapping for readability; apply it to every pixel value when implementing.

---

## 1. Game Configuration — HiDPI-Correct Setup

Use `devicePixelRender` (global provided by the shell) for correct HiDPI rendering. Do not set `resolution:` manually.

```js
const LOGICAL_W = 390;
const LOGICAL_H = 844;

const { game: hidpiConfig, px } = devicePixelRender({ width: LOGICAL_W, height: LOGICAL_H });

const W = px(LOGICAL_W);   // physical pixel width — use for all layout
const H = px(LOGICAL_H);   // physical pixel height — use for all layout

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

---

## 2. Depth Layer Constants — Always Define These First

Z-order bugs are invisible until they ruin a playtest. Define constants at the top of your file and assign every object to a layer. Never use raw numbers in `setDepth()` calls.

```js
const DEPTH = {
  BG:         0,
  BG_DETAIL:  1,
  PICKUPS:    5,
  ENTITIES:   10,   // enemies, obstacles, NPCs — any game object the player interacts with
  PLAYER:     20,
  PROJECTILES:30,   // bullets, thrown objects, any short-lived physics object
  FX:         40,   // particles, hit flashes
  HUD_BG:     90,   // semi-transparent HUD backing strips
  HUD:        100,  // score, timer, primary counter
  OVERLAY:    200,  // selection screens, pause, game over panels
};
```

---

## 3. Texture Generation in BootScene

All graphics are procedural — never load external URLs. Generate every texture in `BootScene.create()` using this exact pattern. The `generateTexture()` call converts the graphics object into a named texture key that all other scenes can use.

```js
class BootScene extends Phaser.Scene {
  constructor() { super('BootScene'); }

  create() {
    // Always generate a 1×1 white pixel — required for particles
    const px = this.make.graphics({ add: false });
    px.fillStyle(0xffffff, 1);
    px.fillRect(0, 0, 1, 1);
    px.generateTexture('pixel', 1, 1);
    px.destroy();

    // Example: player sprite (28×28)
    const player = this.make.graphics({ add: false });
    // Shadow / glow layer
    player.fillStyle(0xF55018, 0.18);
    player.fillCircle(14, 16, 13);
    // Main body
    player.fillStyle(0xF55018, 1);
    player.fillCircle(14, 13, 11);
    // Highlight
    player.fillStyle(0xff8c66, 0.6);
    player.fillCircle(11, 10, 5);
    // Specular dot
    player.fillStyle(0xffffff, 0.8);
    player.fillCircle(10, 9, 2);
    // Outline
    player.lineStyle(1.5, 0x000000, 0.25);
    player.strokeCircle(14, 13, 11);
    player.generateTexture('player', 28, 28);
    player.destroy();

    // Always transition immediately — BootScene has no gameplay
    this.scene.start('MenuScene');
  }
}
```

**Rule:** every character, enemy, projectile, and pickup must be generated here. Never generate textures in GameScene — texture creation is expensive and causes frame drops.

---

## 4. Scene Data Passing — init() not create()

Data passed to `scene.start('Name', data)` arrives in `init(data)`, not `create()`. This is the most common wiring bug in multi-scene games — if your run state is `undefined` in room two, this is why.

```js
class GameScene extends Phaser.Scene {
  constructor() { super('GameScene'); }

  // init() runs before preload() and create() — this is where scene data arrives
  init(data) {
    this.runState = data.runState || {
      depth:    1,
      floor:    1,
      score:    0,
      hp:       100,
      maxHp:    100,
      kills:    0,
      upgrades: [],
      tookDamageThisRoom: false,
    };
  }

  create() {
    // this.runState is fully populated here
  }
}

// Passing state to the next room:
this.scene.start('GameScene', { runState: this.runState });
```

---

## 5. Fixed Joystick — Complete Implementation

The correct pattern for games requiring directional movement input on mobile. Fixed position builds muscle memory. Copy this verbatim — every variation produces subtle input bugs.

```js
// In GameScene.create():
_createJoystick() {
  const jx = 90, jy = H - 110;   // fixed bottom-left position
  const DEAD = 12, MAX = 45;      // dead zone radius, max displacement

  // Base ring
  this._joyBase = this.add.circle(jx, jy, MAX, 0xffffff, 0.08)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0);
  this.add.circle(jx, jy, MAX, 0x000000, 0)
    .setStrokeStyle(1.5, 0xffffff, 0.2)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0);

  // Thumb nub
  this._joyNub = this.add.circle(jx, jy, 18, 0xffffff, 0.35)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0);

  // State
  this._joyActive  = false;
  this._joyPointerId = null;
  this._joyVec     = { x: 0, y: 0 };   // normalised -1 to 1

  this.input.on('pointerdown', (p) => {
    if (this._joyPointerId !== null) return;  // one joystick pointer only
    if (p.x > W / 2) return;                 // right half is fire button
    this._joyActive    = true;
    this._joyPointerId = p.id;
  });

  this.input.on('pointermove', (p) => {
    if (!this._joyActive || p.id !== this._joyPointerId) return;
    const dx = p.x - jx, dy = p.y - jy;
    const dist = Math.sqrt(dx * dx + dy * dy);
    if (dist < DEAD) {
      this._joyVec.x = 0;
      this._joyVec.y = 0;
      this._joyNub.setPosition(jx, jy);
      return;
    }
    const clamped = Math.min(dist, MAX);
    const angle   = Math.atan2(dy, dx);
    this._joyVec.x = Math.cos(angle);
    this._joyVec.y = Math.sin(angle);
    this._joyNub.setPosition(
      jx + Math.cos(angle) * clamped,
      jy + Math.sin(angle) * clamped,
    );
  });

  this.input.on('pointerup', (p) => {
    if (p.id !== this._joyPointerId) return;
    this._joyActive    = false;
    this._joyPointerId = null;
    this._joyVec.x = 0;
    this._joyVec.y = 0;
    this._joyNub.setPosition(jx, jy);
  });
}

// In update():
_applyJoystick(speed) {
  if (this._joyVec.x !== 0 || this._joyVec.y !== 0) {
    this.player.body.setVelocity(
      this._joyVec.x * speed,
      this._joyVec.y * speed,
    );
  } else {
    this.player.body.setVelocity(0, 0);
  }
}

// Keyboard fallback (always include for desktop testing):
_applyKeyboard(speed) {
  const keys = this._keys;
  let vx = 0, vy = 0;
  if (keys.left.isDown  || keys.a.isDown) vx = -1;
  if (keys.right.isDown || keys.d.isDown) vx =  1;
  if (keys.up.isDown    || keys.w.isDown) vy = -1;
  if (keys.down.isDown  || keys.s.isDown) vy =  1;
  if (vx !== 0 && vy !== 0) { vx *= 0.707; vy *= 0.707; }  // diagonal normalise
  this.player.body.setVelocity(vx * speed, vy * speed);
}

// In create():
this._keys = this.input.keyboard.addKeys({
  up: 'UP', down: 'DOWN', left: 'LEFT', right: 'RIGHT',
  w: 'W',   s: 'S',       a: 'A',       d: 'D',
  space: 'SPACE',
});
```

---

## 6. Action Button — Complete Implementation

Paired with the joystick above. A visible primary action button for the right half of the screen — shoot, jump, interact, or any genre-appropriate action. Tracks a separate pointer from the joystick.

```js
_createActionButton() {
  const fx = W - 75, fy = H - 110;

  this._actionBtn = this.add.circle(fx, fy, 38, 0xF55018, 0.15)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0)
    .setInteractive();
  this.add.circle(fx, fy, 38, 0x000000, 0)
    .setStrokeStyle(2, 0xF55018, 0.5)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0);

  // Icon — customise to match your genre (triangle = shoot, circle = jump, etc.)
  const icon = this.add.triangle(fx, fy, 0, 10, -9, -6, 9, -6, 0xF55018, 0.9)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0);

  this._actionHeld = false;

  this.input.on('pointerdown', (p) => {
    if (p.x < W / 2) return;   // left half is joystick
    this._actionHeld = true;
    this._actionBtn.setFillStyle(0xF55018, 0.35);
    icon.setAlpha(1);
  });
  this.input.on('pointerup', (p) => {
    if (p.x < W / 2) return;
    this._actionHeld = false;
    this._actionBtn.setFillStyle(0xF55018, 0.15);
    icon.setAlpha(0.9);
  });
}
```

---

## 7. Particle Emitters — Phaser 3.60+ API

**The old API (`this.add.particles('key').createEmitter({})`) was removed in Phaser 3.60. Using it produces a silent crash.** The new API takes position and config directly on `this.add.particles()`.

```js
// Death burst — call once at enemy position, auto-destroys
_spawnDeathBurst(x, y, colour) {
  // Outer colour burst
  this.add.particles(x, y, 'pixel', {
    speed:     { min: 80, max: 220 },
    angle:     { min: 0, max: 360 },
    scale:     { start: 3.5, end: 0 },
    alpha:     { start: 1,   end: 0 },
    tint:      colour,
    lifespan:  380,
    quantity:  14,
    emitting:  false,   // ← fire once, not continuous
  }).explode(14);        // ← explode(count) fires immediately then auto-destroys

  // Inner white core
  this.add.particles(x, y, 'pixel', {
    speed:     { min: 40, max: 100 },
    angle:     { min: 0, max: 360 },
    scale:     { start: 2.5, end: 0 },
    alpha:     { start: 1,   end: 0 },
    lifespan:  200,
    quantity:  7,
    emitting:  false,
  }).explode(7);
}

// Hit spark — lighter, smaller
_spawnHitSpark(x, y) {
  this.add.particles(x, y, 'pixel', {
    speed:    { min: 40, max: 120 },
    angle:    { min: 0, max: 360 },
    scale:    { start: 2, end: 0 },
    alpha:    { start: 0.9, end: 0 },
    lifespan: 180,
    quantity: 6,
    emitting: false,
  }).explode(6);
}

// Continuous emitter (e.g. player trail) — store reference to stop it
_startTrail() {
  this._trail = this.add.particles(0, 0, 'pixel', {
    follow:    this.player,             // follows the target automatically
    speed:     { min: 10, max: 30 },
    scale:     { start: 1.5, end: 0 },
    alpha:     { start: 0.4, end: 0 },
    tint:      0xF55018,
    lifespan:  200,
    frequency: 40,                      // ms between emissions
  });
}

// Stop the continuous emitter cleanly
_stopTrail() {
  if (this._trail) { this._trail.stop(); this._trail = null; }
}
```

---

## 8. Physics Groups — Choosing the Right Type

Using the wrong group type causes `overlap()` and `collide()` to silently never fire.

```js
// ✅ CORRECT — physics-aware group, works with overlap() and collide()
this.enemies = this.physics.add.group();
this.bullets = this.physics.add.group();

// ✅ CORRECT — static bodies that never move (walls, platforms)
this.walls = this.physics.add.staticGroup();

// ❌ WRONG — no physics body, overlap() never fires
this.enemies = this.add.group();     // don't use this.add.group() for collisions
this.enemies = this.add.group({ classType: Phaser.Physics.Arcade.Sprite }); // also wrong

// Adding objects to a physics group
const enemy = this.enemies.create(x, y, 'enemy');  // ✅ auto-adds physics body
// OR
const enemy = this.physics.add.sprite(x, y, 'enemy');
this.enemies.add(enemy);                             // ✅ also correct
```

---

## 9. Safe Group Iteration

Iterating `getChildren()` while destroying members causes skipped indices and crashes. Always filter first or iterate a copy.

```js
// ✅ CORRECT — filter active children before iterating
this.enemies.getChildren().filter(e => e.active).forEach(enemy => {
  // safe to read, safe to destroy inside this loop
});

// ✅ CORRECT — iterate a snapshot copy when you may destroy members
const active = [...this.enemies.getChildren()];
active.forEach(enemy => {
  if (!enemy.active) return;
  // your logic
});

// ❌ WRONG — .entries is internal, index shifts on destroy
this.enemies.children.entries.forEach(e => { ... });

// ❌ WRONG — not filtered, includes destroyed (inactive) members
this.enemies.getChildren().forEach(e => { ... });
```

---

## 10. gameOver() — Safe Call Pattern

`window.gameOver()` called directly inside a physics overlap callback fires during the physics step. Phaser throws errors and the score may not submit. Always defer by one tick.

```js
// ❌ WRONG — called during physics step
this.physics.add.overlap(this.player, this.hazards, () => {
  window.gameOver(this.score);   // crashes in physics callback
});

// ✅ CORRECT — defer by one tick with delayedCall(0)
this.physics.add.overlap(this.player, this.hazards, () => {
  if (this._dead) return;
  this._dead = true;
  this._runDeathSequence();
});

_runDeathSequence() {
  this.cameras.main.shake(400, 0.022);
  this._spawnDeathBurst(this.player.x, this.player.y, 0xF55018);
  this.player.setActive(false).setVisible(false);

  // Slow motion during death
  this.physics.world.timeScale = 3;
  this.time.timeScale = 0.33;

  this.time.delayedCall(900, () => {
    this.physics.world.timeScale = 1;
    this.time.timeScale = 1;
    window.gameOver(this.score);        // ✅ safe — called outside physics step
    this.cameras.main.fadeOut(400);
    this.time.delayedCall(420, () => {
      this.scene.start('GameOverScene', {
        score:    this.score,
        runState: this.runState,
      });
    });
  });
}
```

---

## 11. Hit Flash — Enemy and Player

The white flash must happen first, then the tint return. Reversing the order looks like a colour change, not an impact.

```js
// Enemy hit flash
_flashEnemy(enemy) {
  if (!enemy.active || enemy._flashing) return;
  enemy._flashing = true;
  enemy.setTint(0xffffff);                        // instant white
  this.time.delayedCall(65, () => {
    if (!enemy.active) return;
    enemy.clearTint();                            // back to normal
    enemy._flashing = false;
  });
}

// Player hit flash + invincibility flicker
_flashPlayer() {
  if (this._iFrames) return;
  this._iFrames = true;
  this.player.setTint(0xffffff);

  // Flicker alpha during i-frames to signal invincibility
  const flicker = this.time.addEvent({
    delay: 80, repeat: 10,
    callback: () => {
      if (!this.player.active) return;
      this.player.setAlpha(this.player.alpha < 1 ? 1 : 0.25);
    },
  });

  this.time.delayedCall(900, () => {
    flicker.remove();
    if (this.player.active) {
      this.player.clearTint();
      this.player.setAlpha(1);
    }
    this._iFrames = false;
  });
}
```

---

## 12. HUD Text — Stroke, Scroll Factor, Depth

HUD text without these three properties will be unreadable, scroll with the camera, and appear behind game objects.

```js
_createHUD() {
  // Semi-transparent backing strip for readability
  this._hudBg = this.add.rectangle(0, 0, W, 52, 0x000000, 0.45)
    .setOrigin(0, 0)
    .setDepth(DEPTH.HUD_BG)
    .setScrollFactor(0);                   // ← does not scroll with camera

  // Primary counter (score, distance, time) — top right
  this._primaryTxt = this.add.text(W - 16, 14, '0', {
    fontFamily: 'monospace',
    fontSize:   '22px',
    color:      '#ffffff',
    stroke:     '#000000',                 // ← required — renders over any background
    strokeThickness: 4,
    align:      'right',
  })
    .setOrigin(1, 0)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0);

  // Secondary info (level, depth, lives, timer) — top centre
  // What this shows is your design decision — keep it to one piece of information
  this._secondaryTxt = this.add.text(W / 2, 14, '', {
    fontFamily: 'monospace',
    fontSize:   '13px',
    color:      '#aaaaaa',
    stroke:     '#000000',
    strokeThickness: 3,
  })
    .setOrigin(0.5, 0)
    .setDepth(DEPTH.HUD)
    .setScrollFactor(0);
}

_updateHUD() {
  this._primaryTxt.setText(this.score.toLocaleString());
  // Update secondary text with whatever your genre tracks — depth, lives, time remaining, etc.
}
```

---

## 13. Timer Events — addEvent Pattern

`time.addEvent` is the correct way to schedule recurring game logic. Never use `setInterval` — it runs outside Phaser's time scale and ignores `timeScale` changes (slow motion breaks it).

```js
// One-shot delay
this.time.delayedCall(500, () => { this._spawnWave(); });

// Recurring event — store reference to remove later
this._spawnTimer = this.time.addEvent({
  delay:    2000,
  callback: this._spawnEnemy,
  callbackScope: this,
  loop:     true,
});

// Remove when room is cleared
this._spawnTimer.remove();

// Repeat N times
this.time.addEvent({
  delay:    150,
  repeat:   5,                            // fires 6 times total (initial + 5 repeats)
  callback: () => { this._staggerSpawn(); },
});

// Scale difficulty — replace the timer with a new one (don't mutate the delay)
_increaseDifficulty() {
  this._spawnTimer.remove();
  this._spawnTimer = this.time.addEvent({
    delay:    Math.max(600, 2000 - this.runState.depth * 120),
    callback: this._spawnEnemy,
    callbackScope: this,
    loop:     true,
  });
}
```

---

## 14. Selection Overlay — Pause + Choice + Resume Pattern

For any moment where the game pauses and presents the player with a set of choices before continuing — upgrades, route selection, item picks, level rewards. The game world freezes but stays visible behind the overlay.

```js
// In GameScene — show selection screen:
_showSelectionScreen(choices) {
  this.scene.pause('GameScene');                 // freeze game world
  this.scene.launch('SelectionScene', {         // launch overlay on top
    choices:  choices,
    gameData: this._buildGameData(),            // pass whatever your genre needs
  });

  // Listen for the player's choice
  this.scene.get('SelectionScene').events.once('choice-made', (choice) => {
    this._applyChoice(choice);
    this.scene.resume('GameScene');
    this._onSelectionComplete();
  });
}

// SelectionScene — renders on top of the paused GameScene
class SelectionScene extends Phaser.Scene {
  constructor() { super({ key: 'SelectionScene', active: false }); }

  init(data) {
    this.choices  = data.choices;
    this.gameData = data.gameData;
  }

  create() {
    // Backdrop — blocks input to the paused scene below (mandatory)
    this.add.rectangle(0, 0, W, H, 0x000000, 0.72)
      .setOrigin(0)
      .setDepth(0)
      .setInteractive();

    this.choices.forEach((choice, i) => {
      const y    = 280 + i * 145;

      // Card background — style to your game's design
      const card = this.add.rectangle(W / 2, y, 300, 120, 0x1a1a1a)
        .setStrokeStyle(2, 0xffffff, 0.4)
        .setInteractive()
        .setDepth(1);

      // Primary label — name, title, or option identifier
      this.add.text(W / 2, y - 22, choice.name, {
        fontFamily: 'monospace', fontSize: '18px',
        color: '#ffffff', stroke: '#000', strokeThickness: 3,
      }).setOrigin(0.5).setDepth(2);

      // Secondary label — description, cost, effect summary
      this.add.text(W / 2, y + 12, choice.description, {
        fontFamily: 'monospace', fontSize: '12px',
        color: '#aaaaaa', stroke: '#000', strokeThickness: 2,
        wordWrap: { width: 260 },
      }).setOrigin(0.5).setDepth(2);

      // Staggered entry animation
      card.setAlpha(0).setScale(0.88);
      this.tweens.add({
        targets: card, alpha: 1, scaleX: 1, scaleY: 1,
        duration: 180, delay: i * 90, ease: 'Back.Out',
      });

      card.on('pointerdown', () => {
        this.events.emit('choice-made', choice);
        this.scene.stop();
      });
    });
  }
}

// Register SelectionScene in the game config:
// scene: [BootScene, MenuScene, GameScene, SelectionScene, GameOverScene]
```

---

## 15. Audio — Web Audio API Pattern

Never create `AudioContext` at file scope or in `create()`. Create it on the first pointer interaction. Cache the single instance — never create more than one.

```js
// In GameScene.create():
this._audio = null;
this.input.once('pointerdown', () => {
  if (this._audio) return;
  this._audio = new (window.AudioContext || window.webkitAudioContext)();
});

// Generic sound helper — call this for every game sound
_playSound(type) {
  if (!this._audio) return;
  const ctx = this._audio;
  const g   = ctx.createGain();
  g.connect(ctx.destination);

  const osc = ctx.createOscillator();
  osc.connect(g);

  switch (type) {
    case 'shoot':
      osc.type = 'square';
      osc.frequency.setValueAtTime(520, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(180, ctx.currentTime + 0.07);
      g.gain.setValueAtTime(0.18, ctx.currentTime);
      g.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.08);
      osc.start(); osc.stop(ctx.currentTime + 0.08);
      break;

    case 'hit':
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(200, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(60, ctx.currentTime + 0.12);
      g.gain.setValueAtTime(0.3, ctx.currentTime);
      g.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.13);
      osc.start(); osc.stop(ctx.currentTime + 0.13);
      break;

    case 'death':
      osc.type = 'sawtooth';
      osc.frequency.setValueAtTime(300, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(40, ctx.currentTime + 0.5);
      g.gain.setValueAtTime(0.5, ctx.currentTime);
      g.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.55);
      osc.start(); osc.stop(ctx.currentTime + 0.55);
      break;

    case 'upgrade':
      // Two-tone rising chime
      const osc2 = ctx.createOscillator();
      osc2.connect(g);
      osc.type = 'sine';
      osc.frequency.setValueAtTime(440, ctx.currentTime);
      osc2.type = 'sine';
      osc2.frequency.setValueAtTime(660, ctx.currentTime + 0.12);
      g.gain.setValueAtTime(0.25, ctx.currentTime);
      g.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.4);
      osc.start(); osc.stop(ctx.currentTime + 0.12);
      osc2.start(ctx.currentTime + 0.12); osc2.stop(ctx.currentTime + 0.4);
      break;

    case 'pickup':
      osc.type = 'sine';
      osc.frequency.setValueAtTime(600, ctx.currentTime);
      osc.frequency.exponentialRampToValueAtTime(900, ctx.currentTime + 0.1);
      g.gain.setValueAtTime(0.2, ctx.currentTime);
      g.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.12);
      osc.start(); osc.stop(ctx.currentTime + 0.12);
      break;
  }
}
```

---

## 16. Object Pooling — Short-Lived Entities

Never create objects in `update()`. Pre-allocate a pool in `create()` and recycle from it. This applies to any frequently spawned and destroyed object — projectiles, particles stand-ins, collectibles, debris.

```js
// In create():
this.projectiles = this.physics.add.group({
  classType:  Phaser.Physics.Arcade.Image,
  maxSize:    40,
  runChildUpdate: false,
});

// Pre-populate the pool (optional — group lazy-creates if needed)
for (let i = 0; i < 40; i++) {
  const b = this.projectiles.create(0, 0, 'projectile');
  b.setActive(false).setVisible(false);
}

// Spawn from pool — position, activate, set velocity
_spawnProjectile(x, y, angle) {
  const b = this.projectiles.getFirstDead(false);
  if (!b) return;   // pool exhausted — skip

  b.setPosition(x, y)
   .setActive(true)
   .setVisible(true)
   .setDepth(DEPTH.PROJECTILES)
   .setRotation(angle);

  const speed = 560;
  this.physics.velocityFromAngle(
    Phaser.Math.RadToDeg(angle), speed, b.body.velocity
  );

  // Auto-despawn after max range — use time, not off-screen check
  this.time.delayedCall(700, () => {
    if (b.active) b.setActive(false).setVisible(false);
  });
}

// In the overlap callback — recycle on hit:
this.physics.add.overlap(this.projectiles, this.hazards, (projectile, hazard) => {
  projectile.setActive(false).setVisible(false);
  this._onHit(hazard);
});
```

---

## 17. World Boundary — Visible Edge

Players must perceive the boundary before hitting it. Draw it in `create()`.

```js
_drawBoundary() {
  this.physics.world.setBounds(0, 60, W, H - 60);  // 60px for HUD

  const border = this.add.graphics().setDepth(DEPTH.BG_DETAIL);
  border.lineStyle(2, 0x333333, 0.6);
  border.strokeRect(4, 64, W - 8, H - 68);

  // Corner accents — visual style should match your game's theme
  const len = 20, thick = 2;
  const corners = [
    [4, 64], [W - 4, 64], [4, H - 4], [W - 4, H - 4]
  ];
  corners.forEach(([cx, cy]) => {
    border.lineStyle(thick, 0xF55018, 0.7);
    const sx = cx === 4 ? 1 : -1;
    const sy = cy === 64 ? 1 : -1;
    border.beginPath();
    border.moveTo(cx, cy + sy * len);
    border.lineTo(cx, cy);
    border.lineTo(cx + sx * len, cy);
    border.strokePath();
  });

  this.player.setCollideWorldBounds(true);
}
```

---

## 18. Floating Score Text

Always use `stroke` + `strokeThickness`. Animate up and fade.

```js
_floatText(x, y, text, colour = '#ffffff') {
  const t = this.add.text(x, y, text, {
    fontFamily: 'monospace',
    fontSize:   '16px',
    color:      colour,
    stroke:     '#000000',
    strokeThickness: 4,
  })
    .setOrigin(0.5)
    .setDepth(DEPTH.FX);

  this.tweens.add({
    targets:  t,
    y:        y - 55,
    alpha:    0,
    duration: 900,
    ease:     'Cubic.Out',
    onComplete: () => t.destroy(),
  });
}
```

---

## 19. Scene Lifecycle — Full Order

Phaser calls these in this exact order, every scene start:

```
constructor() → init(data) → preload() → create() → update(time, delta) [loop]
```

- **`constructor()`** — call `super('SceneName')` only. No game logic.
- **`init(data)`** — receive data from `scene.start('Name', data)`. Set instance variables. No Phaser APIs available yet.
- **`preload()`** — for scenes that load external assets. For Sparkade games this is always empty — all textures are generated in BootScene.
- **`create()`** — build the scene. All Phaser APIs available. Run state from `init()` is ready.
- **`update(time, delta)`** — runs every frame. `delta` is milliseconds since last frame. Always gate time-based logic through `delta`, never assume frame rate.

```js
class GameScene extends Phaser.Scene {
  constructor() { super('GameScene'); }    // ✅

  init(data) {
    this.runState = data.runState || this._defaultRunState();
  }

  preload() {
    // Leave empty — textures come from BootScene
  }

  create() {
    this._buildRoom();
    this._createPlayer();
    this._createHUD();
    this._createJoystick();
    this._createFireButton();
    this._dead = false;
    this._iFrames = false;
  }

  update(time, delta) {
    if (this._dead) return;
    this._applyJoystick(220);
    this._handleKeyboard();
    this._autoFire(delta);
    this._clampPlayer();
  }
}
```

---

## 20. Hitstop — Freeze Frames on Impact

Briefly pauses the physics simulation on a significant hit. Guard against stacking with a flag — nested hitstops compound incorrectly.

```js
// In create():
this._hitstop = false;

// Call on any significant hit
_hitstop(duration = 80) {
  if (this._hitstop) return;           // never stack
  this._hitstop = true;

  this.physics.world.timeScale = 999;  // freeze physics
  this.time.timeScale = 0.001;         // freeze timers

  // Use real time (not game time) to resume — game time is frozen
  setTimeout(() => {
    this.physics.world.timeScale = 1;
    this.time.timeScale = 1;
    this._hitstop = false;
  }, duration);
}

// Usage — scale duration to significance:
// Regular enemy kill:  this._hitstop(60);
// Boss phase change:   this._hitstop(120);
// Player death:        this._hitstop(150);  then run death sequence
```

**Why `setTimeout` not `time.delayedCall`:** `time.delayedCall` uses game time, which is frozen by `time.timeScale = 0.001`. It would never fire. `setTimeout` uses real wall-clock time and always fires correctly.

---

## 21. Score Counter Tween — Animated Number Counting

Use `tweens.addCounter()` to animate a score counting up rather than jumping instantly to its new value.

```js
// In-game HUD — fast, snappy (called whenever score changes)
_animateScoreHUD(from, to) {
  this.tweens.addCounter({
    from:     from,
    to:       to,
    duration: 300,
    ease:     'Cubic.Out',
    onUpdate: (tween) => {
      this._scoreTxt.setText(Math.floor(tween.getValue()).toLocaleString());
    },
  });
}

// GameOver screen — slower, more ceremony
_animateScoreFinal(finalScore) {
  const display = { val: 0 };

  this.tweens.addCounter({
    from:     0,
    to:       finalScore,
    duration: 1800,
    ease:     'Cubic.Out',
    onUpdate: (tween) => {
      this._finalScoreTxt.setText(Math.floor(tween.getValue()).toLocaleString());
    },
    onComplete: () => {
      if (this._isPersonalBest) this._showPersonalBestBadge();
    },
  });
}

// Personal best badge — animate in after score finishes counting
_showPersonalBestBadge() {
  const badge = this.add.text(W / 2, 320, '★ NEW BEST', {
    fontFamily: 'monospace',
    fontSize:   '16px',
    color:      '#FFD700',
    stroke:     '#000000',
    strokeThickness: 4,
  }).setOrigin(0.5).setAlpha(0).setScale(0.5).setDepth(DEPTH.OVERLAY);

  this.tweens.add({
    targets:  badge,
    alpha:    1,
    scaleX:   1,
    scaleY:   1,
    duration: 350,
    ease:     'Back.Out',
  });
}
```
