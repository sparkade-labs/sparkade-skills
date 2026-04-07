> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# In-Game UI — Sparkade Core Skill

Every piece of UI in a Sparkade game is drawn with Phaser's Graphics and Text objects. No HTML, no DOM, no CSS. This skill covers every UI pattern you will need — panels, buttons, modals, health bars, tooltips, and notifications.

Read this after `patterns.md`. The depth constants (`DEPTH.HUD`, `DEPTH.OVERLAY`) defined there are used throughout.

---

## Non-Negotiable Technical Rules

These are not design decisions — they are correctness requirements. Every other visual choice (colours, fonts, corner radius, opacity, layout) belongs to the game's creative brief.

- **All text rendered over gameplay must have `stroke` + `strokeThickness`** — the game world behind it can be any colour. Text without stroke becomes unreadable the moment a bright sprite passes underneath.
- **All HUD elements must have `.setScrollFactor(0)`** — without it they scroll with the camera and drift off screen.
- **All HUD and overlay elements must have an explicit `.setDepth()`** — without it z-order is undefined and UI disappears behind game objects.
- **All interactive elements must have a minimum 48×48px hit area** — regardless of visual size. Use `setInteractive(geom)` to expand the hit area without changing the visual.
- **All modal backdrops must call `.setInteractive()`** — even with no handler. This is what blocks touch input from passing through to the game world below.
- **Exit must appear in menus, not overlaid on gameplay** — see Pattern 11.

Everything else — what the UI looks like — is your creative decision. Match it to the game's theme.

---

## 1. Panel — Rounded Rectangle with Header

The base container for any in-game overlay. Visual style (colours, radius, opacity) should match your game's theme — the structure below is the mechanism, not the aesthetic.

```js
_createPanel(x, y, w, h, title) {
  const panel = this.add.container(x, y).setDepth(DEPTH.OVERLAY);

  // Background — choose colours that fit your game's theme
  const bg = this.add.graphics();
  bg.fillStyle(0x141414, 0.97);          // dark panel — adjust to theme
  bg.fillRoundedRect(-w / 2, -h / 2, w, h, 16);
  bg.lineStyle(1.5, 0x2a2a2a, 1);        // border — adjust to theme
  bg.strokeRoundedRect(-w / 2, -h / 2, w, h, 16);

  // Header strip — slightly lighter than panel bg
  const header = this.add.graphics();
  header.fillStyle(0x1e1e1e, 1);         // adjust to theme
  header.fillRoundedRect(-w / 2, -h / 2, w, 52, { tl: 16, tr: 16, bl: 0, br: 0 });

  // Title — font family, size, and colour should match your game's identity
  const titleTxt = this.add.text(0, -h / 2 + 26, title.toUpperCase(), {
    fontFamily: 'monospace',             // replace with your game's font
    fontSize:   '14px',
    color:      '#ffffff',
    stroke:     '#000000',
    strokeThickness: 3,                  // stroke is mandatory — value is your choice
    letterSpacing: 2,
  }).setOrigin(0.5);

  // Drag handle
  const handle = this.add.rectangle(0, -h / 2 + 8, 36, 4, 0x444444, 1)
    .setOrigin(0.5);

  panel.add([bg, header, titleTxt, handle]);
  return panel;
}
```

---

## 2. Button — Three States: Default, Pressed, Disabled

Every button needs these three states regardless of visual style. The mechanics below are fixed — colours, fonts, and radius are yours to define to match the game's theme.

