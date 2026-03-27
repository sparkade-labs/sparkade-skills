> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# Obstacles & Difficulty — Runner Sub-Skill

Obstacle generation is where most runner implementations fail. Pure randomness produces impossible patterns — two obstacles with no gap between them at high speed, or a jump obstacle immediately after a duck. Good obstacle generation is not random. It is *constrained random* — random within rules that guarantee fairness.

---

## The Core Rule: Minimum Reaction Time

Every obstacle must be visible and reachable for at least as long as it takes the player to react and execute the required input. At any scroll speed, this is a fixed time window — not a fixed pixel distance.

```js
// Minimum reaction time: 400ms at all scroll speeds
// Minimum gap before an obstacle = scrollSpeed * 0.4 (seconds)

_minGapPx() {
  return this.scrollSpeed * 0.5;   // 500ms of scroll distance — generous margin
}
```

This means at 220 px/sec the minimum gap is 110px. At 680 px/sec the minimum gap is 340px. Gaps scale with speed automatically. **Never hardcode a fixed pixel gap** — it will be fair at one speed and lethal at another.

---

## Obstacle Archetypes

Design exactly three obstacle types. More creates visual noise. Fewer reduces variety too quickly.

**Ground obstacle** — sits on the ground surface, player must jump over. The most common obstacle type. Heights should vary: short obstacles can be jumped with a late tap, tall obstacles require an early tap.

**Aerial obstacle** — floats above the ground at a fixed height, player must duck under. Only include this if your game has a duck mechanic.

**Gap** — a hole in the ground the player must jump across. The gap itself is the obstacle. Width determines difficulty — narrow gaps are forgiving, wide gaps demand precise timing.

Define your obstacle types in BootScene as generated textures. Each must be visually distinct at a glance.

```js
// Example archetype data — heights and visual design are yours to decide
const OBSTACLE_TYPES = [
  {
    key:      'obs_low',        // short ground obstacle — late jump clears it
    type:     'ground',
    w: 40,    h: 55,
    jumpable: true,
    duckable: false,
  },
  {
    key:      'obs_tall',       // tall ground obstacle — early jump required
    type:     'ground',
    w: 38,    h: 95,
    jumpable: true,
    duckable: false,
  },
  {
    key:      'obs_low_aerial', // low ceiling — duck required
    type:     'aerial',
    w: 90,    h: 35,
    yOffset:  -75,              // px above ground surface
    jumpable: false,
    duckable: true,
  },
  // Add or remove types — design to match your theme
];
```

---

## Spawn System — Constrained Random

Never spawn obstacles on a fixed timer. Spawn the next obstacle based on a minimum distance from the last one, with a random additional gap to prevent rhythm-guessing.

```js
// In create():
_setupObstacles() {
  this.obstacleGroup = this.physics.add.group();
  this._lastObstacleX = W + 200;   // first obstacle spawns off-screen right
  this._lastObstacleType = null;
  this._lastAction = null;          // 'jump' or 'duck' — prevents consecutive same action
}

// In update():
_checkSpawn() {
  // Spawn when the last obstacle has scrolled far enough left
  const rightEdge = this._lastObstacleX;   // tracks rightmost obstacle X

  const minGap  = this._minGapPx();
  const maxGap  = minGap * 2.8;            // breathing room — scales with speed
  const spawnAt = W + 60;                  // spawn just off right edge

  if (rightEdge < spawnAt - minGap) {
    const extraGap = Phaser.Math.Between(0, maxGap - minGap);
    if (rightEdge < spawnAt - minGap - extraGap) {
      this._spawnObstacle();
    }
  }
}

_spawnObstacle() {
  const type = this._pickObstacleType();
  if (!type) return;

  const obs = this.obstacleGroup.create(
    W + type.w / 2 + 20,
    this.GROUND_Y - type.h / 2 + (type.yOffset || 0),
    type.key
  );

  obs.body.setImmovable(true);
  obs.body.setAllowGravity(false);
  obs.body.setSize(type.w * 0.75, type.h * 0.85);   // favour-the-player hitbox
  obs._obsType = type;

  this._lastObstacleX = obs.x;
  this._lastObstacleType = type;
  this._lastAction = type.jumpable ? 'jump' : 'duck';
}
```

---

## Obstacle Type Selection — Preventing Impossible Patterns

Random type selection with no constraints produces unplayable sequences. Apply these rules on every spawn:

```js
_pickObstacleType() {
  let candidates = [...OBSTACLE_TYPES];

  // Rule 1: Never require the same action twice in a row at high speed
  // At low speed it's fine — at high speed there's no time to reset
  if (this.scrollSpeed > 420 && this._lastAction) {
    candidates = candidates.filter(t => {
      const action = t.jumpable ? 'jump' : 'duck';
      return action !== this._lastAction;
    });
  }

  // Rule 2: Never spawn a tall obstacle immediately after a gap
  // Player is in the air — they cannot avoid a ground obstacle they land into
  if (this._lastObstacleType?.type === 'gap') {
    candidates = candidates.filter(t => t.type !== 'ground');
  }

  // Rule 3: Introduce obstacle types gradually with depth
  // Don't throw every type at the player in the first 10 seconds
  if (this._elapsed < 10) {
    // Only low ground obstacles in the opening
    candidates = candidates.filter(t => t.key === 'obs_low');
  } else if (this._elapsed < 25) {
    // Add tall obstacles after 10s
    candidates = candidates.filter(t => t.type === 'ground');
  }
  // All types available after 25s

  if (candidates.length === 0) candidates = OBSTACLE_TYPES;

  return Phaser.Utils.Array.GetRandom(candidates);
}
```

