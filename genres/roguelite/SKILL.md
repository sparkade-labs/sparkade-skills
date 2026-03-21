# Roguelite — Genre Skill

You are building a roguelite for the Sparkade platform. Read this file in full, then fetch each sub-skill listed below.

---

## What a great Sparkade roguelite feels like

A complete, satisfying run in 3–6 minutes. The player dies, immediately understands why, and wants to try again with a different approach. Every run feels distinct. Difficulty escalates smoothly — never a wall, never a plateau.

**The four non-negotiables:**
1. **Momentum** — there is always something happening or about to happen. Dead air kills engagement.
2. **Escalation** — the player should feel pressure increasing with every room. Depth must matter.
3. **Agency** — upgrade choices must change how the run plays, not just make numbers bigger.
4. **Clarity** — health, score, and depth are always visible. The player always knows how they're doing.

---

## Sub-skills — fetch all of these

| Sub-skill | URL |
|-----------|-----|
| Run Structure | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/run-structure.md |
| Combat | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/combat.md |
| Upgrades | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/upgrades.md |
| Procedural Generation | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/procedural.md |

Also fetch if not already fetched:
- https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/polish.md

---

## Score design

Score should reflect both depth reached and performance within each room. Reward the player for playing well, not just for surviving. A multiplier system, combo system, or room-clear bonus each work — choose what fits your theme.

Score must be a non-negative integer. Submit with `window.gameOver(score)`.

---

## Visual identity principles

**Readability above all.** In a fast-paced combat game, the player needs to read the scene instantly. Every visual decision should serve readability first, style second.

- The player character must be immediately distinguishable from all enemies at a glance. This means shape differentiation, not just colour.
- Each enemy type must have a visually distinct silhouette. A player should be able to identify enemy types in their peripheral vision.
- Damage feedback (hit flashes, particles) must use colours that contrast with both the enemy colour and the environment.
- The game background must never compete with gameplay elements for visual attention. Keep it dark, subtle, and consistent.
- Health and danger must communicate instantly — the player should feel their HP dropping before they consciously read the HUD number.

**Colour palette:** you have creative freedom here. Choose a palette that fits your theme and commit to it. The player character should use the Sparkade accent colour `#F55018` unless your theme has a strong reason otherwise.

---

## Run arc

The run is infinite — depth scales until the player dies. There is no win state. Structure the run in floors of 3 rooms each, with an upgrade room every 3rd room and a boss every 9th room.

Score is the measure of success.

---

## What NOT to build

These patterns kill the Sparkade roguelite experience:

- **Inventory management** — too complex for a 3-minute mobile session
- **Crafting or item combining** — same reason
- **Permanent meta-progression** — Sparkade games have no persistent storage between sessions
- **Unskippable intros or cutscenes** — the audience has zero patience for them
- **More than 3 active upgrade slots** — cognitive overload destroys replayability
- **Regenerating health** — removes tension entirely. Health pickups only.
- **Enemies that are off-screen threats** — on mobile, if you can't see it, you can't react to it. Keep combat within the visible space.