```js
// Call with your game's colour choices — do not copy these hex values verbatim
_createButton(x, y, label, style, onTap) {
  // style: { fillColour, fillAlpha, strokeColour, labelColour, radius }
  // Define styles that fit your game's visual identity, for example:
  //   primary:   { fillColour: 0x<yourAccent>, fillAlpha: 1, ... }
  //   secondary: { fillColour: 0x000000, fillAlpha: 0, strokeColour: 0x<yourBorder>, ... }

  const W_BTN = 260, H_BTN = 52;

  const bg = this.add.graphics()
    .setDepth(DEPTH.OVERLAY)
    .setInteractive(
      new Phaser.Geom.Rectangle(-W_BTN / 2, -H_BTN / 2, W_BTN, H_BTN),
      Phaser.Geom.Rectangle.Contains
    );

  const txt = this.add.text(x, y, label, {
    fontFamily: style.fontFamily || 'monospace',
    fontSize:   '16px',
    fontStyle:  'bold',
    color:      style.labelColour,
    stroke:     '#000000',              // stroke is mandatory
    strokeThickness: 3,
  }).setOrigin(0.5).setDepth(DEPTH.OVERLAY);

  const _draw = (pressed, disabled) => {
    bg.clear();
    bg.fillStyle(style.fillColour, disabled ? style.fillAlpha * 0.3 : style.fillAlpha);
    bg.fillRoundedRect(x - W_BTN / 2, y - H_BTN / 2, W_BTN, pressed ? H_BTN - 2 : H_BTN, style.radius || 13);
    bg.lineStyle(1.5, style.strokeColour, disabled ? 0.3 : 0.8);
    bg.strokeRoundedRect(x - W_BTN / 2, y - H_BTN / 2, W_BTN, pressed ? H_BTN - 2 : H_BTN, style.radius || 13);
    txt.setAlpha(disabled ? 0.35 : 1).setY(pressed ? y + 1 : y);
  };

  _draw(false, false);

  bg.on('pointerover',  () => { bg.setAlpha(0.85); });
  bg.on('pointerout',   () => { bg.setAlpha(1); });
  bg.on('pointerdown',  () => _draw(true,  false));
  bg.on('pointerup',    () => { _draw(false, false); onTap(); });

  return {
    setDisabled: (v) => { _draw(false, v); if (v) bg.removeInteractive(); else bg.setInteractive(); },
  };
}
```

---

## 3. Touch Target Expansion

Any interactive element smaller than 48×48px needs an invisible expanded hit zone. Failure to do this produces buttons that players miss on mobile, which they blame on the game feeling broken.

```js
// Small icon button — visual is 28×28, hit area is 52×52
_createIconButton(x, y, iconTexture, onTap) {
  // Invisible hit zone — always create this first
  const zone = this.add.zone(x, y, 52, 52)
    .setInteractive()
    .setDepth(DEPTH.HUD);

  // Visual icon
  const icon = this.add.image(x, y, iconTexture)
    .setDisplaySize(28, 28)
    .setDepth(DEPTH.HUD);

  zone.on('pointerdown', () => {
    this.tweens.add({ targets: icon, scaleX: 0.82, scaleY: 0.82, duration: 80, yoyo: true });
    onTap();
  });

  return { zone, icon };
}

// For text-based buttons that are too small:
const btn = this.add.text(x, y, 'SKIP', {
  fontFamily: 'monospace', fontSize: '12px', color: '#888888',
}).setOrigin(0.5).setDepth(DEPTH.HUD);

// Set a minimum hit area regardless of text size
btn.setInteractive(
  new Phaser.Geom.Rectangle(-30, -20, 60, 40),
  Phaser.Geom.Rectangle.Contains
);
```

---

## 4. Health Bar — Segmented and Smooth Variants

Two patterns: segmented (roguelite, showing discrete lives) and smooth (continuous HP pool).

```js
// Smooth HP bar — fills from left, colour-shifts with health
_createHealthBar(x, y, w = 130, h = 10) {
  const bg = this.add.rectangle(x, y, w, h, 0x222222)
    .setOrigin(0, 0.5).setDepth(DEPTH.HUD).setScrollFactor(0);

  const bar = this.add.rectangle(x, y, w, h, 0x22cc66)
    .setOrigin(0, 0.5).setDepth(DEPTH.HUD).setScrollFactor(0);

  // Optional: thin highlight strip along top
  const shine = this.add.rectangle(x, y - 2, w, 2, 0xffffff, 0.12)
    .setOrigin(0, 0.5).setDepth(DEPTH.HUD).setScrollFactor(0);

  return {
    update(current, max) {
      const pct = Math.max(0, Math.min(1, current / max));
      bar.setDisplaySize(w * pct, h);
      shine.setDisplaySize(w * pct, 2);
      bar.setFillStyle(
        pct > 0.6 ? 0x22cc66 :
        pct > 0.3 ? 0xf5a623 :
                    0xee3333
      );
    },
  };
}

// Segmented HP — discrete pips (good for lives / limited HP systems)
_createSegmentedHP(x, y, maxHP) {
  const pips = [];
  const PIP_W = 18, PIP_H = 12, GAP = 5;
  const totalW = maxHP * PIP_W + (maxHP - 1) * GAP;
  const startX = x - totalW / 2;

  for (let i = 0; i < maxHP; i++) {
    const pip = this.add.graphics().setDepth(DEPTH.HUD).setScrollFactor(0);
    const px  = startX + i * (PIP_W + GAP);
    pip._draw = (filled) => {
      pip.clear();
      pip.fillStyle(filled ? 0xee3333 : 0x333333, 1);
      pip.fillRoundedRect(px, y - PIP_H / 2, PIP_W, PIP_H, 4);
      if (filled) {
        pip.fillStyle(0xff6666, 0.4);
        pip.fillRoundedRect(px + 2, y - PIP_H / 2 + 2, PIP_W - 4, 3, 2);
      }
    };
    pip._draw(true);
    pips.push(pip);
  }

  return {
    update(current) {
      pips.forEach((pip, i) => pip._draw(i < current));
    },
  };
}
```