---

## Obstacle Cleanup

Obstacles that scroll off the left edge must be destroyed or recycled. Accumulating inactive obstacles causes memory leaks and eventual frame drops.

```js
// In update():
_cleanupObstacles() {
  this.obstacleGroup.getChildren().filter(o => o.active).forEach(obs => {
    if (obs.x + obs.width / 2 < -20) {
      obs.destroy();
    }
  });
}
```

---

## Difficulty Scaling — Obstacle Complexity Over Time

Beyond speed, vary obstacle composition as time increases. This prevents experienced players from going into autopilot.

```js
// Obstacle complexity gates — unlock combinations with time
_getDifficultyTier() {
  if (this._elapsed < 15)  return 'easy';    // single obstacles only
  if (this._elapsed < 40)  return 'medium';  // pairs allowed
  if (this._elapsed < 80)  return 'hard';    // tight pairs, mixed types
  return 'expert';                           // maximum pressure
}

// At 'medium'+ difficulty, occasionally spawn obstacle pairs
// A pair is two obstacles close together that require a specific 2-action sequence
// e.g. jump then immediately duck, or duck then immediately jump
// Pairs must be validated: the gap between them must be exactly enough to
// land from the first action and execute the second

_maybeSpawnPair() {
  if (this._getDifficultyTier() === 'easy') return false;
  return Math.random() < 0.22;   // 22% chance of a pair at medium+
}
```

---

## Visual Telegraphing — Shadow Tells

Every obstacle should cast a shadow on the ground slightly ahead of its position. This gives the player an extra 100–150ms of visual warning — enough to tip fair decisions into consistently fair ones.

```js
// Spawn a ground shadow just before an obstacle appears
_spawnObstacleShadow(obs) {
  const shadow = this.add.ellipse(
    obs.x,
    this.GROUND_Y + 4,
    obs.width * 1.2,
    10,
    0x000000,
    0.3
  ).setDepth(DEPTH.BG_DETAIL);

  // Shadow tracks the obstacle
  obs._shadow = shadow;
}

// In update() — update shadow position with obstacle
_updateShadows() {
  this.obstacleGroup.getChildren().filter(o => o.active && o._shadow).forEach(obs => {
    obs._shadow.x = obs.x;
    if (!obs.active) obs._shadow.destroy();
  });
}
```

---

## Score Multiplier — Rewarding Skilled Play

Layer a multiplier on top of distance score to reward players who clear obstacles cleanly. This creates meaningful score differentiation between players who survive the same time.

```js
// In create():
this._multiplier    = 1;
this._multiplierMax = 5;
this._streak        = 0;   // consecutive obstacles cleared without dying

// Called when an obstacle scrolls fully past the player without collision:
_onObstacleCleared() {
  this._streak++;
  if (this._streak % 5 === 0) {
    // Every 5 cleared, increase multiplier (cap at max)
    this._multiplier = Math.min(this._multiplierMax, this._multiplier + 1);
    this._toast(`x${this._multiplier} MULTIPLIER`, this._accentColour);
  }
}

// On death — streak resets (already ending, but for display):
// this._streak = 0; this._multiplier = 1;

// Apply to distance score:
_updateScore(move) {
  this._distPx += move;
  const newScore = Math.floor((this._distPx / this.PX_PER_PT) * this._multiplier);
  if (newScore !== this.score) {
    this.score = newScore;
    this._updateScoreHUD();
  }
}
```

---

## Collectibles — Optional Layer of Engagement

Collectibles (coins, gems, power-ups) give the player something to chase beyond survival. They should appear in the space between obstacles — never directly before or after one in a way that requires the player to choose between collecting and surviving.

Rules for fair collectible placement:
- Never within `_minGapPx() * 0.5` pixels of an obstacle
- Never in a position that requires a different action than avoiding the nearest obstacle
- Always reachable from a standing position without a jump, OR clearly floating at jump height — never ambiguous

```js
_spawnCollectible(x) {
  // Validate: far enough from last obstacle
  if (Math.abs(x - this._lastObstacleX) < this._minGapPx() * 0.5) return;

  const y = Math.random() < 0.7
    ? this.GROUND_Y - 30                              // ground level — walk into it
    : this.GROUND_Y - (this.JUMP_HEIGHT * 0.55);     // mid-air — requires a jump

  // Spawn and bob in place
  const gem = this.add.image(x, y, 'collectible').setDepth(DEPTH.PICKUPS);
  this.tweens.add({
    targets: gem, y: y - 8, duration: 600, yoyo: true, repeat: -1, ease: 'Sine.InOut',
  });

  // Track for collision and cleanup
  this._collectibles.push(gem);
}
```
