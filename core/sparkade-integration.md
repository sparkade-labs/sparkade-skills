# Sparkade Platform Integration — Core Skill

The platform bridge is fixed infrastructure. Every game uses the same protocol — no creative decisions here, just correctness.

---

## The Message Protocol

The shell injects these functions onto `window` before your game loads.

### Required — every game must call these

```js
window.gameStarted()   // call once when gameplay begins
window.gameExit()      // call only from an explicit exit button
```

These two are mandatory. A game file missing either will be rejected on submission.

### Optional — only for games with scoring

```js
window.gameOver(score) // call once when a run ends — score is a positive integer
window.gameOver()      // call with no argument if the session ends but there is no score
```

`gameOver` is not required. Clicker games, cozy games, and casual games where the player simply exits do not need it. If your game has a leaderboard, call `gameOver(score)` at the end of every run.

**Timing rules that must not be broken:**
- `gameStarted()` fires when the player is actually playing, not when the menu appears.
- `gameOver()` fires exactly once, immediately at run end. Never on menu, never on exit.
- `gameExit()` fires only from a deliberate exit action. Not on death, not on game over.

---

## Scoring — Optional

Scoring is opt-in. Not every game needs a leaderboard.

### Games with scoring

Pass the final score as an integer to `gameOver()`. The platform submits it, updates the leaderboard, and fires `onScoreResult` back to your game.

```js
window.gameOver(score);

window.onScoreResult = function(sparksEarned, isPersonalBest, rank, prevBest) {
  // Show sparks earned, personal best badge, global rank
};
```

The leaderboard trophy button appears automatically in the feed on the first score submission — no configuration needed.

### Games without scoring

Call `gameOver()` with no argument. The platform handles the session cleanly with no score submission and no leaderboard. The trophy button never appears for this game.

```js
window.gameOver(); // no score — session ends cleanly
```

`onScoreResult` does not fire for scoreless games. Do not register it if your game has no scoring.

---

## Save System

Games can persist JSON data between sessions. The platform stores one save blob per user per game.

### Reading save data

Call `window.loadData()` early in your scene's `create()`. The result arrives asynchronously via the callback — **do not assume data is available synchronously.**

```js
window.onDataLoaded = function(save) {
  // save is a plain object — read whatever your game stored
  var level = save.level || 1;
  var gold  = save.gold  || 0;

  // _iap is injected by the platform — read-only, never write it yourself
  var hasSword = save._iap && save._iap.sword;
};

window.loadData();
```

### Writing save data

Call `window.saveData(data)` whenever meaningful progress occurs. The platform debounces writes automatically — call it freely without worrying about performance.

```js
window.saveData({
  level:    5,
  gold:     250,
  unlocked: ['forest', 'cave'],
});
```

**Rules:**
- `saveData` is fire-and-forget — no callback, no confirmation needed.
- Never include `_iap` in your save payload — the platform strips it. IAP ownership is managed exclusively by the platform.
- `saveData` and `gameOver` / `gameExit` are safe to call together — the shell flushes any pending save on exit automatically.
- Max save size is 64KB. This is generous — a detailed RPG save is typically under 5KB.
- Guest users (not logged in) receive an empty save `{}`. Handle this gracefully.

---

## IAP — In-App Purchases with Sparks