---

## 5. Modal — Blocking Overlay with Backdrop

For anything that requires a player decision before the game can continue: confirmation dialogs, game over, level complete. The backdrop must block input to the game world below.

```js
_showModal({ title, body, primaryLabel, secondaryLabel, onPrimary, onSecondary }) {
  const MODAL_W = 320, MODAL_H = 240;
  const cx = W / 2, cy = H / 2;

  // Backdrop — blocks all input to scene below. Colour/opacity: your choice.
  const backdrop = this.add.rectangle(0, 0, W, H, 0x000000, 0.78)
    .setOrigin(0).setDepth(DEPTH.OVERLAY).setInteractive();   // setInteractive() is mandatory

  // Panel — style to match your game's theme
  const panel = this.add.graphics().setDepth(DEPTH.OVERLAY + 1);
  panel.fillStyle(0x141414, 1);
  panel.fillRoundedRect(cx - MODAL_W / 2, cy - MODAL_H / 2, MODAL_W, MODAL_H, 16);
  panel.lineStyle(1.5, 0x2a2a2a, 1);
  panel.strokeRoundedRect(cx - MODAL_W / 2, cy - MODAL_H / 2, MODAL_W, MODAL_H, 16);

  // Title — font and colour: your choice. Stroke: mandatory.
  const titleTxt = this.add.text(cx, cy - 72, title, {
    fontFamily: 'monospace', fontSize: '18px', fontStyle: 'bold',
    color: '#ffffff', stroke: '#000000', strokeThickness: 4,   // stroke mandatory
  }).setOrigin(0.5).setDepth(DEPTH.OVERLAY + 2);

  // Body — stroke optional on body text, required if over gameplay
  const bodyTxt = this.add.text(cx, cy - 28, body, {
    fontFamily: 'monospace', fontSize: '13px',
    color: '#aaaaaa', stroke: '#000000', strokeThickness: 2,
    wordWrap: { width: MODAL_W - 40 }, align: 'center',
  }).setOrigin(0.5).setDepth(DEPTH.OVERLAY + 2);

  // Pop-in animation
  const elements = [panel, titleTxt, bodyTxt];
  elements.forEach(el => el.setAlpha(0));
  this.tweens.add({ targets: elements, alpha: 1, duration: 180, ease: 'Cubic.Out' });

  const _destroy = () => [backdrop, panel, titleTxt, bodyTxt].forEach(e => e.destroy());

  this._createButton(cx, cy + 60,  primaryLabel,   primaryStyle,   () => { _destroy(); onPrimary?.(); });
  this._createButton(cx, cy + 118, secondaryLabel, secondaryStyle, () => { _destroy(); onSecondary?.(); });
}
```

---

## 6. Toast Notification — Non-Blocking

For feedback that doesn't require a response: "Perfect Room!", "Streak x3", "New Best!". Appears, holds briefly, fades. Never blocks input. Colour and style should fit your game's theme.

