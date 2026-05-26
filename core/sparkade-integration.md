# Sparkade Platform Integration — Core Skill

The platform bridge is fixed infrastructure. Every game uses the same protocol — no creative decisions here, just correctness.

---

## The Message Protocol

The Sparkade shell (`game-shell.html`) injects these functions onto `window` before your game script runs. Your game communicates with the Sparkade platform exclusively through these functions — there is no direct network access, no DOM manipulation, and no postMessage calls from game code. The shell handles all communication with the parent page.

### Player identity

```js
var name = window.getUsername();   // returns username string, or null for guests
```

Returns the logged-in player's username, or `null` if the player is not logged in. Do not gate gameplay on being logged in — guests must be able to play. Treat `null` the same as any anonymous player. Useful if your game displays the player's name.

---

## The API

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

> **Persistence is two halves — you must do BOTH or progress is silently lost.**
> Saving without loading is the single most common mistake. If your game has any
> progression (score, level, currency, unlocks, upgrades, settings), you must:
> 1. **Load on startup** — call `window.loadData()` and register `window.onDataLoaded`
>    to restore the saved state *before* the player starts playing.
> 2. **Save on change** — call `window.saveData(state)` when meaningful progress
>    occurs, and again before `gameOver()` / `gameExit()`.
>
> A game that calls `saveData()` but never `loadData()` will appear to work in a
> single session, then reset to zero every time it reopens — which for a clicker,
> idle, RPG, or any progression game means the whole point of the game is broken.
> If your game has nothing worth persisting, skip the save system entirely. There
> is no middle ground: either wire up both halves, or use neither.

### Reading save data

Call `window.loadData()` early in your scene's `create()`. The result arrives asynchronously via the callback — **do not assume data is available synchronously.** Apply the returned state to your game before the player can act on it.

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
- `saveData` and `gameOver` / `gameExit` are safe to call together — the shell flushes any pending debounced save automatically before sending either signal.
- Max save size is 64KB. This is generous — a detailed RPG save is typically under 5KB.
- Guest users (not logged in) receive an empty save `{}`. Handle this gracefully.

### Deleting save data

```js
window.deleteSave();
```

Clears the player's save immediately. Use only for explicit "reset all progress" features. The call is fire-and-forget — no callback.

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

Keep your persisted state in **one object with one consistent shape**. Save that
object, load it back, and apply it — the shape you write must be the shape you read.
Mismatched shapes (storing a value one way and reading it another) are a common
source of progression bugs.

```js
// A single state object — this exact shape is what you save AND load.
var state = { score: 0, level: 1, coins: 0, upgrades: {} };

// In your opening scene's create():
window.onDataLoaded = function(save) {
  // Merge saved values over the defaults. Reading the same keys you wrote.
  state.score    = save.score    || 0;
  state.level    = save.level     || 1;
  state.coins    = save.coins     || 0;
  state.upgrades = save.upgrades  || {};

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

// Later, whenever progress changes, save the SAME object you loaded into:
function persist() {
  window.saveData(state);   // same shape, every time
}
```

Both calls return instantly from a preloaded cache — the platform fetches save data and items in parallel before your game script even loads, so there is no network wait in practice.

**Derived values vs stored values:** store the raw facts, derive everything else at
load time. For example, store `upgrades.click_power = 3` (the level owned) and compute
the actual multiplier from it on load — don't store the computed multiplier, or the
stored level and the live value will drift apart. One source of truth, recomputed on load.

### Exit button

Every game must have a visible exit button. When tapped, flush any pending save first then call `gameExit()`:

```js
exitButton.on('pointerdown', function() {
  window.saveData(currentState); // flush save
  window.gameExit();             // hand control back to the platform
});
```

---

## Platform-Forced Exit

The platform sends a `GAME_EXIT` signal if the player swipes away from the game while it's running. Register `window.onGameExit` to handle any cleanup:

```js
window.onGameExit = function() {
  // The shell already flushes any pending saveData call automatically.
  // Only register this if you need extra cleanup beyond what saveData covers.
};
```

This callback is optional. The shell auto-flushes pending saves on `GAME_EXIT` — you only need `onGameExit` if your game has cleanup work that `saveData` alone doesn't handle.

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

## GameOver and End Screens

### With scoring

The GameOverScene must:
- Display the final score
- Show sparks earned when `onScoreResult` fires (`sparksEarned > 0`)
- Show a personal best indicator when `isPersonalBest` is `true`
- Provide a "PLAY AGAIN" action and an "EXIT" action

`onScoreResult` may fire slightly after the scene loads — design the screen to handle late arrival of that data gracefully.

### Without scoring

The end screen must provide a "PLAY AGAIN" action and an "EXIT" action that calls `window.gameExit()`.

---

## MenuScene

MenuScene must:
- Provide a start action that calls `window.gameStarted()` then transitions to GameScene.

---

## Input

Touch is the primary input. On-screen controls must be rendered visible elements — not invisible tap zones.

Always implement keyboard input (WASD/arrows + spacebar) as a secondary path for desktop testing.