# Run Structure — Roguelite Sub-Skill

How to architect the run loop, room progression, and scene state for a Sparkade roguelite.

---

## Run State Object

Initialise this at the start of every run in `GameScene.create()`. Pass it between rooms — never reset mid-run.

```js
this._run = {
  depth:       1,       // current room number (1-indexed, increments every room)
  floor:       1,       // floor = Math.ceil(depth / 3)
  kills:       0,       // total kills this run
  score:       0,       // current score
  multiplier:  1,       // score multiplier (upgrades can increase this)
  hp:          100,     // player current hp
  maxHp:       100,     // player max hp
  upgrades:    [],      // array of active upgrade keys
  roomBonus:   true,    // reset to true at room start, false if player took damage
};
```

---

## Room State Machine

Each room has exactly four states:

```
ENTERING → ACTIVE → CLEARED → TRANSITIONING
```

```js
_setState(state) {
  this._roomState = state;

  if (state === 'ENTERING') {
    this._spawnEnemies();
    this._showRoomEntrance();
    this.time.delayedCall(600, () => this._setState('ACTIVE'));
  }

  if (state === 'ACTIVE') {
    // Input enabled, enemies start moving
    this._inputEnabled = true;
  }

  if (state === 'CLEARED') {
    this._inputEnabled = false;
    this._onRoomCleared();
  }

  if (state === 'TRANSITIONING') {
    this._loadNextRoom();
  }
}
```

Check for room cleared in `update()`:
```js
if (this._roomState === 'ACTIVE' && this._enemies.getChildren().every(e => !e.active)) {
  this._setState('CLEARED');
}
```

---

## Room Types

Three room types, weighted by depth:

| Type | Trigger | Weights by depth |
|------|---------|------------------|
| Combat | depth % 3 !== 0 | Always |
| Upgrade | depth % 3 === 0 | Every 3rd room |
| Boss | depth % 9 === 0 | Every 9th room (overrides Upgrade) |

```js
_getRoomType(depth) {
  if (depth % 9 === 0) return 'boss';
  if (depth % 3 === 0) return 'upgrade';
  return 'combat';
}
```

---

## Room Cleared Handler

```js
_onRoomCleared() {
  this._run.depth++;
  this._run.floor = Math.ceil(this._run.depth / 3);

  // Score bonus for clean room
  if (this._run.roomBonus) {
    const bonus = this._run.floor * 200;
    this._run.score += bonus;
    this.floatText(195, 422, `PERFECT +${bonus}`, '#22c55e');
  }

  // Health drop — 30% chance per combat room
  if (Phaser.Math.Between(0, 99) < 30) {
    this._spawnHealthOrb(195, 422);
  }

  const type = this._getRoomType(this._run.depth);

  if (type === 'upgrade') {
    this._showUpgradeScreen();
  } else {
    // Short delay, then transition
    this.time.delayedCall(800, () => this._setState('TRANSITIONING'));
  }
}
```

---

## Room Transition

```js
_loadNextRoom() {
  this.roomTransition(() => {
    // Midpoint of flash — safe to reset room state
    this._clearRoom();
    this._run.roomBonus = true;
    const type = this._getRoomType(this._run.depth);
    if (type === 'boss') {
      this._spawnBoss();
    } else {
      this._setState('ENTERING');
    }
    this._updateHUD();
  });
}

_clearRoom() {
  // Destroy all enemies, bullets, pickups
  this._enemies.clear(true, true);
  this._enemyBullets?.clear(true, true);
  this._pickups?.clear(true, true);
}
```

---

## HUD Layout

Top strip, always visible, never obscured by gameplay:

```
┌─────────────────────────────────────────┐
│  ❤❤❤❤❤      DEPTH 5        12,450      │
└─────────────────────────────────────────┘
```

```js
_createHUD() {
  // Health pips (max 5 visual pips, each represents maxHp/5 hp)
  this._hpPips = [];
  for (let i = 0; i < 5; i++) {
    const pip = this.add.rectangle(18 + i * 18, 36, 12, 12, 0xef4444).setDepth(80);
    this._hpPips.push(pip);
  }

  this._depthText = this.add.text(195, 30, 'DEPTH 1', {
    fontFamily: 'monospace', fontSize: '11px', color: '#555',
    letterSpacing: 3,
  }).setOrigin(0.5).setDepth(80);

  this._scoreText = this.add.text(375, 30, '0', {
    fontFamily: 'monospace', fontSize: '14px', fontStyle: 'bold', color: '#ffffff',
  }).setOrigin(1, 0.5).setDepth(80);
}

_updateHUD() {
  // Update depth
  this._depthText.setText(`DEPTH ${this._run.depth}`);

  // Update HP pips
  const hpFrac = this._run.hp / this._run.maxHp;
  const activePips = Math.ceil(hpFrac * 5);
  this._hpPips.forEach((pip, i) => {
    pip.setFillStyle(i < activePips ? 0xef4444 : 0x2a2a2a);
  });
}
```

---

## Player Death

```js
_playerDeath() {
  if (this._dead) return;
  this._dead = true;

  this._inputEnabled = false;

  // Final score calculation
  this._run.score += this._run.kills * 50;

  this.cameras.main.shake(300, 0.02);
  this.hitFlash(this._player);

  // Slow motion death
  this.physics.world.timeScale = 4;
  this.time.timeScale = 0.25;

  this.time.delayedCall(1200, () => {
    this.physics.world.timeScale = 1;
    this.time.timeScale = 1;
    window.gameOver(this._run.score);
    this.scene.start('GameOver', {
      score:  this._run.score,
      depth:  this._run.depth,
      kills:  this._run.kills,
    });
  });
}
```

---

## Depth Scaling

Difficulty must feel like it's always increasing. Apply these scalars when spawning enemies:

```js
_getDepthScalars(depth) {
  const floor = Math.ceil(depth / 3);
  return {
    enemyHp:      1 + (floor - 1) * 0.3,    // +30% hp per floor
    enemySpeed:   1 + (floor - 1) * 0.12,   // +12% speed per floor
    enemyDamage:  1 + (floor - 1) * 0.2,    // +20% damage per floor
    enemyCount:   2 + Math.floor(floor * 1.2), // more enemies per room
    spawnRate:    Math.max(0.5, 1 - floor * 0.06), // faster spawn interval
  };
}
```

Cap `enemyCount` at 12. Cap `enemySpeed` multiplier at 2.5 — enemies faster than this become unfair on mobile.
