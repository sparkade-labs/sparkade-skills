# Upgrades — Roguelite Sub-Skill

The upgrade selection is the primary source of player agency. It must feel exciting, meaningful, and worth the pause in action.

---

## What makes a good upgrade set

Design 10–15 upgrades at a minimum. They should span three categories:

**Stat upgrades (common):** increase a numeric value — damage, fire rate, speed, bullet range, max HP. These stack and feel good to accumulate. They should feel significant (30–50% improvement), not marginal (5–10%).

**Build-defining upgrades (rare):** change how the player plays — piercing bullets, multishot spread, lifesteal on kill, explosion on kill. These shift strategy and make runs feel distinct. A player who gets Piercing + Multishot plays completely differently than a player who gets lifesteal + speed.

**Power upgrades (epic):** dramatic effects that transform the run — a dash ability, a score multiplier, an on-kill effect that chains. These should feel like lottery wins.

Rarity affects both drop rate and how exciting the moment feels. An epic drop should feel like an event.

---

## The selection screen as a moment

The upgrade screen is not just a menu — it is the emotional highlight of every room clear. The player has just survived, and now gets to grow stronger. This moment deserves appropriate ceremony.

**What the screen must do:**
- Show exactly 3 choices (more is decision paralysis, fewer is too little agency)
- Communicate each upgrade's rarity visually and immediately
- Make the selected upgrade feel confirmed, not just removed
- Transition back into gameplay with energy

**What the screen must look like:**
- Choices must be clearly readable — name, description, rarity all legible at a glance
- The visual hierarchy must lead with the upgrade name, then description, then rarity
- Rarity should be communicated through colour and label — and ideally a visual treatment on the card (glow, border weight, background)
- Common, rare, and epic upgrades should feel meaningfully different to look at, not just differently coloured

**Animation:** cards that animate in feel more exciting than cards that appear instantly. Staggered entry (each card after the last) builds anticipation. A press response (scale, flash, or colour change) confirms selection. An exit animation before returning to gameplay closes the moment cleanly.

**Common failure mode:** a static list of text options with no visual differentiation and no animation. This looks like a debug menu, not a game feature.

---

## Balance principles

- A player who chooses upgrades randomly should still have a functional build
- The best upgrade in any pick should be obvious enough that players feel smart for choosing it, but not so obvious that choice feels meaningless
- Stacking the same stat upgrade multiple times should be powerful but not required — build variety is the goal
- Abilities (pierce, multishot, dash) should be non-stackable — getting the same ability twice should be prevented
- Stat upgrades can stack and should feel better each time

---

## Upgrade feedback

When an upgrade is selected, something in the game world should respond. The player sprite might briefly scale up, emit a burst of colour, or flash. This connects the menu choice to the game character and makes the upgrade feel real, not just numerical.

Play a distinct sound on upgrade selection — different from combat sounds, clearly "positive and exciting."
