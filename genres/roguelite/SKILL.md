# Roguelite — Genre Skill

You are building a roguelite for the Sparkade platform. Read this file in full, then fetch each sub-skill listed below before writing any code.

---

## What makes a great Sparkade roguelite

A great Sparkade roguelite delivers a complete, satisfying run in **3–6 minutes**. The run must feel fair, escalate in tension, and end with the player immediately wanting to try again. On mobile, everything must be controllable one-handed.

**The four pillars:**
1. **Momentum** — the player always has something to do and something to look forward to. No dead air.
2. **Escalation** — depth should visibly increase difficulty and reward. Players should feel the pressure build.
3. **Agency** — upgrade choices must feel meaningful. Bad builds exist but bad luck alone should never be the cause of death.
4. **Clarity** — health, score, and depth must always be visible. Never obscure critical information.

---

## Sub-skills — fetch all of these

| Sub-skill | URL | When to fetch |
|-----------|-----|---------------|
| Run Structure | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/run-structure.md | Always |
| Combat | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/combat.md | Always |
| Upgrades | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/upgrades.md | Always |
| Procedural Generation | https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/genres/roguelite/procedural.md | Always |

Also fetch the polish skill if not already fetched:
- https://raw.githubusercontent.com/sparkade-labs/sparkade-skills/main/core/polish.md

---

## Score design

Score in a roguelite should reflect **depth + performance**, not just time alive.

Recommended formula:
```
score = (depth * 1000) + (kills * 50) + (bonus_multiplier * depth * 200)
```

- `depth` — floor/room number reached
- `kills` — total enemies defeated
- `bonus_multiplier` — awarded for taking no damage in a room, clearing quickly, etc.

Submit with `window.gameOver(score)` at run end. Score must be an integer.

---

## Visual identity

A roguelite on Sparkade should have a consistent, readable aesthetic:

- **Dark background** — always `#080808` or very close. Light backgrounds look wrong in the feed.
- **Enemy colour coding** — each enemy type must have a distinct colour. Players learn enemy types visually within one run.
- **Player colour** — always the Sparkade orange `#F55018`. This is the player, always.
- **Health** — red `#ef4444` for damage, green `#22c55e` for pickups. Universal.
- **Depth indicator** — always visible top-centre. Format: `DEPTH 3` in small monospace caps.

---

## Run arc

```
MenuScene
  └─ GameScene
       ├─ Floor 1 (Rooms 1–3) → normal enemies, one upgrade choice
       ├─ Floor 2 (Rooms 4–6) → harder enemies, two upgrade choices
       ├─ Floor 3 (Rooms 7–9) → elite enemies + mini-boss, two upgrade choices
       └─ Floor N... (infinite scaling)
          └─ Every 3rd room = upgrade choice
          └─ Every 9th room = boss encounter
```

The run is infinite — depth scales until the player dies. There is no win condition. Score is the measure of success.

---

## What NOT to do

- Do not add inventory management — too complex for a 3-minute mobile run.
- Do not add crafting or combining — same reason.
- Do not add permanent meta-progression (unlocks between runs) — Sparkade has no persistent storage per game.
- Do not add cutscenes or unskippable animations — the platform audience has zero patience for them.
- Do not add more than 3 active upgrade slots — cognitive load kills replayability.
- Do not add regenerating health — it removes tension. Health pickups only.
