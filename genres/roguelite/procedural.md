# Procedural Generation — Roguelite Sub-Skill

How to generate enemy compositions, room layouts, and loot tables that feel varied and fair at every depth.

---

## Enemy Composition

Never hard-code enemy groups. Generate the composition from a weighted table that shifts with depth.

```js
const ENEMY_TABLE = [
  { type: 'charger', minDepth: 1, weight: (depth) => Math.max(10, 40 - depth * 1.5) },
  { type: 'shooter', minDepth: 2, weight: (depth) => Math.min(35, depth * 2.5) },
  { type: 'dasher',  minDepth: 4, weight: (depth) => Math.min(30, Math.max(0, (depth - 3) * 3)) },
];

_rollEnemyComposition(depth, count) {
  const scalars  = this._getDepthScalars(depth);
  const available = ENEMY_TABLE.filter(e => e.minDepth <= depth);
  const totalWeight = available.reduce((sum, e) => sum + e.weight(depth), 0);
  const result = [];

  for (let i = 0; i < count; i++) {
    let roll = Math.random() * totalWeight;
    for (const entry of available) {
      roll -= entry.weight(depth);
      if (roll < 0) { result.push(entry.type); break; }
    }
  }

  return result;
}
```

At depth 1: mostly chargers. By depth 8: balanced mix of all three. By depth 15+: shooters and dashers dominate.

---

## Spawn Positions

Never spawn enemies on top of the player. Enforce a minimum spawn distance.

```js
_getSpawnPositions(count) {
  const positions = [];
  const minDist = 160;   // minimum distance from player
  const margin  = 40;    // distance from canvas edge
  const maxTries = 50;

  while (positions.length < count) {
    let tries = 0;
    let pos;

    do {
      pos = {
        x: Phaser.Math.Between(margin, 390 - margin),
        y: Phaser.Math.Between(margin + 60, 844 - margin - 80), // avoid HUD and bottom nav
      };
      tries++;
    } while (
      tries < maxTries &&
      (
        Phaser.Math.Distance.Between(pos.x, pos.y, this._player.sprite.x, this._player.sprite.y) < minDist ||
        positions.some(p => Phaser.Math.Distance.Between(p.x, p.y, pos.x, pos.y) < 60)
      )
    );

    positions.push(pos);
  }

  return positions;
}
```

---

## Staged Spawning

Do not spawn all enemies at once. Stagger spawns to give the player time to react.

```js
_spawnEnemies() {
  const scalars     = this._getDepthScalars(this._run.depth);
  const count       = scalars.enemyCount;
  const types       = this._rollEnemyComposition(this._run.depth, count);
  const positions   = this._getSpawnPositions(count);

  types.forEach((type, i) => {
    this.time.delayedCall(i * 220, () => {
      if (!this.scene.isActive()) return;
      const pos = positions[i];
      this._spawnEnemy(type, pos.x, pos.y, scalars);
      this._spawnInEffect(pos.x, pos.y); // visual "arrival" effect
    });
  });
}

_spawnInEffect(x, y) {
  // Brief circle flash at spawn point
  const circle = this.add.graphics();
  circle.lineStyle(2, 0xffffff, 0.6);
  circle.strokeCircle(x, y, 20);
  circle.setDepth(50);
  this.tweens.add({
    targets: circle,
    scaleX: 2.5, scaleY: 2.5, alpha: 0,
    duration: 300, ease: 'Quad.Out',
    onComplete: () => circle.destroy(),
  });
}
```

---

## Boss Design

A boss is a single high-HP enemy with multiple attack phases. Boss runs on a phase state machine.

