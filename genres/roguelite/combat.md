# Combat — Roguelite Sub-Skill

How to implement tight, responsive, readable combat for a Sparkade roguelite.

---

## Player

### Stats
```js
this._player = {
  sprite:    null,    // Phaser arcade physics sprite
  speed:     180,     // pixels/sec base move speed
  damage:    25,      // base damage per bullet
  fireRate:  280,     // ms between shots
  bulletSpeed: 320,   // pixels/sec
  bulletRange: 340,   // pixels before bullet despawns
  invincible: false,  // true during i-frames after taking damage
};
```

### Invincibility Frames
After taking damage, grant 800ms of invincibility:
```js
_damagePlayer(amount) {
  if (this._player.invincible || this._dead) return;

  this._run.hp = Math.max(0, this._run.hp - amount);
  this._run.roomBonus = false;
  this._updateHUD();
  this.hitFlash(this._player.sprite);
  this.cameras.main.shake(100, 0.008);
  this.knockback(this._player.sprite, this._lastHitX, this._lastHitY, 120);
  this.playSound('hit');

  this._player.invincible = true;
  // Flash sprite during i-frames
  this.tweens.add({
    targets: this._player.sprite,
    alpha: { from: 0.3, to: 1 },
    duration: 100,
    repeat: 3,
    onComplete: () => {
      this._player.sprite.setAlpha(1);
      this._player.invincible = false;
    },
  });

  if (this._run.hp <= 0) this._playerDeath();
}
```

### Auto-aim Shooting
On Sparkade, the player always fires at the nearest enemy automatically. Do not require the player to aim. The right-half tap fires immediately; auto-aim handles direction.

```js
_playerShoot() {
  const nearest = this._getNearestEnemy(this._player.sprite.x, this._player.sprite.y);
  if (!nearest) return;

  const angle  = Phaser.Math.Angle.Between(
    this._player.sprite.x, this._player.sprite.y,
    nearest.x, nearest.y,
  );

  const bullet = this._playerBullets.get(this._player.sprite.x, this._player.sprite.y);
  if (!bullet) return;

  bullet.setActive(true).setVisible(true);
  bullet.setRotation(angle);
  bullet._travelLeft = this._player.bulletRange;
  this.physics.velocityFromAngle(
    Phaser.Math.RadToDeg(angle),
    this._player.bulletSpeed,
    bullet.body.velocity,
  );

  this.playSound('shoot');
}

_getNearestEnemy(x, y) {
  let nearest = null, nearestDist = Infinity;
  this._enemies.getChildren().forEach(e => {
    if (!e.active) return;
    const d = Phaser.Math.Distance.Between(x, y, e.x, e.y);
    if (d < nearestDist) { nearestDist = d; nearest = e; }
  });
  return nearest;
}
```

Auto-fire when the right half is held — use a timer in `update()`:
```js
if (this._actionActive && this._shootCooldown <= 0) {
  this._playerShoot();
  this._shootCooldown = this._player.fireRate;
}
this._shootCooldown = Math.max(0, this._shootCooldown - delta);
```

---

## Enemies

### Three Core Enemy Types

Every roguelite needs three enemy archetypes. Use these as your foundation:

#### 1. Charger (melee)
Rushes directly at the player. Simple but dangerous in groups.

```js
_createCharger(x, y, scalars) {
  const e = this._enemies.create(x, y, 'enemy_charger');
  e.setDepth(10);
  e._type    = 'charger';
  e._hp      = Math.round(40 * scalars.enemyHp);
  e._maxHp   = e._hp;
  e._damage  = Math.round(15 * scalars.enemyDamage);
  e._speed   = 80 * scalars.enemySpeed;
  e._color   = 0xe11d48; // red
  return e;
}

// In update — charger AI
_updateCharger(e, delta) {
  const angle = Phaser.Math.Angle.Between(e.x, e.y, this._player.sprite.x, this._player.sprite.y);
  this.physics.velocityFromAngle(Phaser.Math.RadToDeg(angle), e._speed, e.body.velocity);
  e.setRotation(angle + Math.PI / 2);
}
```

#### 2. Shooter (ranged)
Keeps distance, fires projectiles at the player.

```js
_createShooter(x, y, scalars) {
  const e = this._enemies.create(x, y, 'enemy_shooter');
  e._type       = 'shooter';
  e._hp         = Math.round(30 * scalars.enemyHp);
  e._maxHp      = e._hp;
  e._damage     = Math.round(10 * scalars.enemyDamage);
  e._speed      = 50 * scalars.enemySpeed;
  e._fireRate   = 1800;  // ms between shots
  e._fireCooldown = Phaser.Math.Between(0, 1800); // stagger initial shot
  e._preferredDist = 200; // tries to stay this far from player
  e._color      = 0x7c3aed; // purple
  return e;
}

_updateShooter(e, delta) {
  const dist  = Phaser.Math.Distance.Between(e.x, e.y, this._player.sprite.x, this._player.sprite.y);
  const angle = Phaser.Math.Angle.Between(e.x, e.y, this._player.sprite.x, this._player.sprite.y);

  // Strafe to maintain preferred distance
  if (dist < e._preferredDist - 20) {
    // Too close — back away
    this.physics.velocityFromAngle(Phaser.Math.RadToDeg(angle) + 180, e._speed, e.body.velocity);
  } else if (dist > e._preferredDist + 40) {
    // Too far — approach
    this.physics.velocityFromAngle(Phaser.Math.RadToDeg(angle), e._speed * 0.8, e.body.velocity);
  } else {
    // At range — strafe sideways
    this.physics.velocityFromAngle(Phaser.Math.RadToDeg(angle) + 90, e._speed * 0.6, e.body.velocity);
  }

  // Fire
  e._fireCooldown -= delta;
  if (e._fireCooldown <= 0) {
    this._enemyShoot(e, angle);
    e._fireCooldown = e._fireRate;
  }
}
```

