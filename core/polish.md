# Polish & Juice — Sparkade Core Skill

Polish is not decoration — it is communication. Every juice technique exists to give the player information about what just happened, and to make that information feel satisfying to receive. Understand why each technique works, then implement it in a way that fits your game's tone.

---

## Screen Shake

Shake communicates force. A hit that moves the screen feels heavier than one that doesn't.

Calibrate intensity to the event. A graze is different from a death blow. Over-shaking destroys the effect — if everything shakes equally, nothing feels impactful. Use `this.cameras.main.shake(duration, intensity)`.

**Common failure mode:** using the same shake value for every event, or not shaking at all.

---

## Hit Flash

A brief white flash on a hit gives instant visual confirmation of damage. Without it, hits feel like they passed through.

The flash should be very short (50–80ms). For enemies, follow the white flash with a brief tint in the enemy's colour family — this tells the player the hit registered without removing the enemy's visual identity. For the player, white only.

**Common failure mode:** tinting without flashing first, which looks like a colour change rather than an impact.

---

## Hitstop (Freeze Frames)

Pausing the physics simulation for 60–150ms on a significant impact is the single most impactful juice technique in action games. It gives the player's brain time to register the hit, and makes every kill feel deliberate rather than accidental.

Scale duration to significance. A regular kill gets ~60ms. A boss death gets ~150ms. Never stack hitstops — guard against it with a flag.

**Why it works:** human visual processing needs ~80ms to consciously register a frame. A hitstop that matches this window is perceived as a "perfect hit" frame.

---

## Knockback

Everything that gets hit should move away from the source of the hit. Stationary damage is unconvincing.

Apply knockback to enemies on player bullet impact. Apply lighter knockback to the player when taking damage — this also communicates the direction the threat came from.

---

## Particles

Death bursts confirm kills with visceral, colour-matched feedback. Match the particle colour to the enemy type — this reinforces visual identity even in the moment of destruction.

Use two burst layers: a main spread of colour-matched particles, and a smaller inner burst of near-white particles for the bright core of the explosion. This two-layer approach makes bursts feel energetic rather than flat.

Requires a 1×1 white `'pixel'` texture generated in BootScene.

**Common failure mode:** single-colour, single-layer bursts that look like confetti rather than an explosion.

---

## Floating Combat Text

Numbers that float upward and fade communicate exactly what happened and by how much. They must be readable against any background — always use `stroke` + `strokeThickness`.

Use colour to communicate type: white for score/kills, red for damage, green for healing, gold for bonuses. Keep font size proportional to significance — a boss kill deserves bigger text than a regular hit.

**Common failure mode:** floating text without stroke, rendering it unreadable against light-coloured game elements.

---

## Score Animation

A score that jumps instantly to its new value is a missed opportunity. Animate it counting up using `tweens.addCounter()`. The duration should be short (250–400ms) — long enough to be satisfying, short enough not to feel sluggish.

---

## Death Sequence

The moment the player dies deserves ceremony. At minimum: a camera shake, a particle burst at the player's position, a brief slow-motion effect (reduce `physics.world.timeScale` and `time.timeScale`), and a fade to the GameOver scene.

The slow-motion lets the player see what killed them and process the loss before being yanked to the score screen. This reduces frustration significantly.

---

## Transitions

Scene transitions should never be instantaneous cuts. Use `cameras.main.fadeIn()` on scene start and `cameras.main.fadeOut()` before `scene.start()`. Between game rooms or levels, a flash effect works well — it masks the instantaneous content swap and makes it feel like a deliberate transition.

---

## Audio

No audio files are available. Use the Web Audio API to generate all sound effects procedurally with oscillators. Design sounds to match game events — a shoot sound should feel snappy, a death sound should feel heavy, a level-up sound should feel rewarding.

Cache a single `AudioContext` instance — never create a new one per sound. The AudioContext must be created after a user interaction (Phaser's first input event satisfies this).

---

## HUD Legibility

The HUD must be readable at a glance during fast-paced gameplay.

A semi-transparent background strip behind HUD elements dramatically improves readability against dynamic game content. Score should always be visible during play. Critical information (health, remaining enemies) must never be obscured by gameplay.

All HUD text must use `stroke` + `strokeThickness` as a baseline — the game world behind it can be any colour.

---

## Depth Layer Discipline

Set explicit depths on everything. Z-order bugs — a UI element hidden behind a game object, or a particle appearing behind a wall — immediately break the feeling of polish.

Establish a clear layering system at the start of every game and assign every object to a layer. The specific values are your choice; consistency is what matters.
