# Sparkade Platform Integration — Core Skill

The platform bridge is fixed infrastructure. Every game uses the same protocol — no creative decisions here, just correctness.

---

## The Message Protocol

The shell injects these functions onto `window` before your game loads:

```js
window.gameStarted()        // call once when gameplay begins
window.gameOver(score)      // call once when the run ends — score is a positive integer
window.gameExit()           // call only from an explicit exit button
```

The platform sends back a score result after `gameOver()`. Register this handler in `GameOverScene.create()`:

```js
window.onScoreResult = function(sparksEarned, isPersonalBest, rank, prevBest) { ... }
```

**Timing rules that must not be broken:**
- `gameStarted()` fires when the player is actually playing, not when the menu appears.
- `gameOver()` fires exactly once, immediately at run end. Never on menu, never on exit.
- `gameExit()` fires only from a deliberate exit action. Not on death, not on game over.

---

## Lifecycle

```
BootScene → MenuScene → [player starts] → GameScene → [run ends] → GameOverScene
                                                                         │
                                                           "Play Again" → GameScene
                                                           "Exit" → window.gameExit()
```

---

## GameOverScene Requirements

The GameOver screen is what the player sees after every single run. It must be polished — not an afterthought.

**What it must show:**
- The final score, large and prominent, animated counting up from zero
- Sparks earned (when `onScoreResult` fires with `sparksEarned > 0`)
- A "NEW BEST" indicator (when `isPersonalBest` is true) — make this feel exciting
- Relevant run stats (whatever makes sense for the genre — depth, kills, time, etc.)
- A clearly primary "PLAY AGAIN" action
- A clearly secondary "EXIT" action

**Quality standard:** The score counting up from zero feels satisfying. The personal best badge should animate in — it's a moment worth celebrating. `onScoreResult` may arrive slightly after the scene loads, so design for late arrival.

---

## MenuScene Requirements

The menu is the player's first impression of the game. It must communicate the game's visual identity immediately.

**What it must show:**
- The game title — large, styled to match the game's theme
- A brief one-line description or tagline
- A prominent start action
- Optionally: an idle animation, ambient effect, or preview of the game world

**What it must not be:** a black screen with white text and a button.

The start action must call `window.gameStarted()` then transition to GameScene.

---

## Mobile Controls Principle

Sparkade is mobile-first. Touch is the primary input.

If your genre requires directional or action input, implement visible on-screen control elements. Invisible tap zones are not acceptable — players need to see what they are pressing, and controls must have clear active and inactive visual states.

Auto-fire (automatically targeting the nearest valid target) should be used whenever manual aiming would interrupt movement flow. Reserve explicit fire input for genres where aiming is a core mechanic.

Always implement keyboard input as a secondary path for desktop testing. The game must be fully playable with WASD/arrow keys and spacebar.

**The specific control design is your creative decision** — it should fit the genre, feel natural on a phone, and be immediately understandable without explanation.
