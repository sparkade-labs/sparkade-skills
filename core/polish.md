# Polish & Juice — Sparkade Core Skill

The difference between a game that feels cheap and one that feels premium is almost entirely juice. Implement every item in this skill. None of them are optional for a high-quality Sparkade game.

---

## Screen Shake

```js
// Light shake — on player hit
this.cameras.main.shake(80, 0.006);

// Heavy shake — on boss death, room clear
this.cameras.main.shake(200, 0.014);

// Directional nudge — on player taking damage
this.cameras.main.shake(120, 0.009);
```

Call shake sparingly — every hit, not every frame. Over-shaking is worse than none.

---

## Hit Flash

When any character takes damage, flash them white for 2 frames:

```js
hitFlash(sprite) {
  sprite.setTintFill(0xffffff);
  this.time.delayedCall(60, () => sprite.clearTint());
}
```

For enemies, chain a red tint after the white flash:
```js
hitFlash(sprite) {
  sprite.setTintFill(0xffffff);
  this.time.delayedCall(50, () => sprite.setTint(0xff4444));
  this.time.delayedCall(150, () => sprite.clearTint());
}
```

---

## Hitstop (Freeze Frames)

Brief time-freeze on a significant hit makes impacts feel weighty:

```js
hitstop(duration = 80) {
  this.physics.pause();
  this._hitstopActive = true;
  this.time.delayedCall(duration, () => {
    this.physics.resume();
    this._hitstopActive = false;
  });
}
```

Use `hitstop(60)` on regular kills, `hitstop(120)` on boss kills. Do not stack hitstops — guard with `if (this._hitstopActive) return`.

---

## Knockback

```js
knockback(sprite, fromX, fromY, force = 180, duration = 120) {
  const angle = Phaser.Math.Angle.Between(fromX, fromY, sprite.x, sprite.y);
  const vx = Math.cos(angle) * force;
  const vy = Math.sin(angle) * force;
  sprite.body.setVelocity(vx, vy);
  this.time.delayedCall(duration, () => {
    if (sprite.active) sprite.body.setVelocity(0, 0);
  });
}
```

Apply to enemies on hit. Apply weaker knockback to the player on taking damage (`force: 120`).

---

## Death Burst Particles

On enemy death, emit a burst of colour-matched particles:

```js
spawnDeathBurst(x, y, color = 0xF55018, count = 12) {
  const particles = this.add.particles(x, y, 'pixel', {
    speed: { min: 60, max: 180 },
    angle: { min: 0, max: 360 },
    scale: { start: 2.5, end: 0 },
    lifespan: { min: 280, max: 480 },
    quantity: count,
    tint: color,
    gravityY: 80,
    emitting: false,
  });
  particles.explode(count);
  this.time.delayedCall(600, () => particles.destroy());
}
```

Requires a `'pixel'` texture (1×1 white square) generated in BootScene:
```js
// In BootScene
const g = this.make.graphics({ add: false });
g.fillStyle(0xffffff);
g.fillRect(0, 0, 1, 1);
g.generateTexture('pixel', 1, 1);
g.destroy();
```

---

## Floating Score Text

Pop up "+100" text on kills, floating upward and fading:

```js
floatText(x, y, text, color = '#ffffff') {
  const t = this.add.text(x, y, text, {
    fontFamily: 'monospace', fontSize: '14px', fontStyle: 'bold', color,
  }).setOrigin(0.5).setDepth(90);

  this.tweens.add({
    targets: t,
    y: y - 40,
    alpha: { from: 1, to: 0 },
    duration: 700,
    ease: 'Quad.Out',
    onComplete: () => t.destroy(),
  });
}
```

Use sparingly — only on kills or notable events, not every hit.

---

## Score Counter Tween

When the score updates, tween the displayed number rather than jumping:

```js
updateScore(newScore) {
  const oldScore = this._displayScore;
  this._displayScore = newScore;
  this.tweens.addCounter({
    from: oldScore,
    to: newScore,
    duration: 300,
    ease: 'Quad.Out',
    onUpdate: (tween) => {
      this._scoreText.setText(Math.floor(tween.getValue()).toString());
    },
  });
}
```

---

## Death Slow Motion

When the player dies, slow time before transitioning to GameOver:

```js
playerDeath() {
  this.physics.world.timeScale = 4; // slow to 25% speed
  this.time.timeScale           = 0.25;

  this.time.delayedCall(1200, () => {
    this.physics.world.timeScale = 1;
    this.time.timeScale           = 1;
    this.scene.start('GameOver', { score: this._score, depth: this._depth });
  });
}
```

---

## UI Depth Layers

Always set explicit depths to prevent z-order bugs:

| Layer | Depth | Contents |
|-------|-------|----------|
| Background | 0 | Floor, walls, tiles |
| Shadows | 1 | Drop shadows |
| Pickups | 5 | Items, health orbs |
| Enemies | 10 | Enemy sprites |
| Player | 20 | Player sprite |
| Projectiles | 30 | Bullets, effects |
| Particles | 40 | Death bursts, hit sparks |
| VFX | 50 | Explosions, area effects |
| HUD | 80 | Health bar, score |
| Floating text | 90 | Score popups |
| Joystick | 100 | Touch controls |
| Overlay | 200 | Pause, room transition |

---

## Room Transition Flash

Between rooms, flash to white and back:

```js
roomTransition(onMidpoint) {
  const flash = this.add.rectangle(195, 422, 390, 844, 0xffffff).setDepth(200).setAlpha(0);
  this.tweens.add({
    targets: flash,
    alpha: { from: 0, to: 1 },
    duration: 150,
    yoyo: true,
    onYoyo: () => onMidpoint(),
    onComplete: () => flash.destroy(),
  });
}
```

---

## Audio (Procedural Beeps)

No audio files are available. Use the Web Audio API for generated sound effects:

```js
playSound(type) {
  const ctx = new (window.AudioContext || window.webkitAudioContext)();
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  osc.connect(gain);
  gain.connect(ctx.destination);

  const sounds = {
    shoot:  { freq: 440, type: 'square', duration: 0.05, vol: 0.15 },
    hit:    { freq: 180, type: 'sawtooth', duration: 0.08, vol: 0.2 },
    die:    { freq: 80,  type: 'sawtooth', duration: 0.3,  vol: 0.25 },
    pickup: { freq: 660, type: 'sine',    duration: 0.1,  vol: 0.15 },
    levelup:{ freq: 880, type: 'sine',    duration: 0.2,  vol: 0.2  },
  };

  const s = sounds[type] || sounds.hit;
  osc.type = s.type;
  osc.frequency.setValueAtTime(s.freq, ctx.currentTime);
  osc.frequency.exponentialRampToValueAtTime(s.freq * 0.5, ctx.currentTime + s.duration);
  gain.gain.setValueAtTime(s.vol, ctx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + s.duration);
  osc.start(ctx.currentTime);
  osc.stop(ctx.currentTime + s.duration);
}
```

Cache the `AudioContext` instance — don't create a new one per call.
