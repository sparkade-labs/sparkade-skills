# Sparkade Platform Integration — Core Skill

How your game communicates with the Sparkade platform shell. This is non-negotiable — every game must implement this correctly.

---

## Message Protocol

The game shell (`game-shell.html`) establishes a `postMessage` bridge between your game and the platform. These functions are injected into `window` by the shell before your game loads:

### Outbound (your game → platform)

```js
window.gameStarted()        // Call when the player starts a run (not on menu show)
window.gameOver(score)      // Call at run end. score must be a positive integer
window.gameExit()           // Call when the player taps "Exit" voluntarily
```

**Critical timing rules:**
- `gameStarted()` — call exactly once, when gameplay actually begins (first enemy spawns / first input registered). Never call on menu load.
- `gameOver(score)` — call exactly once per run, immediately when the player dies or the run ends. `score` must be a non-negative integer. Floats will be floored server-side.
- `gameExit()` — only call from an explicit exit button. Do not call on game over.

### Inbound (platform → your game)

The platform posts back after score submission. Implement these on `window`:

```js
window.onScoreResult = function(sparksEarned, isPersonalBest, rank, prevBest) {
  // sparksEarned  — integer, Sparks currency rewarded this run
  // isPersonalBest — boolean
  // rank           — integer or null (leaderboard position)
  // prevBest       — integer or null (previous personal best score)
  
  // Update your GameOver screen with this data
  // Show sparksEarned if > 0
  // Show "NEW BEST" badge if isPersonalBest
};
```

Register `onScoreResult` in `GameOverScene.create()`, before the scene renders. The platform sends `SCORE_RESULT` shortly after `gameOver()` — you may receive it while the scene is still transitioning.

---

## Game Lifecycle Flow

```
Shell loads → Phaser boots → BootScene → MenuScene
                                              │
                             Player taps "Start"
                                              │
                                    window.gameStarted()
                                              │
                                         GameScene
                                              │
                                       Player dies
                                              │
                                    window.gameOver(score)
                                              │
                                       GameOverScene
                                    window.onScoreResult(...)
                                              │
                          ┌───────────────────┴──────────────────┐
                    "Play Again"                              "Exit"
                          │                                       │
                     GameScene                        window.gameExit()
```

---

## Complete GameOverScene Template

```js
class GameOverScene extends Phaser.Scene {
  constructor() { super('GameOver'); }

  create(data) {
    const { score = 0, depth = 1, kills = 0 } = data;
    const cx = 390 / 2;

    // Background
    this.add.rectangle(0, 0, 390, 844, 0x080808).setOrigin(0);

    // Score display
    this.add.text(cx, 280, score.toString(), {
      fontFamily: 'monospace', fontSize: '72px', fontStyle: 'bold',
      color: '#F55018', align: 'center',
    }).setOrigin(0.5);

    this.add.text(cx, 340, 'SCORE', {
      fontFamily: 'monospace', fontSize: '11px',
      color: '#444', letterSpacing: 4,
    }).setOrigin(0.5);

    // Sparks / PB container — populated by onScoreResult
    this._rewardText = this.add.text(cx, 400, '', {
      fontFamily: 'monospace', fontSize: '14px', color: '#F55018', align: 'center',
    }).setOrigin(0.5).setAlpha(0);

    this._pbBadge = this.add.text(cx, 430, 'NEW BEST', {
      fontFamily: 'monospace', fontSize: '11px', color: '#fff',
      backgroundColor: '#F55018', padding: { x: 10, y: 4 },
    }).setOrigin(0.5).setAlpha(0);

    // Register score result handler
    window.onScoreResult = (sparks, pb, rank, prevBest) => {
      if (sparks > 0) {
        this._rewardText.setText(`+${sparks} SPARKS`).setAlpha(1);
        this.tweens.add({ targets: this._rewardText, y: 390, alpha: 1, duration: 300, ease: 'Back.Out' });
      }
      if (pb) {
        this._pbBadge.setAlpha(1);
        this.tweens.add({ targets: this._pbBadge, scaleX: [0, 1], scaleY: [0, 1], duration: 350, ease: 'Back.Out', delay: 150 });
      }
    };

    // Run stats
    this.add.text(cx, 500, `DEPTH  ${depth}\nKILLS  ${kills}`, {
      fontFamily: 'monospace', fontSize: '13px', color: '#555',
      align: 'center', lineSpacing: 10,
    }).setOrigin(0.5);

    // Play Again button
    const playBtn = this.add.text(cx, 640, 'PLAY AGAIN', {
      fontFamily: 'monospace', fontSize: '14px', fontStyle: 'bold',
      color: '#080808', backgroundColor: '#F55018',
      padding: { x: 28, y: 14 },
    }).setOrigin(0.5).setInteractive({ useHandCursor: true });

    playBtn.on('pointerdown', () => this.scene.start('Game'));

    // Exit button
    const exitBtn = this.add.text(cx, 710, 'EXIT', {
      fontFamily: 'monospace', fontSize: '12px', color: '#444',
    }).setOrigin(0.5).setInteractive({ useHandCursor: true });

    exitBtn.on('pointerdown', () => window.gameExit());
  }
}
```

---

## Mobile Input Setup

Sparkade is mobile-first. Always implement a virtual joystick for movement and tap zones for actions.

```js
// In GameScene.create():
this._setupTouch();

_setupTouch() {
  // Left half = movement joystick
  // Right half = action (shoot/dodge)
  
  this._joystick     = { active: false, startX: 0, startY: 0, dx: 0, dy: 0 };
  this._actionActive = false;

  this.input.on('pointerdown', (p) => {
    if (p.x < 195) {
      this._joystick = { active: true, startX: p.x, startY: p.y, dx: 0, dy: 0, id: p.id };
      this._drawJoystick(p.x, p.y, 0, 0);
    } else {
      this._actionActive = true;
    }
  });

  this.input.on('pointermove', (p) => {
    if (this._joystick.active && p.id === this._joystick.id) {
      const dx = Phaser.Math.Clamp(p.x - this._joystick.startX, -48, 48);
      const dy = Phaser.Math.Clamp(p.y - this._joystick.startY, -48, 48);
      this._joystick.dx = dx / 48;
      this._joystick.dy = dy / 48;
      this._drawJoystick(this._joystick.startX, this._joystick.startY, dx, dy);
    }
  });

  this.input.on('pointerup', (p) => {
    if (p.id === this._joystick?.id) {
      this._joystick.active = false;
      this._joystick.dx     = 0;
      this._joystick.dy     = 0;
      this._joystickGfx?.clear();
    } else {
      this._actionActive = false;
    }
  });
}

_drawJoystick(bx, by, dx, dy) {
  if (!this._joystickGfx) this._joystickGfx = this.add.graphics().setDepth(100).setAlpha(0.35);
  this._joystickGfx.clear();
  this._joystickGfx.lineStyle(2, 0xffffff);
  this._joystickGfx.strokeCircle(bx, by, 48);
  this._joystickGfx.fillStyle(0xffffff);
  this._joystickGfx.fillCircle(bx + dx, by + dy, 18);
}
```

In `update()`:
```js
if (this._joystick.active) {
  const speed = 180;
  this.player.body.setVelocity(
    this._joystick.dx * speed,
    this._joystick.dy * speed,
  );
} else {
  this.player.body.setVelocity(0, 0);
}
```

Also support keyboard for desktop testing:
```js
this._keys = this.input.keyboard.addKeys('W,A,S,D,UP,DOWN,LEFT,RIGHT,SPACE');
```

Merge keyboard and joystick inputs in update — keyboard takes priority when both active.