Games can sell items to players using Sparks (Sparkade's virtual currency). Items are registered in the Sparkade admin panel before the game goes live. IAP is optional — only implement it if your game has things to sell.

### Two item types

- **Permanent** — bought once, owned forever. Example: a playable character, a skin, a game mode unlock.
- **Consumable** — can be purchased multiple times, has a quantity. Example: a resource pack, a power-up bundle.

### Loading the item catalogue

Call `window.loadItems()` to fetch what the game sells, including whether the current player already owns each item. Prices are always set in the admin panel — never hardcode them in your game.

```js
window.onItemsLoaded = function(items) {
  // items is a key-indexed object matching your registered item keys
  // {
  //   dave:          { name: 'Dave', cost: 100, item_type: 'permanent',  owned: false },
  //   shield_potion: { name: 'Shield Potion', cost: 30, item_type: 'consumable', owned: 3 },
  // }

  // Use this to populate your in-game shop UI with real names and prices
  showShopUI(items);
};

window.loadItems();
```

### Requesting a purchase

Pass only the item key. The platform shows its own native purchase overlay — your game does not draw it.

```js
window.requestPurchase('dave');

window.onPurchaseResult = function(success, updatedSave) {
  if (success) {
    // updatedSave._iap.dave is now true
    unlockCharacter('dave');
  }
};
```

**Rules:**
- Never pass a price to `requestPurchase` — cost is always read from the database.
- The purchase overlay is owned by Sparkade. The player confirms or cancels there. Your game just waits.
- If the player doesn't own the item yet, `owned` will be `false` in `onItemsLoaded`. Check this before showing a buy button.
- Permanent items block duplicate purchase server-side — you don't need to guard against it yourself.
- Consumables can always be repurchased regardless of quantity already owned.

### Spending consumables

When a player uses a consumable item, call `window.spendItem(key, amount)`. The platform deducts the quantity server-side and returns the updated `_iap`.

```js
// Player uses one shield potion
window.spendItem('shield_potion', 1);

window.onSpendResult = function(success, updatedSave) {
  if (success) {
    activateShield();
    // updatedSave._iap.shield_potion is now 2 (was 3)
    updatePotionDisplay(updatedSave._iap.shield_potion || 0);
  } else {
    // Player had 0 — server rejected it
    showMessage('No potions left!');
  }
};
```

**Rules:**
- Only call `spendItem` on consumables. Calling it on a permanent item will fail.
- Always use `|| 0` when reading consumable quantities — when the last one is spent the key is removed from `_iap` entirely rather than sitting at zero.
- Amount must be a positive integer. The server rejects negative values.

### Reading IAP ownership from save data

`_iap` is always present on the save object returned by `loadData`, `onPurchaseResult`, and `onSpendResult`. It is computed fresh from the database on every read — it cannot be faked by writing to `saveData`.

```js
window.onDataLoaded = function(save) {
  // Permanent item — boolean
  if (save._iap && save._iap.dave) {
    unlockCharacter('dave');
  }

  // Consumable — quantity (or 0 if none owned)
  var potions = (save._iap && save._iap.shield_potion) || 0;
  updatePotionDisplay(potions);
};
```

---

## Recommended Initialisation Pattern

```js
// In your opening scene's create():
window.onDataLoaded = function(save) {
  // Apply save state to your game before showing anything
  applyGameState(save);
  // _iap is available here too — read-only, never write it
  applyIapUnlocks(save._iap || {});
  // Now it is safe to start
  window.gameStarted();
};

window.onItemsLoaded = function(items) {
  // Store for shop UI use later — only needed if your game has IAP
  _itemCatalogue = items;
};

window.loadData();   // fires onDataLoaded when ready
window.loadItems();  // fires onItemsLoaded when ready — omit if no IAP
```

Both calls return instantly from a preloaded cache — the platform fetches save data and items in parallel before your game script even loads, so there is no network wait in practice.

### Exit button

Every game must have a visible exit button. When tapped, flush any pending save first then call `gameExit()`:

```js
exitButton.on('pointerdown', function() {
  window.saveData(currentState); // flush save
  window.gameExit();             // hand control back to the platform
});
```

---

## Lifecycle

### With scoring

```
BootScene → MenuScene → [player starts] → GameScene → [run ends] → GameOverScene
                                                                         │
                                                           "Play Again" → GameScene
                                                           "Exit" → window.gameExit()
```

### Without scoring

```
BootScene → MenuScene → [player starts] → GameScene → [run ends] → EndScene
                                                                         │
                                                           "Play Again" → GameScene
                                                           "Exit" → window.gameExit()
```

The structure is identical — the only difference is whether `gameOver()` receives a score and whether you register `onScoreResult`.

---

## GameOverScene Requirements

### With scoring

The GameOver screen is what the player sees after every single run. It must be polished — not an afterthought.

**What it must show:**
- The final score, large and prominent, animated counting up from zero
- Sparks earned (when `onScoreResult` fires with `sparksEarned > 0`)
- A "NEW BEST" indicator (when `isPersonalBest` is true) — make this feel exciting
- Relevant run stats (whatever makes sense for the genre — depth, kills, time, etc.)
- A clearly primary "PLAY AGAIN" action
- A clearly secondary "EXIT" action

**Quality standard:** The score counting up from zero feels satisfying. The personal best badge should animate in — it's a moment worth celebrating. `onScoreResult` may arrive slightly after the scene loads, so design for late arrival.

### Without scoring

The end screen should still feel rewarding. Show whatever is meaningful for your genre — time survived, items collected, rooms cleared, story progress reached. A "PLAY AGAIN" and "EXIT" action are still required.

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
