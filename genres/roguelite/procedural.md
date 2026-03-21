# Procedural Generation — Roguelite Sub-Skill

How to generate enemy compositions, loot, and bosses that feel varied and fair at every depth.

---

## Enemy Composition

Never hard-code which enemies spawn in a room. Generate composition from a weighted table that shifts with depth.

Early floors should introduce enemy types gradually — chargers first, shooters appear from floor 2, dashers from floor 3 or 4. As depth increases, the composition should shift toward the harder archetypes. By floor 5+, all three types should be in the mix and the player should be facing meaningful variety in every room.

The exact weights and thresholds are your design decision. The principle is: easy → mixed → hard over the course of the run, with no hard cutoffs.

---

## Spawn Positioning

Never spawn enemies on top of the player or on top of each other. Enforce a minimum distance from the player spawn point (120–180px) and a minimum distance between enemies.

Enemies that spawn inside the player create instant unavoidable damage, which feels unfair regardless of the player's skill. This is a solvable problem — always solve it.

**Staged spawning** (staggering spawns by 150–250ms each) gives the player time to react and makes rooms feel alive rather than instantly overwhelming. Use a brief visual effect at each spawn point — a flash, a materialisation effect, any visual signal that an enemy is about to appear. This converts "surprise damage" into "telegraphed threat."

---

## Boss Design

A boss is the culmination of a floor — it should feel like a significant threat. Design bosses with:

**Multiple phases.** A boss that behaves the same at 10% HP as at 100% HP is not a boss, it is a large enemy. Phase transitions should happen at meaningful HP thresholds (50%, 25%) and each phase should be visually and behaviourally distinct. Mark phase transitions with a dramatic visual effect — flash, shake, colour change, scale pulse.

**A visual phase indicator.** The player must always know what phase the boss is in. A prominent HP bar at the top of the screen (separate from the regular HUD) is conventional and clear.

**Readable attack patterns.** Boss attacks must be telegraphed. An attack that fires with no warning is not difficult — it is unfair. The tell should be long enough (300–500ms) that a skilled player can react, short enough that it still feels fast.

**Appropriate scale.** A boss that looks the same size as a regular enemy does not feel like a boss. Scale it up — 2–3× the size of a standard enemy is appropriate.

**An introduction moment.** When the boss spawns, the room should acknowledge it — a brief pause, camera shake, a roar sound, an introduction text. This signals to the player that something significant is happening.

---

## Health Drop Logic

Health pickups should not be random noise — they should feel like the game giving the player a chance.

Drop probability should increase when the player is at low HP. A player at 20% HP should see health more often than a player at 80% HP. This is not hand-holding — it is good game design that keeps runs competitive rather than ending them through statistical bad luck.

Health orbs should visually communicate their value. A small heal and a large heal should look different. An orb that bobs gently draws the eye and communicates "pick me up."

---

## Variety Beyond Enemies

Procedural variety does not have to mean only enemy composition. Consider:
- Room visual atmosphere that shifts with floor
- Environmental hazards that appear from mid-game onwards
- Elite enemy variants (higher HP, faster, tinted differently) that appear at higher depths

The goal is that no two runs feel identical, even if the player chooses the same upgrades. Variety keeps the game alive for replayability.