```js
_toast(message, colour, duration = 1800) {
  // colour: pass whatever fits the message in your game's palette
  //   e.g. bonus/good news → your accent colour
  //        warning         → your warning colour
  //        neutral info    → white

  const y0 = 140;   // below HUD, above gameplay

  // Background — style to match your game's theme
  const bg = this.add.graphics().setDepth(DEPTH.OVERLAY).setScrollFactor(0);
  bg.fillStyle(0x1a1a1a, 0.92);
  bg.fillRoundedRect(W / 2 - 120, y0 - 20, 240, 40, 10);
  bg.lineStyle(1.5, 0x333333, 1);
  bg.strokeRoundedRect(W / 2 - 120, y0 - 20, 240, 40, 10);

  const txt = this.add.text(W / 2, y0, message, {
    fontFamily: 'monospace', fontSize: '14px', fontStyle: 'bold',
    color:      colour,
    stroke:     '#000000',
    strokeThickness: 3,                // mandatory — toast floats over gameplay
  }).setOrigin(0.5).setDepth(DEPTH.OVERLAY + 1).setScrollFactor(0);

  const targets = [bg, txt];
  targets.forEach(t => { t.setAlpha(0); t.y -= 10; });

  this.tweens.add({
    targets, alpha: 1, y: `+=10`, duration: 220, ease: 'Back.Out',
    onComplete: () => {
      this.time.delayedCall(duration, () => {
        this.tweens.add({
          targets, alpha: 0, y: `-=8`, duration: 300,
          onComplete: () => targets.forEach(t => t.destroy()),
        });
      });
    },
  });
}
```

---

## 7. Progress Bar — Generic (XP, Loading, Boss HP)

A reusable bar for any progress context. Different from the health bar — supports a label and animated fill.

```js
_createProgressBar(x, y, w, h, options = {}) {
  const {
    bgColour    = 0x222222,
    fillColour  = 0xF55018,
    label       = '',
    showPct     = false,
    radius      = 5,
  } = options;

  const bg   = this.add.graphics().setDepth(DEPTH.HUD).setScrollFactor(0);
  const fill = this.add.graphics().setDepth(DEPTH.HUD + 1).setScrollFactor(0);

  bg.fillStyle(bgColour, 1);
  bg.fillRoundedRect(x - w / 2, y - h / 2, w, h, radius);

  let _pct = 0;

  const labelTxt = label ? this.add.text(x, y - h - 6, label, {
    fontFamily: 'monospace', fontSize: '11px',
    color: '#888888', stroke: '#000000', strokeThickness: 2,
  }).setOrigin(0.5).setDepth(DEPTH.HUD).setScrollFactor(0) : null;

  const pctTxt = showPct ? this.add.text(x, y, '0%', {
    fontFamily: 'monospace', fontSize: '10px',
    color: '#ffffff', stroke: '#000000', strokeThickness: 2,
  }).setOrigin(0.5).setDepth(DEPTH.HUD + 2).setScrollFactor(0) : null;

  return {
    // Instant set
    set(pct) {
      _pct = Math.max(0, Math.min(1, pct));
      fill.clear();
      if (_pct > 0) {
        fill.fillStyle(fillColour, 1);
        fill.fillRoundedRect(x - w / 2, y - h / 2, w * _pct, h, radius);
      }
      if (pctTxt) pctTxt.setText(Math.round(_pct * 100) + '%');
    },
    // Animated set
    animateTo(pct, duration = 400) {
      const from = _pct;
      const to   = Math.max(0, Math.min(1, pct));
      this._scene.tweens.addCounter({
        from: from * 100,
        to:   to   * 100,
        duration,
        ease: 'Cubic.Out',
        onUpdate: (t) => this.set(t.getValue() / 100),
      });
    },
  };
}
```

---

## 8. Stat Row — Icon + Label + Value

Used in GameOver screens, upgrade descriptions, and shop panels. Keeps data presentation consistent.

```js
_createStatRow(x, y, label, value, iconColour = 0xF55018) {
  // Coloured dot icon
  this.add.circle(x, y, 4, iconColour)
    .setDepth(DEPTH.OVERLAY);

  // Label
  this.add.text(x + 14, y, label, {
    fontFamily: 'monospace', fontSize: '13px',
    color: '#888888', stroke: '#000000', strokeThickness: 2,
  }).setOrigin(0, 0.5).setDepth(DEPTH.OVERLAY);

  // Value — right-aligned to a fixed column
  this.add.text(x + 220, y, String(value), {
    fontFamily: 'monospace', fontSize: '13px', fontStyle: 'bold',
    color: '#ffffff', stroke: '#000000', strokeThickness: 2,
  }).setOrigin(1, 0.5).setDepth(DEPTH.OVERLAY);
}

// Usage — GameOver stats block (values are examples — use whatever your genre tracks):
const statsY = 480;
this._createStatRow(W / 2 - 110, statsY,       'SCORE',  runState.score);
this._createStatRow(W / 2 - 110, statsY + 28,  'TIME',   runState.time);
this._createStatRow(W / 2 - 110, statsY + 56,  'LEVEL',  runState.level);
```