```js
_spawnBoss() {
  const scalars = this._getDepthScalars(this._run.depth);
  const bossHp  = Math.round(400 * scalars.enemyHp);

  const boss = this.physics.add.sprite(195, 200, 'enemy_boss');
  boss.setDepth(10);
  boss._type      = 'boss';
  boss._hp        = bossHp;
  boss._maxHp     = bossHp;
  boss._damage    = Math.round(25 * scalars.enemyDamage);
  boss._phase     = 1;
  boss._phaseTimer = 0;
  boss._attackCooldown = 0;

  this._boss = boss;
  this._createBossHpBar();

  // Boss intro shake
  this.cameras.main.shake(400, 0.016);
  this.add.text(195, 422, 'BOSS', {
    fontFamily: 'monospace', fontSize: '32px', fontStyle: 'bold',
    color: '#ef4444', align: 'center',
  }).setOrigin(0.5).setDepth(200).setAlpha(1);
}

_updateBoss(delta) {
  if (!this._boss?.active) return;

  this._boss._attackCooldown -= delta;

  // Phase transitions at HP thresholds
  if (this._boss._phase === 1 && this._boss._hp < this._boss._maxHp * 0.5) {
    this._boss._phase = 2;
    this.cameras.main.flash(200, 239, 68, 68); // red flash on phase change
    this.cameras.main.shake(200, 0.012);
  }
  if (this._boss._phase === 2 && this._boss._hp < this._boss._maxHp * 0.25) {
    this._boss._phase = 3;
    this.cameras.main.flash(200, 239, 68, 68);
    this.cameras.main.shake(300, 0.018);
  }

  // Phase 1: circular movement + burst fire
  // Phase 2: faster + spiral fire
  // Phase 3: erratic + rapid fire
  const attacks = {
    1: { moveSpeed: 60, fireRate: 1200, bulletCount: 4 },
    2: { moveSpeed: 90, fireRate: 800,  bulletCount: 6 },
    3: { moveSpeed: 130, fireRate: 500, bulletCount: 8 },
  };

  const atk = attacks[this._boss._phase];

  // Circle the player
  const angle = Phaser.Math.Angle.Between(this._boss.x, this._boss.y, this._player.sprite.x, this._player.sprite.y);
  const perpAngle = angle + Math.PI / 2;
  this._boss.body.setVelocity(Math.cos(perpAngle) * atk.moveSpeed, Math.sin(perpAngle) * atk.moveSpeed);

  // Burst fire
  if (this._boss._attackCooldown <= 0) {
    for (let i = 0; i < atk.bulletCount; i++) {
      const spread = (Math.PI * 2 / atk.bulletCount) * i;
      this._enemyShootAngle(this._boss, spread);
    }
    this._boss._attackCooldown = atk.fireRate;
  }
}

_createBossHpBar() {
  // Prominent boss hp bar at top of screen
  this._bossHpBg  = this.add.rectangle(195, 55, 280, 8, 0x1a1a1a).setDepth(80);
  this._bossHpFill = this.add.rectangle(195 - 140, 55, 280, 8, 0xef4444).setDepth(81).setOrigin(0, 0.5);
  this._bossLabel = this.add.text(195, 68, 'BOSS', {
    fontFamily: 'monospace', fontSize: '9px', color: '#444', letterSpacing: 3,
  }).setOrigin(0.5).setDepth(81);
}

_updateBossHpBar() {
  if (!this._bossHpFill) return;
  const frac = this._boss._hp / this._boss._maxHp;
  this._bossHpFill.width = 280 * frac;
}
```

---

## Loot Table

Health orbs are the only loot drop. Spawn probability scales with depth — harder rooms drop health more often.

```js
_rollPostRoomLoot() {
  const floor = this._run.floor;
  const hpFrac = this._run.hp / this._run.maxHp;

  // Higher chance if low HP, guaranteed if below 20%
  const baseChance = 0.28 + floor * 0.03;
  const lowHpBonus = hpFrac < 0.2 ? 0.4 : hpFrac < 0.4 ? 0.15 : 0;
  const chance = Math.min(0.85, baseChance + lowHpBonus);

  if (Math.random() < chance) {
    this._spawnHealthOrb(
      Phaser.Math.Between(80, 310),
      Phaser.Math.Between(200, 640),
    );
  }
}

_spawnHealthOrb(x, y) {
  const orb = this.physics.add.sprite(x, y, 'health_orb');
  orb.setDepth(5);
  orb._healAmount = Math.round(20 + this._run.floor * 5);

  // Gentle bob
  this.tweens.add({
    targets: orb, y: y - 8,
    duration: 900, yoyo: true, repeat: -1, ease: 'Sine.InOut',
  });

  this.physics.add.overlap(this._player.sprite, orb, () => {
    this._run.hp = Math.min(this._run.maxHp, this._run.hp + orb._healAmount);
    this._updateHUD();
    this.floatText(orb.x, orb.y, `+${orb._healAmount} HP`, '#22c55e');
    this.playSound('pickup');
    orb.destroy();
  });
}
```

---

## Seeded Randomness (Optional)

If reproducible runs are needed for fairness testing, seed Phaser's RNG at run start:

```js
// At start of GameScene.create():
const seed = Date.now(); // or accept from MenuScene
this._rng = new Phaser.Math.RandomDataGenerator([seed.toString()]);

// Use this._rng.between() instead of Phaser.Math.Between()
// Use this._rng.realInRange() instead of Math.random()
```

Not required for production — standard Math.random() is fine for gameplay.
