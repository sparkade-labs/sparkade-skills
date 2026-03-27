> © Sparkade. Licensed under CC BY-NC-ND 4.0.  
> These skills are provided for AI-assisted game development on the Sparkade platform only.

# Runner — Genre Skill

You are building an **endless runner** for the Sparkade platform. Read this file in full, then fetch each sub-skill listed below.

An endless runner is a game where the player character moves forward automatically and continuously — the player never controls forward movement. The world scrolls past at increasing speed. The run ends when the player collides with an obstacle or fails a reaction test. There is no finish line. The goal is always to survive longer than last time.

If the game brief does not involve automatic forward movement and infinite obstacle scrolling, this is the wrong genre skill — return to INDEX.md.

---

## What a great Sparkade runner feels like

A session lasts 60–180 seconds. The player dies, instantly understands what killed them, and taps Play Again before consciously deciding to. The difficulty ramp feels fair — the first 20 seconds are approachable, the last 20 seconds before death feel genuinely threatening. Every death feels like the player's fault, not the game's.

**The four non-negotiables:**

1. **Instant readability** — the player understands the controls within 3 seconds of the first run, without reading anything. One input to jump, one input to slide/duck if applicable. No tutorial needed.
2. **Fairness** — every obstacle the player hits must have been visible and avoidable with the reaction time available at that scroll speed. An obstacle that appears with less than 400ms to react is a death the player will blame on the game. They are right.
3. **Escalation** — scroll speed must increase continuously and perceptibly. The player should feel the game getting harder. A session that feels the same speed at 30 seconds as at 120 seconds has failed.
4. **Rhythm** — great runners develop a flow state. Obstacles should have breathing room between them at low speed, compressing as speed increases. The player should feel in control until suddenly they aren't.

---

## Sub-skills — fetch all of these

| Sub-skill | URL |
|-----------|-----|
| Scrolling & World | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/scrolling.md |
| Player Feel | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/player.md |
| Obstacles & Difficulty | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/runner/obstacles.md |

---

## Score design

Distance is the primary score — every metre survived earns points. Layer on top of distance with performance bonuses: narrow misses, consecutive obstacles cleared, style points for specific actions. The score must always feel like it reflects how well the player played, not just how long they survived.

Score must be a non-negative integer. Submit with `window.gameOver(score)`.

A good target: a skilled player on their 10th run should be scoring roughly 3–5× what a first-time player scores. If the gap is smaller, skill expression is too low. If larger, the early difficulty is too punishing.

---

## Visual identity principles

**Contrast above all.** The player's eye is tracking the right side of the screen for incoming obstacles while managing the player character on the left. Every obstacle must pop against the background instantly. Every safe gap must read as safe instantly.

- The player character and obstacles must be visually distinct from the background at a glance — no camouflage, no similar colours between hazards and environment.
- Incoming obstacles should have a visual "tell" — a shadow on the ground, a silhouette at the screen edge — giving the player a fraction more reaction time and making deaths feel telegraphed.
- The background should have depth — a parallax of 2–3 layers at different scroll speeds makes the world feel real without cluttering the gameplay plane.
- Ground must be visually distinct from the sky. The player lives on the ground — it must read as solid, stable, and clearly defined.

**Theme:** fully your creative decision. A neon cityscape, a prehistoric jungle, a space station corridor, an underwater trench — the mechanic is identical, the world is yours to invent. Commit to the theme visually — every element should feel like it belongs to the same world.

---

## Session arc

```
MenuScene → [tap to start] → GameScene [endless scroll, increasing speed] → [collision] → GameOverScene
                                                                                               │
                                                                              "Play Again" → GameScene
                                                                              "Exit"      → window.gameExit()
```

The run is infinite — there is no win state. The game ends on first contact with any obstacle (or after a set number of lives if your design uses them). Call `window.gameOver(score)` immediately at run end before transitioning to GameOverScene.

---

## What NOT to build

These patterns kill the Sparkade runner experience:

- **Manual forward movement** — the player must never control speed or direction of travel. Auto-move only. If the player can stop, it's not a runner.
- **More than 2 inputs** — jump and duck covers everything a runner needs. Adding attack, dash, and special abilities creates cognitive overload on mobile.
- **Unpredictable obstacle spawning** — obstacles generated with pure randomness will occasionally produce impossible patterns. Always validate spawns. See `obstacles.md`.
- **Instant max speed** — starting at full difficulty removes the approachable entry that hooks new players. Always start slow.
- **Speed that never caps** — scroll speed must have a ceiling. At extreme speeds no human can react in time, which feels like the game cheating rather than a personal failure.
- **Invisible ground boundaries** — the player must always know exactly where the ground plane is. Ambiguous floor height makes jump timing feel broken.
- **Floating score display** — score must always be visible. Players play for score. If they can't see it, they have no feedback loop.