---

## 9. Pause Menu

Every game needs a pause button and a pause overlay. The pause button lives in the HUD. The overlay uses `scene.pause()` — the game world freezes but stays visible.

```js
// In create() — pause button top-right (below exit button)
_createPauseButton() {
  const btn = this.add.text(W - 16, 56, '⏸', {
    fontFamily: 'monospace', fontSize: '18px', color: '#666666',
  }).setOrigin(1, 0).setDepth(DEPTH.HUD).setScrollFactor(0)
    .setInteractive(new Phaser.Geom.Rectangle(-20, -10, 50, 40), Phaser.Geom.Rectangle.Contains);

  btn.on('pointerdown', () => this._pause());
}

_pause() {
  this.scene.pause('GameScene');
  this.scene.launch('PauseScene', { from: 'GameScene' });
}

// PauseScene — launched over the frozen game
class PauseScene extends Phaser.Scene {
  constructor() { super({ key: 'PauseScene', active: false }); }

  init(data) { this.fromScene = data.from; }

  create() {
    // Backdrop
    this.add.rectangle(0, 0, W, H, 0x000000, 0.65)
      .setOrigin(0).setInteractive();   // block input to scene below

    this.add.text(W / 2, H / 2 - 80, 'PAUSED', {
      fontFamily: 'monospace', fontSize: '28px', fontStyle: 'bold',
      color: '#ffffff', stroke: '#000000', strokeThickness: 5,
    }).setOrigin(0.5);

    // Resume
    this._btn(W / 2, H / 2 + 10, 'RESUME', 'primary', () => {
      this.scene.stop();
      this.scene.resume(this.fromScene);
    });

    // Exit
    this._btn(W / 2, H / 2 + 76, 'EXIT', 'secondary', () => {
      this.scene.stop();
      this.scene.stop(this.fromScene);
      window.gameExit();
    });
  }

  // Inline button helper — use your game's colour scheme
  _btn(x, y, label, isPrimary, cb) {
    // isPrimary: true = your game's primary action colour, false = subdued/outline
    const g = this.add.graphics()
      .setInteractive(new Phaser.Geom.Rectangle(x - 130, y - 26, 260, 52), Phaser.Geom.Rectangle.Contains);
    // Fill with your game's appropriate colour here
    g.fillStyle(isPrimary ? 0x336699 : 0x222222, isPrimary ? 1 : 0.4);
    g.fillRoundedRect(x - 130, y - 26, 260, 52, 13);
    this.add.text(x, y, label, {
      fontFamily: 'monospace', fontSize: '16px', fontStyle: 'bold',
      color: '#ffffff', stroke: '#000000', strokeThickness: 3,
    }).setOrigin(0.5);
    g.on('pointerdown', cb);
  }
}
```

---

## 10. Input Blocking — Prevent Tap-Through

Any overlay that appears over gameplay must block touch input to the game world below. Forgetting this causes players to fire bullets and move the player while interacting with a menu — a common, jarring bug.

```js
// ✅ CORRECT — setInteractive() on the backdrop makes it consume pointer events
const backdrop = this.add.rectangle(0, 0, W, H, 0x000000, 0.6)
  .setOrigin(0)
  .setInteractive();    // ← this line is the blocker, the handler is optional

// ❌ WRONG — non-interactive backdrop, taps pass through to the game world
const backdrop = this.add.rectangle(0, 0, W, H, 0x000000, 0.6)
  .setOrigin(0);        // no setInteractive() — input passes straight through

// For UpgradeScene / PauseScene launched over GameScene:
// The backdrop rectangle must be the FIRST thing added in create()
// and must call setInteractive() — even with no event listeners.
// This is all that's needed to block the scene below.
```

---

## 11. Exit Button — Required in Menus and End Screens

Every game must provide a way to exit back to the Sparkade feed. The exit button belongs in natural pause points where the player has already stopped — not permanently overlaid on gameplay where it wastes screen space and risks accidental taps.

**Where exit must appear:**
- MenuScene — always, as a secondary action
- PauseScene — always, as a secondary action below Resume
- GameOverScene — always, as a secondary action below Play Again

**Where exit must NOT appear:**
- Overlaid on active gameplay at any time

