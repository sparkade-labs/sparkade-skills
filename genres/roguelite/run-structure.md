# Run Structure — Roguelite Sub-Skill

How the run loop, room progression, and scene state should be architected. The principles here apply regardless of theme or visual style.

---

## Run State

Maintain a single run state object that travels with the player through all rooms. It is initialised at run start and only reset on death. It must track at minimum: current depth, floor, kills, score, current HP, max HP, active upgrades, and whether the player took damage this room.

Pass this state when transitioning between rooms. Never use global variables.

---

## Room State Machine

Every room passes through four states in sequence: `ENTERING → ACTIVE → CLEARED → TRANSITIONING`.

During ENTERING: input is disabled, the room is set up, a brief introductory moment plays (visual or audio). This pause gives the player a beat to orient themselves.

During ACTIVE: input is enabled, enemies behave, the player can be damaged.

During CLEARED: input is disabled briefly, a clear reward plays, and the next room type is determined. Drop health pickups here if appropriate.

During TRANSITIONING: the room transition effect plays, then the next room loads at its midpoint.

**Why the ENTERING pause matters:** dropping the player into an active enemy room with no transition is disorienting and feels unfair. Even 500ms of setup time dramatically improves perceived fairness.

---

## Room Types

Three room types, triggered by depth:
- **Combat rooms** — the default. 2–3 combat rooms per floor.
- **Upgrade rooms** — every 3rd room. No enemies. The player chooses one upgrade.
- **Boss rooms** — every 9th room. Overrides the upgrade room. Single high-HP enemy with multiple phases.

---

## Room Visual Design

An empty rectangle on a black background is not a room — it is a void. Every room must feel like a space.

Design rooms to express your game's theme visually. This might be a floor texture, a grid pattern, environmental props, lighting effects, or architectural framing. The specific design is your creative choice.

**The standard:** a player should be able to screenshot any room and, without any UI, know roughly what kind of game they are playing.

Corner details, subtle floor patterns, and wall indicators all contribute to this without cluttering the gameplay space. The boundary of the room must be visible — the player should never be surprised by an invisible wall.

Vary the room atmosphere with depth. Floor 1 should feel different from Floor 5. This can be colour temperature, pattern density, lighting, or any other visual variable — the method is yours to choose.

---

## HUD Design

Three pieces of information must always be visible: health, current score, and current depth.

Health lives top-left. Score lives top-right. Depth lives top-centre. This is conventional — don't reinvent it.

The HUD must not compete with gameplay visually. A semi-transparent background behind HUD elements solves this. All HUD text must use stroke for readability against dynamic backgrounds.

A "remaining enemies" indicator during combat builds tension as the count approaches zero. This is optional but strongly recommended.

The HUD style should match your game's visual theme — the layout above is fixed, but the aesthetic expression is yours.

---

## Depth Scaling

Difficulty must increase continuously and perceptibly. The player should feel the game getting harder.

Scale enemy HP, speed, damage output, and count with depth. The exact scalars are your design choice — calibrate them so floor 1 is approachable and floor 5+ is genuinely threatening. Cap speed increases so enemies don't become physically impossible to dodge on mobile.

Announce floor transitions visually — the player should know when they've entered a new floor. This is a milestone worth marking.

---

## Player Death

Death must have ceremony. At minimum: camera shake, particle burst at the player's position, slow-motion, then a fade to the GameOver scene. The slow-motion lets the player see what killed them. This reduces frustration and increases the desire to retry.

Call `window.gameOver(finalScore)` immediately before transitioning to GameOverScene.

---

## Perfect Room Bonus

Reward players who clear a room without taking damage. A score bonus, visual flourish, and brief audio sting each reinforce skillful play. Mark the run state so you know whether the player took damage in the current room — reset this flag at the start of every new room.