#### 3. Dasher (mobile melee)
Idles briefly, then dashes rapidly at player position. Telegraphed with a windup.

```js
_createDasher(x, y, scalars) {
  const e = this._enemies.create(x, y, 'enemy_dasher');
  e._type       = 'dasher';
  e._hp         = Math.round(55 * scalars.enemyHp);
  e._maxHp      = e._hp;
  e._damage     = Math.round(22 * scalars.enemyDamage);
  e._speed      = 240 * scalars.enemySpeed;
  e._state      = 'idle'; // idle → windup → dashing → recovery
  e._stateTimer = Phaser.Math.Between(800, 1600);
  e._color      = 0xf59e0b; // amber
  return e;
}

_updateDasher(e, delta) {
  e._stateTimer -= delta;

  if (e._state === 'idle' && e._stateTimer <= 0) {
    // Telegraph windup — tint white
    e.setTintFill(0xffffff);
    e._state = 'windup';
    e._stateTimer = 400;
    e._dashTarget = { x: this._player.sprite.x, y: this._player.sprite.y };
    e.body.setVelocity(0, 0);
  }
  else if (e._state === 'windup' && e._stateTimer <= 0) {
    e.clearTint();
    e._state = 'dashing';
    e._stateTimer = 300;
    const angle = Phaser.Math.Angle.Between(e.x, e.y, e._dashTarget.x, e._dashTarget.y);
    this.physics.velocityFromAngle(Phaser.Math.RadToDeg(angle), e._speed, e.body.velocity);
  }
  else if (e._state === 'dashing' && e._stateTimer <= 0) {
    e.body.setVelocity(0, 0);
    e._state = 'recovery';
    e._stateTimer = 600;
  }
  else if (e._state === 'recovery' && e._stateTimer <= 0) {
    e._state = 'idle';
    e._stateTimer = Phaser.Math.Between(1000, 2000);
  }
}
```

---

## Damage Application

```js
_damageEnemy(enemy, amount) {
  enemy._hp -= amount;
  this.hitFlash(enemy);
  this.floatText(enemy.x, enemy.y - 20, `-${amount}`, '#ffffff');

  // Update enemy health bar (see below)
  this._updateEnemyHpBar(enemy);

  if (enemy._hp <= 0) {
    this._killEnemy(enemy);
  }
}

_killEnemy(enemy) {
  this.hitstop(60);
  this.spawnDeathBurst(enemy.x, enemy.y, enemy._color);
  this.playSound('die');

  this._run.kills++;
  const scoreGain = 50 * this._run.floor;
  this._run.score += scoreGain;
  this.updateScore(this._run.score);
  this.floatText(enemy.x, enemy.y, `+${scoreGain}`, '#F55018');

  enemy.destroy();
}
```

---

## Enemy Health Bars

Small health bar above each enemy. Only show when damaged.

```js
_createEnemyHpBar(enemy) {
  const bar = this.add.graphics().setDepth(15);
  enemy._hpBar = bar;
  enemy._hpBarVisible = false;
  this._updateEnemyHpBar(enemy);
}

_updateEnemyHpBar(enemy) {
  if (!enemy._hpBar) return;
  const w = 28, h = 3;
  const x = enemy.x - w / 2;
  const y = enemy.y - 20;
  const frac = enemy._hp / enemy._maxHp;

  enemy._hpBar.clear();
  enemy._hpBar.fillStyle(0x1a1a1a);
  enemy._hpBar.fillRect(x, y, w, h);
  enemy._hpBar.fillStyle(frac > 0.5 ? 0x22c55e : frac > 0.25 ? 0xf59e0b : 0xef4444);
  enemy._hpBar.fillRect(x, y, w * frac, h);
  enemy._hpBar.setDepth(15);
}
```

Update bar position in `update()` since enemies move:
```js
this._enemies.getChildren().forEach(e => {
  if (e._hpBar) {
    e._hpBar.x = 0; e._hpBar.y = 0; // bars use absolute coords, updated in _updateEnemyHpBar
    this._updateEnemyHpBar(e);
  }
});
```

---

## Collision Setup

```js
_setupCollisions() {
  // Player bullets → enemies
  this.physics.add.overlap(this._playerBullets, this._enemies, (bullet, enemy) => {
    bullet.setActive(false).setVisible(false).body.reset(0, 0);
    this._damageEnemy(enemy, this._player.damage);
  });

  // Enemy contact → player
  this.physics.add.overlap(this._player.sprite, this._enemies, (player, enemy) => {
    this._lastHitX = enemy.x;
    this._lastHitY = enemy.y;
    this._damagePlayer(enemy._damage);
  });

  // Enemy bullets → player
  if (this._enemyBullets) {
    this.physics.add.overlap(this._player.sprite, this._enemyBullets, (player, bullet) => {
      this._lastHitX = bullet.x;
      this._lastHitY = bullet.y;
      bullet.setActive(false).setVisible(false).body.reset(0, 0);
      this._damagePlayer(bullet._damage || 10);
    });
  }
}
```
