> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# HiDPI — Sparkade Core Skill

Every Sparkade game must use the `phaser-hidpi` plugin. Without it, all text and graphics are blurry on high pixel density phones (which is every modern smartphone). This is non-negotiable.

---

## How it works

Phaser's default rendering uses CSS pixels. On a phone with `devicePixelRatio: 3`, the canvas backing buffer is 1/3 the native screen resolution and the browser upscales it — producing visible blur. The plugin solves this by:

1. Sizing the internal canvas buffer to physical pixels (`width × DPR` by `height × DPR`)
2. Setting `scale.zoom = 1 / DPR` so it displays at the correct logical size
3. Providing a `px(n)` helper that returns `n × DPR` — used to convert logical measurements to physical pixel space

---

## The shell loads it for you

The Sparkade shell loads both scripts in order before your game runs:

```html
<script src="https://assets.sparkade.io/lib/phaser.min.js"></script>
<script src="https://assets.sparkade.io/lib/phaser-hidpi.js"></script>
```

`devicePixelRender` is available as a global when your game script executes. Do not load it yourself.

---

## Initialization — use this exact pattern

```js
// Sparkade logical canvas size — fixed, do not use window.innerWidth
const LOGICAL_W = 390;
const LOGICAL_H = 844;

const { game: hidpiConfig, px } = devicePixelRender({
  width:  LOGICAL_W,
  height: LOGICAL_H,
});

// W and H are now in physical pixels — use these everywhere
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

`Object.assign(hidpiConfig, { ... })` merges your config into the plugin's config. The plugin sets `width`, `height`, and `scale.zoom` — everything else is yours.

Do not set `resolution:` manually. The plugin handles it.

---

## The px() rule

`px(n)` returns `n × devicePixelRatio` on HiDPI screens and `n` on standard displays.

**Wrap with `px()`:**
- All coordinates passed to Phaser objects: `this.add.text(px(195), px(20), ...)`
- All sizes and radii: `fillCircle(x, y, px(14))`, `fillRect(0, 0, W, px(52))`
- Font sizes: `fontSize: px(22) + 'px'`
- Line widths: `lineStyle(px(1.5), 0xffffff, 0.2)`
- Stroke thickness: `strokeThickness: px(4)`
- Line spacing: `lineSpacing: px(8)`
- Fixed pixel offsets: `setPosition(W / 2, H - px(110))`
- Physics velocities: `setVelocity(px(220), 0)` — the world is in physical pixels
- Tween pixel targets: `tweens.add({ targets: t, y: px(-64), ... })`
- Container child offsets: `text.y = px(-115)`

**Do NOT wrap:**
- `W` and `H` — already pre-multiplied at initialisation
- Expressions derived from `W` or `H`: `W / 2`, `H - px(110)` (the px offset is wrapped, H is not)
- Colours: `0xF55018`, `'#ffffff'`
- Alpha values: `fillStyle(colour, 0.85)`
- Scale arguments: `setScale(1.2)`
- Angles and rotation: `setAngle(-15)`
- Durations and delays: `duration: 400`, `delay: 80`
- Easing strings: `ease: 'Back.Out'`
- `repeat`, `quantity`, `frequency` parameters in particles and timers

---

## Pointer input

Because all your coordinates are in physical pixels (via `px()`), pointer coordinates from Phaser are also in physical pixels. Hit detection works without transformation:

```js
this.input.on('pointerdown', (ptr) => {
  const dist = Phaser.Math.Distance.Between(ptr.x, ptr.y, target.x, target.y);
  if (dist <= RADIUS) { /* hit */ }
});
```

No camera correction needed.

---

## Quick reference — common patterns updated for HiDPI

```js
// Text
this.add.text(W / 2, px(14), 'SCORE', {
  fontFamily:      'monospace',
  fontSize:        px(22) + 'px',
  color:           '#ffffff',
  stroke:          '#000000',
  strokeThickness: px(4),
}).setOrigin(0.5, 0).setDepth(100).setScrollFactor(0);

// Rectangle
this.add.rectangle(0, 0, W, px(52), 0x000000, 0.45)
  .setOrigin(0).setDepth(90).setScrollFactor(0);

// Graphics circle (in BootScene texture generation)
const g = this.make.graphics({ add: false });
g.fillStyle(0xF55018, 0.18);
g.fillCircle(px(14), px(16), px(13));   // shadow
g.fillStyle(0xF55018, 1);
g.fillCircle(px(14), px(13), px(11));   // body
g.lineStyle(px(1.5), 0x000000, 0.25);
g.strokeCircle(px(14), px(13), px(11)); // outline
g.generateTexture('player', px(28), px(28));
g.destroy();

// Physics velocity
this.player.body.setVelocity(this._joyVec.x * px(220), this._joyVec.y * px(220));

// Joystick constants
const jx = px(90), jy = H - px(110);
const DEAD = px(12), MAX = px(45);
```
