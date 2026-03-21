# Upgrades — Roguelite Sub-Skill

How to design and implement the upgrade selection system. This is the primary source of player agency — make it feel impactful.

---

## Upgrade Data

Define all upgrades as a flat array of objects. Each upgrade has a unique `key`, a `label`, a short `desc`, and an `apply` function that mutates the player stats object.

```js
const UPGRADES = [
  {
    key:   'damage_up',
    label: 'OVERCHARGE',
    desc:  'Bullets deal 40% more damage.',
    rarity: 'common',
    apply: (player) => { player.damage = Math.round(player.damage * 1.4); },
  },
  {
    key:   'fire_rate_up',
    label: 'RAPID FIRE',
    desc:  'Fire rate increased by 30%.',
    rarity: 'common',
    apply: (player) => { player.fireRate = Math.round(player.fireRate * 0.7); },
  },
  {
    key:   'speed_up',
    label: 'AFTERBURNER',
    desc:  'Move speed increased by 25%.',
    rarity: 'common',
    apply: (player) => { player.speed = Math.round(player.speed * 1.25); },
  },
  {
    key:   'bullet_speed_up',
    label: 'HYPERSONIC',
    desc:  'Bullets travel 50% faster and further.',
    rarity: 'common',
    apply: (player) => {
      player.bulletSpeed = Math.round(player.bulletSpeed * 1.5);
      player.bulletRange = Math.round(player.bulletRange * 1.3);
    },
  },
  {
    key:   'pierce',
    label: 'PIERCING',
    desc:  'Bullets pierce through enemies.',
    rarity: 'rare',
    apply: (player) => { player.pierce = true; },
  },
  {
    key:   'multishot',
    label: 'SPREAD SHOT',
    desc:  'Fire 3 bullets in a spread.',
    rarity: 'rare',
    apply: (player) => { player.multishot = (player.multishot || 1) + 2; },
  },
  {
    key:   'lifesteal',
    label: 'VAMPIRIC',
    desc:  'Recover 3 HP per kill.',
    rarity: 'rare',
    apply: (player) => { player.lifesteal = (player.lifesteal || 0) + 3; },
  },
  {
    key:   'max_hp_up',
    label: 'REINFORCED',
    desc:  'Max HP +40. Heal to full.',
    rarity: 'common',
    apply: (player, run) => {
      run.maxHp += 40;
      run.hp = run.maxHp;
    },
  },
  {
    key:   'score_multiplier',
    label: 'HIGHROLLER',
    desc:  'Score multiplier +1.',
    rarity: 'rare',
    apply: (player, run) => { run.multiplier += 1; },
  },
  {
    key:   'explosion',
    label: 'CLUSTER',
    desc:  'Kills cause a small explosion.',
    rarity: 'epic',
    apply: (player) => { player.explosion = true; },
  },
  {
    key:   'dash',
    label: 'BLINK',
    desc:  'Tap right side to dash. Brief invincibility.',
    rarity: 'epic',
    apply: (player) => { player.hasDash = true; player.dashCooldown = 0; },
  },
];
```

---

## Rarity Weights

```js
const RARITY_WEIGHTS = { common: 60, rare: 30, epic: 10 };

function weightedRarityPick(pool) {
  const total = pool.reduce((sum, u) => sum + RARITY_WEIGHTS[u.rarity], 0);
  let roll = Phaser.Math.Between(0, total - 1);
  for (const u of pool) {
    roll -= RARITY_WEIGHTS[u.rarity];
    if (roll < 0) return u;
  }
  return pool[pool.length - 1];
}
```

---

## Rolling 3 Upgrade Choices

Never offer the same upgrade twice in one pick. Exclude upgrades that are already maxed or incompatible.

```js
_rollUpgradeChoices() {
  const equipped = new Set(this._run.upgrades);
  
  // Exclude upgrades the player already has (allow stacking for stat boosts, not abilities)
  const abilityKeys = new Set(['pierce', 'multishot', 'lifesteal', 'explosion', 'dash']);
  const available = UPGRADES.filter(u => !(abilityKeys.has(u.key) && equipped.has(u.key)));

  const choices = [];
  const used = new Set();

  for (let i = 0; i < 3; i++) {
    const pool = available.filter(u => !used.has(u.key));
    if (!pool.length) break;
    const pick = weightedRarityPick(pool);
    choices.push(pick);
    used.add(pick.key);
  }

  return choices;
}
```

---

## Upgrade Screen

Displayed as an overlay over the game world (not a separate scene). The world is paused during selection.