```js
// In MenuScene, GameOverScene, PauseScene — exit as a secondary button
_createExitButton(x, y) {
  const g = this.add.graphics()
    .setDepth(DEPTH.OVERLAY)
    .setInteractive(
      new Phaser.Geom.Rectangle(x - 130, y - 26, 260, 52),
      Phaser.Geom.Rectangle.Contains
    );

  g.fillStyle(0x000000, 0);
  g.fillRoundedRect(x - 130, y - 26, 260, 52, 13);
  g.lineStyle(1.5, 0x2a2a2a, 1);
  g.strokeRoundedRect(x - 130, y - 26, 260, 52, 13);

  this.add.text(x, y, 'EXIT', {
    fontFamily: 'monospace', fontSize: '15px', fontStyle: 'bold',
    color: '#666666', stroke: '#000000', strokeThickness: 3,
  }).setOrigin(0.5).setDepth(DEPTH.OVERLAY + 1);

  g.on('pointerdown', () => {
    window.saveData(this._buildSavePayload?.() || {});
    window.gameExit();
  });
}

// Typical GameOverScene layout — Play Again is primary, Exit is below it
_buildGameOverActions() {
  this._createButton(W / 2, H - 160, 'PLAY AGAIN', 'primary',   () => {
    this.scene.start('GameScene', { runState: this._defaultRunState() });
  });
  this._createExitButton(W / 2, H - 96);
}

// Typical PauseScene layout — Resume is primary, Exit is below it
_buildPauseActions() {
  this._createButton(W / 2, H / 2 + 10, 'RESUME', 'primary', () => {
    this.scene.stop();
    this.scene.resume('GameScene');
  });
  this._createExitButton(W / 2, H / 2 + 76);
}
```

---

## 12. Rarity Badge — Coloured Tier Tag

Used in upgrade screens, item shops, and loot displays. The rarity tier names and colours below are examples — define your own to match your game's theme and vocabulary.

```js
// Define your own rarity tiers and colours to match your game's theme
const RARITY = {
  common:    { colour: 0x888888, label: 'COMMON',    glow: false },
  rare:      { colour: 0x4488ff, label: 'RARE',      glow: false },
  epic:      { colour: 0xcc44ff, label: 'EPIC',      glow: true  },
  legendary: { colour: 0xFFD700, label: 'LEGENDARY', glow: true  },
  // Add, rename, or recolour tiers freely
};

_createRarityBadge(x, y, rarity) {
  const r = RARITY[rarity] || RARITY.common;

  const bg = this.add.graphics().setDepth(DEPTH.OVERLAY);
  bg.fillStyle(r.colour, 0.18);
  bg.fillRoundedRect(x - 42, y - 11, 84, 22, 5);
  bg.lineStyle(1, r.colour, 0.6);
  bg.strokeRoundedRect(x - 42, y - 11, 84, 22, 5);

  this.add.text(x, y, r.label, {
    fontFamily: 'monospace', fontSize: '10px', fontStyle: 'bold',
    color: '#' + r.colour.toString(16).padStart(6, '0'),
    letterSpacing: 1,
  }).setOrigin(0.5).setDepth(DEPTH.OVERLAY + 1);

  if (r.glow) {
    this.tweens.add({
      targets: bg, alpha: { from: 0.7, to: 1 },
      duration: 900, yoyo: true, repeat: -1, ease: 'Sine.InOut',
    });
  }
}
```

---

## Common Failures — Quick Reference

These produce broken UI every time. Note: colours, fonts, and visual style are not in this list — those are creative decisions, not correctness requirements.

| Failure | Fix |
|---------|-----|
| UI scrolls with the camera | `.setScrollFactor(0)` on every HUD element |
| UI hidden behind game objects | `.setDepth(DEPTH.HUD)` or `DEPTH.OVERLAY` — never 0 |
| Taps pass through menu to game | `.setInteractive()` on the backdrop rectangle |
| Button too small to tap on mobile | Minimum 48×48 hit area via `setInteractive(geom)` |
| Text unreadable over game world | `stroke: '#000000', strokeThickness: 4` on all text rendered over gameplay |
| HUD missing on scene restart | HUD elements created in `create()` — they don't persist across `scene.start()` |
| Exit button missing | Must appear in MenuScene, PauseScene, and GameOverScene as an action |
| Exit button on gameplay screen | Wrong — exit belongs in menus only, never overlaid on active gameplay |
