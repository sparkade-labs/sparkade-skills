> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# Player Feel — Runner Sub-Skill

The jump is the entire game. Every other system exists to give the jump meaning. A jump that feels floaty, imprecise, or unresponsive will kill the game regardless of how well everything else is built.

---

## The Jump — Non-Negotiables

**Gravity must be high.** Floaty jumps — where the player hangs in the air — feel pleasant in platformers but terrible in runners. The player needs to feel like they are landing quickly and getting back to the ground. High gravity with a strong initial impulse is the correct feel.

**Jump height must be consistent and predictable.** The player will develop muscle memory after 2–3 runs. If jump height varies based on anything other than deliberate tap-hold mechanics, that muscle memory breaks and deaths feel unfair.

**Landing must be instant.** No slide, no landing animation delay, no period where the player cannot jump after landing. The frame the player touches the ground is the frame they can jump again.

```js
// In create():
_setupPlayer() {
  this.player = this.physics.add.sprite(this.PLAYER_X, 0, 'player');
  this.player.setCollideWorldBounds(true);

  // Physics body slightly smaller than visual — favour-the-player hitbox
  this.player.body.setSize(
    this.player.width  * 0.7,
    this.player.height * 0.85
  );

  // High gravity — critical for tight runner feel
  this.player.body.setGravityY(1800);   // additional gravity on top of world gravity

  // Ground collision
  this.physics.add.collider(this.player, this.groundChunks, () => {
    this._onLand();
  });

  this._onGround  = false;
  this._jumpCount = 0;
  this.JUMP_VEL   = -720;     // strong upward impulse — adjust to feel
  this.MAX_JUMPS  = 1;        // 1 = single jump, 2 = double jump
}

_onLand() {
  this._onGround  = true;
  this._jumpCount = 0;
}
```

---

## Jump Input — Tap Anywhere on Screen

For a runner, the entire screen is the jump button. Do not restrict to a specific zone — the player's thumb is wherever it is, and they should not have to think about where to tap.

```js
// In create():
_setupInput() {
  // Touch — whole screen
  this.input.on('pointerdown', () => this._tryJump());

  // Keyboard — space, up, W (desktop testing)
  this._keys = this.input.keyboard.addKeys({
    space: 'SPACE', up: 'UP', w: 'W',
  });
}

// In update():
_handleInput() {
  if (
    Phaser.Input.Keyboard.JustDown(this._keys.space) ||
    Phaser.Input.Keyboard.JustDown(this._keys.up)    ||
    Phaser.Input.Keyboard.JustDown(this._keys.w)
  ) {
    this._tryJump();
  }
}

_tryJump() {
  if (this._dead) return;

  // Coyote time — see below
  const canJump = this._onGround || this._coyoteActive || this._jumpCount < this.MAX_JUMPS;
  if (!canJump) return;

  this.player.body.setVelocityY(this.JUMP_VEL);
  this._onGround    = false;
  this._coyoteActive = false;
  this._jumpCount++;

  this._playJumpAnim();
  this._playSound('jump');

  // Particle burst at feet on jump (optional but satisfying)
  this._spawnJumpDust();
}
```

---

## Coyote Time — Mandatory

Coyote time is a brief window after walking off a ledge during which the player can still jump. Without it, players who tap slightly late at a ledge edge die and feel cheated. With it, the game feels fair and responsive. 120ms is the standard window.

This is especially important for runners where the player is making split-second decisions — the margin for error must favour the player.

```js
// In create():
this._coyoteActive = false;
this.COYOTE_MS     = 120;

// In update() — track the moment the player leaves the ground
_checkCoyote() {
  if (!this._onGround && this.player.body.velocity.y > 0 && !this._coyoteTimer) {
    // Player just left the ground without jumping
    this._coyoteActive = true;
    this._coyoteTimer  = this.time.delayedCall(this.COYOTE_MS, () => {
      this._coyoteActive = false;
      this._coyoteTimer  = null;
    });
  }
  // Clear coyote if they actually jump
  if (this._jumpCount > 0) {
    this._coyoteActive = false;
  }
}

// In the physics collider callback:
_onLand() {
  this._onGround = true;
  this._jumpCount = 0;
  if (this._coyoteTimer) {
    this._coyoteTimer.remove();
    this._coyoteTimer  = null;
  }
  this._coyoteActive = false;
}
```

---

## Duck / Slide — Optional Second Action

If your game includes ducking under obstacles, this is the second and final input. One finger jump, one finger duck. Never add a third action.

