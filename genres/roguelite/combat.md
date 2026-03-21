# Combat — Roguelite Sub-Skill

Principles for tight, readable, satisfying combat. The specific implementation is yours — these are the standards every implementation must meet.

---

## On-Screen Controls

The roguelite uses twin-stick style input: one control for movement, one for shooting. Both must be permanently visible on screen as rendered UI elements — not invisible zones.

**Movement control (typically bottom-left):** A joystick with a fixed base position is strongly preferred over a floating joystick that spawns at touch origin. Fixed position builds muscle memory. Floating joysticks disorient players because the thumb never lands in the same place twice.

**Fire control (typically bottom-right):** A visible button with clear pressed/unpressed states. Auto-fire toward the nearest enemy when held — the player should not need to tap repeatedly. The button's visual state must reflect whether it is active.

**Design standard:** a player picking up the game for the first time should understand the controls within 5 seconds without reading anything. If the controls need a tutorial, they are too complex.

The visual design of the controls should fit your game's theme. Sleek and minimal, rugged and mechanical, glowing and sci-fi — your choice. What is not optional: visibility, fixed position, and clear active states.

Always implement keyboard fallback (WASD/arrows + space) for desktop testing.

---

## Auto-fire

Auto-fire toward the nearest enemy should activate while the fire button is held, or whenever an enemy is within a reasonable range. The player's job is positioning — aiming is handled automatically.

This is correct for mobile roguelites because:
- It lets the player focus entirely on dodging
- It removes the fine-motor precision penalty of small touchscreen targets
- It keeps the experience fast-paced and readable

Do not require the player to tap repeatedly to fire. Continuous fire on hold is the correct pattern.

---

## Player Feel

The player character must feel responsive and precise. Any input lag or floatiness will be noticed immediately.

**Invincibility frames after taking damage are mandatory.** Without i-frames, fast enemy groups deal damage faster than the player can react — this feels unfair and causes frustration rather than learning. 700–900ms of invincibility is standard.

During i-frames, the player sprite must visually communicate invincibility — flickering alpha is the conventional signal. The player must know they are protected.

**Knockback on damage** communicates the direction of the threat and creates separation from the source. Apply it to the player on taking damage, and to enemies on being hit.

---

## Enemy Design

Three archetypal enemy behaviours cover most roguelite needs:

**Charger:** moves directly toward the player. Simple, readable, dangerous in groups. The threat is proximity — the player must maintain distance.

**Shooter:** maintains distance and fires projectiles. Punishes the player for ignoring threats at range. The threat is projectiles — the player must prioritise these or take chip damage.

**Dasher:** idles briefly, then dashes to the player's last position. Punishes predictable movement. **The windup must be telegraphed visually** — a tint change, scale pulse, or glow before the dash. An untelegraphed instant dash is unfair. The telegraph window (300–500ms) is what makes the mechanic fair and learnable.

Each enemy type must be visually distinguishable from the others at a glance, and from the player. Use different shapes — not just different colours.

**Enemy health bars** communicate remaining threat. They should appear only when an enemy has taken damage (or always — your design choice), and should use colour to signal health state (full → half → critical).

---

## Combat Feedback

Every hit — by the player and against the player — must have three simultaneous responses:
1. Visual (hit flash on the target)
2. Physics (knockback)
3. Audio (a sound that matches the impact weight)

Missing any of these makes hits feel incomplete. All three together create the sensation of a satisfying, weighty combat system.

Enemy deaths must be more dramatic than hits. A death burst, a brief hitstop, and a louder sound together signal that something significant happened.

**Common failure mode:** enemies that disappear instantly on death with no feedback. The player gets no confirmation, no satisfaction, and no information about what they killed.

---

## Projectile Clarity

Bullets and projectiles must be visually readable at game speed. At minimum:
- Player projectiles and enemy projectiles must be visually distinct (different colours, different shapes, or both)
- Projectiles must be large enough to see clearly on a phone screen
- Enemy projectiles must be slightly brighter or more saturated than the environment — they must pop visually

Projectiles that blend into the background are a difficulty spike that feels unfair rather than skillful.

---

## Collision Fairness

Use physics hitboxes that are slightly smaller than the visual sprite. This is called "favour the player" hitbox design — it makes near-misses feel exciting rather than frustrating, and reduces complaints about "getting hit by nothing."

This applies to both player hitboxes (make them smaller than the sprite) and enemy projectile hitboxes (slightly smaller than the visible bullet).
