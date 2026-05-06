# Fix: "Play Again" Button Not Working

## Problem
The "Play Again" button appears on the game-over overlay but clicking it doesn't start a new game.

## Root Cause: Double event binding for `restartGame()`

In `startGame()`, the `restartGame` function is bound to the button via **two mechanisms simultaneously**:

1. **Inline HTML**: `<button onclick="restartGame()">` — the primary handler
2. **Programmatic JS**: `btn.addEventListener('click', restartGame)` — added in `startGame()`

This creates two compounding bugs:

### Bug 1: Duplicate listener accumulation
Every time `startGame()` is called (including after the game starts initially), a **new** `click` listener is appended to the button — old ones are never removed. After one restart cycle, the button fires `restartGame()` twice; after another, three times; and so on.

### Bug 2: State reset mid-countdown
When `restartGame()` fires the second time (from the accumulated listener) within the same microtask tick as the first call:
- First call: hides game-over overlay, resets scores, calls `startCD(3)` → countdown begins
- Second call (milliseconds later): hides overlay again, **resets scores back to 0**, **resets paddleY positions**, **clears particles**, and calls `startCD(3)` again

The net effect is that the countdown restarts from the second call, but the **score display flickers** to 0-0 during the transition. In many cases the double-fire is fast enough that the user sees the scores flash to 0 then back, or the game state becomes inconsistent.

## Fix

**Remove the redundant programmatic listener in `startGame()`.** The inline `onclick` attribute already fires `restartGame()` correctly. The `addEventListener` adds nothing useful and causes the double-fire bug.

### Changes

**File: `index.html`**

Remove these lines from `startGame()`:
```javascript
  // Explicit click binding as fallback for restart button
  const btn = document.getElementById('btn-play-again');
  if (btn) { btn.addEventListener('click', restartGame); }
```

No other changes needed — the inline `onclick="restartGame()"` on the button is sufficient.

## Verification
1. Start a game → score points → game over overlay appears
2. Click "Play Again" → button should fire exactly once
3. New countdown starts, scores reset to 0-0 (once), new game begins normally
4. Repeat multiple restart cycles — button should still fire once each time