```js
// In create():
this._ducking     = false;
this.DUCK_MS      = 600;       // minimum duck duration before auto-rise

// Duck hitbox — smaller than standing
this.STAND_SIZE   = { w: this.player.width * 0.7,  h: this.player.height * 0.85 };
this.DUCK_SIZE    = { w: this.player.width * 0.85, h: this.player.height * 0.45 };

// Input — second pointer (right side of screen) OR swipe down
this.input.on('pointerdown', (p) => {
  // If your game uses separate tap zones:
  if (p.x > W * 0.5) this._tryDuck();
  else this._tryJump();
});

_tryDuck() {
  if (this._dead || this._ducking || !this._onGround) return;
  this._ducking = true;
  this.player.body.setSize(this.DUCK_SIZE.w, this.DUCK_SIZE.h);
  // Shift body offset so feet stay on ground
  this.player.body.setOffset(0, this.player.height - this.DUCK_SIZE.h);

  this._playDuckAnim();

  this.time.delayedCall(this.DUCK_MS, () => this._standUp());
}

_standUp() {
  if (!this._ducking) return;
  this._ducking = false;
  this.player.body.setSize(this.STAND_SIZE.w, this.STAND_SIZE.h);
  this.player.body.setOffset(0, 0);
  this._playRunAnim();
}
```

---

## Player Animation — Tween-Based

Since all textures are procedural, use tweens to suggest animation rather than frame-by-frame sprites. These three states are all a runner needs.

```js
// Running — continuous idle bob
_playRunAnim() {
  this.tweens.killTweensOf(this.player);
  this.tweens.add({
    targets:  this.player,
    scaleY:   { from: 1, to: 0.94 },
    duration: 180,
    yoyo:     true,
    repeat:   -1,
    ease:     'Sine.InOut',
  });
}

// Jumping — stretch upward on launch, squash on land
_playJumpAnim() {
  this.tweens.killTweensOf(this.player);
  // Stretch on launch
  this.tweens.add({
    targets:  this.player,
    scaleX:   0.78,
    scaleY:   1.28,
    duration: 80,
    ease:     'Cubic.Out',
    onComplete: () => {
      // Return to normal during arc
      this.tweens.add({
        targets: this.player, scaleX: 1, scaleY: 1, duration: 140,
      });
    },
  });
}

// Landing — squash, then snap back
_playLandAnim() {
  this.tweens.killTweensOf(this.player);
  this.tweens.add({
    targets:  this.player,
    scaleX:   1.35,
    scaleY:   0.72,
    duration: 55,
    ease:     'Cubic.Out',
    onComplete: () => {
      this.tweens.add({
        targets: this.player, scaleX: 1, scaleY: 1,
        duration: 110, ease: 'Back.Out',
        onComplete: () => this._playRunAnim(),
      });
    },
  });
}

// Landing — call from the collider
_onLand() {
  const wasAirborne = !this._onGround;
  this._onGround  = true;
  this._jumpCount = 0;
  if (wasAirborne) this._playLandAnim();
}
```

---

## Jump Dust — Ground Particle Effect

A small burst of particles at the player's feet on jump and landing costs nothing and adds significant perceived polish.

```js
_spawnJumpDust() {
  // Colour match to your ground — should look like disturbed ground material
  this.add.particles(this.player.x, this.GROUND_Y, 'pixel', {
    speed:    { min: 30, max: 90 },
    angle:    { min: 190, max: 350 },   // spray downward and to the sides
    scale:    { start: 2.5, end: 0 },
    alpha:    { start: 0.7, end: 0 },
    lifespan: 260,
    quantity: 8,
    emitting: false,
  }).explode(8);
}
```

---

## Death — Collision Handling

Death must be immediate but feel fair. Two rules: the hitbox must be smaller than the visual (see above), and there must be no ambiguity — the player sees exactly what hit them.

```js
// In create():
this.physics.add.overlap(
  this.player,
  this.obstacleGroup,
  (player, obstacle) => {
    if (this._dead || this._iFrames) return;
    this._die();
  }
);

_die() {
  this._dead = true;
  this.scrollSpeed = 0;          // freeze world immediately

  // Visual feedback — from patterns.md Pattern 10
  this.cameras.main.shake(300, 0.018);
  this._spawnDeathBurst(this.player.x, this.player.y, this._playerColour);
  this.player.setVisible(false);

  // Hitstop then transition — from patterns.md Pattern 20
  this._hitstop(120);

  this.time.delayedCall(900, () => {
    window.gameOver(this.score);
    this.cameras.main.fadeOut(350);
    this.time.delayedCall(370, () => {
      this.scene.start('GameOverScene', { score: this.score });
    });
  });
}
```