```js
_showUpgradeScreen() {
  const choices = this._rollUpgradeChoices();

  // Dim world
  const overlay = this.add.rectangle(195, 422, 390, 844, 0x000000, 0.75).setDepth(200);
  this.add.text(195, 160, 'CHOOSE UPGRADE', {
    fontFamily: 'monospace', fontSize: '12px', color: '#555', letterSpacing: 4,
  }).setOrigin(0.5).setDepth(201);

  const cards = [];

  choices.forEach((upgrade, i) => {
    const cardY = 280 + i * 150;
    const rarity = upgrade.rarity;
    const borderColor = { common: 0x334155, rare: 0x7c3aed, epic: 0xf59e0b }[rarity];
    const labelColor  = { common: '#94a3b8', rare: '#a78bfa', epic: '#fbbf24' }[rarity];

    // Card background
    const card = this.add.graphics().setDepth(201);
    card.fillStyle(0x111111);
    card.fillRoundedRect(195 - 150, cardY - 55, 300, 110, 12);
    card.lineStyle(1.5, borderColor);
    card.strokeRoundedRect(195 - 150, cardY - 55, 300, 110, 12);

    // Rarity badge
    this.add.text(195 - 134, cardY - 38, rarity.toUpperCase(), {
      fontFamily: 'monospace', fontSize: '9px', color: labelColor, letterSpacing: 3,
    }).setDepth(202);

    // Upgrade name
    this.add.text(195, cardY - 12, upgrade.label, {
      fontFamily: 'monospace', fontSize: '18px', fontStyle: 'bold', color: '#ffffff',
    }).setOrigin(0.5).setDepth(202);

    // Description
    this.add.text(195, cardY + 22, upgrade.desc, {
      fontFamily: 'monospace', fontSize: '11px', color: '#888', align: 'center', wordWrap: { width: 260 },
    }).setOrigin(0.5).setDepth(202);

    // Hit zone
    const hitZone = this.add.rectangle(195, cardY, 300, 110, 0xffffff, 0)
      .setDepth(203).setInteractive({ useHandCursor: true });

    hitZone.on('pointerover', () => { card.clear(); card.fillStyle(0x1a1a1a); card.fillRoundedRect(195 - 150, cardY - 55, 300, 110, 12); card.lineStyle(2, borderColor); card.strokeRoundedRect(195 - 150, cardY - 55, 300, 110, 12); });
    hitZone.on('pointerout',  () => { card.clear(); card.fillStyle(0x111111); card.fillRoundedRect(195 - 150, cardY - 55, 300, 110, 12); card.lineStyle(1.5, borderColor); card.strokeRoundedRect(195 - 150, cardY - 55, 300, 110, 12); });

    hitZone.on('pointerdown', () => {
      // Apply upgrade
      upgrade.apply(this._player, this._run);
      this._run.upgrades.push(upgrade.key);

      this.playSound('levelup');

      // Destroy upgrade UI
      this.children.list
        .filter(c => c.depth >= 200)
        .forEach(c => c.destroy());

      // Bounce the player sprite as feedback
      this.tweens.add({
        targets: this._player.sprite,
        scaleX: 1.4, scaleY: 1.4,
        duration: 120, yoyo: true, ease: 'Back.Out',
      });

      // Proceed to next room
      this.time.delayedCall(300, () => this._setState('TRANSITIONING'));
    });

    cards.push({ card, hitZone });
  });

  // Animate cards in
  cards.forEach(({ card }, i) => {
    card.setAlpha(0);
    this.tweens.add({ targets: card, alpha: 1, y: -20, duration: 200, delay: i * 80, ease: 'Quad.Out' });
  });
}
```

---

## Applying Upgrade Effects In-Game

Upgrade effects are applied at selection time by mutating `this._player`. The game loop reads these stats dynamically — no additional wiring needed for stat-based upgrades.

For ability upgrades, check flags in the relevant game logic:

```js
// In _playerShoot():
if (this._player.multishot > 1) {
  const spread = 18; // degrees
  for (let i = 0; i < this._player.multishot; i++) {
    const angleOffset = (i - Math.floor(this._player.multishot / 2)) * spread;
    this._fireBullet(baseAngle + Phaser.Math.DegToRad(angleOffset));
  }
} else {
  this._fireBullet(baseAngle);
}

// In _killEnemy():
if (this._player.lifesteal > 0) {
  this._run.hp = Math.min(this._run.maxHp, this._run.hp + this._player.lifesteal);
  this._updateHUD();
}

if (this._player.explosion) {
  this._spawnExplosion(enemy.x, enemy.y);
}
```
