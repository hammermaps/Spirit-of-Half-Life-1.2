# Spirit of Half-Life 1.2 – MoveWith System Analysis

## Overview

The **MoveWith system** (implemented in `SpiritSource/dlls/movewith.cpp` and
`SpiritSource/dlls/movewith.h`) is a custom feature of Spirit of Half-Life that
allows any entity to be parented to another entity at map-load time via the
`movewith` keyvalue. It maintains three singly-linked lists:

| Structure | Purpose |
|---|---|
| `CBaseEntity::m_pChildMoveWith` | Head of the list of entities parented to this entity |
| `CBaseEntity::m_pSiblingMoveWith` | Next sibling in the same parent's child list |
| `CBaseEntity::m_pAssistLink` | Global per-frame assist/desired processing list (rooted at `g_pWorld`) |

Two per-frame passes are run each game frame:

1. **`CheckAssistList`** (called from `StartFrame`) – adjusts velocities of PUSH
   entities that are about to stop so that child entities arrive at the correct
   position.
2. **`CheckDesiredList`** (called from `PostThink`) – applies deferred
   (desired) actions such as `DesiredAction`, `DesiredThink`, and
   `HandlePostAssist` after physics have been resolved.

---

## Bugs and Logic Errors Found

### 1. Null-Pointer Dereference – `ApplyDesiredSettings` (`LF_DESIRED_INFO` branch)

**File:** `SpiritSource/dlls/movewith.cpp`, function `ApplyDesiredSettings`

**Problem:**  
When the `LF_DESIRED_INFO` flag is set (via `UTIL_DesiredInfo`), the code
unconditionally accesses `pListMember->m_pChildMoveWith->pev->origin` and
related fields. If the entity has no children (`m_pChildMoveWith == NULL`),
this is an immediate null-pointer dereference and crash.

```cpp
// BEFORE (crash if m_pChildMoveWith == NULL):
ALERT(at_debug, "DesiredInfo: ... Child pos %f %f %f ...",
    ...,
    pListMember->m_pChildMoveWith->pev->origin.x, ...); // NULL dereference!
```

**Fix:** Guard the child-access block with an `if (pListMember->m_pChildMoveWith)` check.
When there is no child, a reduced info message is printed instead.

---

### 2. `g_doingDesired` Flag Not Reset on Early Return – `CheckDesiredList`

**File:** `SpiritSource/dlls/movewith.cpp`, function `CheckDesiredList`

**Problem:**  
`g_doingDesired` is set to `TRUE` at the top of the function. If `g_pWorld`
is `NULL` the function returns early *without* resetting `g_doingDesired` back
to `FALSE`. After this every subsequent call to `UTIL_MarkForDesired` will
shortcut directly into `ApplyDesiredSettings` instead of queuing the entity on
the assist list, corrupting the intended call order for all remaining entities
in that session.

```cpp
// BEFORE (g_doingDesired stays TRUE permanently):
g_doingDesired = TRUE;
if (!g_pWorld) {
    ALERT(...);
    return;   // ← g_doingDesired never reset!
}
```

**Fix:** Set `g_doingDesired = FALSE` immediately before the early `return`.

---

### 3. Wrong Error Message – `UTIL_SetMoveWithAvelocity`

**File:** `SpiritSource/dlls/movewith.cpp`, function `UTIL_SetMoveWithAvelocity`

**Problem:**  
The error message printed when the sibling loop-breaker fires inside
`UTIL_SetMoveWithAvelocity` says `"SetMoveWithVelocity: Infinite sibling list
for MoveWith!"` – naming the *velocity* function instead of the *avelocity*
function. This makes crash reports and server logs misleading when debugging.

**Fix:** Changed to `"SetMoveWithAvelocity: Infinite sibling list for MoveWith!"`.

---

### 4. Unbounded Recursion – `HandlePostAssist`

**File:** `SpiritSource/dlls/movewith.cpp`, function `HandlePostAssist`

**Problem:**  
`HandlePostAssist` calls itself recursively for every child entity without any
depth limit. A sufficiently deep (or accidentally cyclic) child hierarchy will
overflow the call stack and crash the server.

**Fix:** Added a `depth` parameter (default `0`). When `depth > MAX_MOVEWITH_DEPTH`
the function logs an error and returns immediately, preventing a stack overflow.

---

### 5. Unbounded Recursion – `AssistChildren`

**File:** `SpiritSource/dlls/movewith.cpp`, function `AssistChildren`

**Problem:**  
Same pattern as `HandlePostAssist`: `AssistChildren` recurses into child
entities without a depth limit.

**Fix:** Added a `depth` parameter (default `0`); aborts with an error when the
limit is exceeded.

---

## Performance Issues and Optimizations

### P1. Dead Counting Loop in `CheckAssistList` (Fixed)

**Impact:** O(n) wasted traversal every frame, even when no entity needs assist.

**File:** `SpiritSource/dlls/movewith.cpp`, function `CheckAssistList`

**Problem:**  
A `for` loop walked the entire assist list counting entities flagged
`LF_DOASSIST`. The result was stored in `count`, which was then immediately
reset to `0` before the actual processing loop. The counting was entirely dead
code – its result was never used.

```cpp
// BEFORE (dead counting loop):
int count = 0;
for (pListMember = g_pWorld; pListMember; pListMember = pListMember->m_pAssistLink) {
    if (pListMember->m_iLFlags & LF_DOASSIST)
        count++;
}
// count is immediately reset below – the loop above accomplished nothing
count = 0;
```

**Fix:** The entire dead counting loop (and the now-redundant `count` variable) has
been removed.

---

### P2. `CVAR_GET_FLOAT("sohl_mwdebug")` String Lookup on Every Position Update (Fixed)

**Impact:** Called from `UTIL_AssignOrigin` and `UTIL_SetAngles`, which execute
every frame for every moving entity. `CVAR_GET_FLOAT` performs a string hash
lookup through the engine's cvar table on every call.

**File:** `SpiritSource/dlls/movewith.cpp`

**Problem:**  
```cpp
// BEFORE (string lookup every call):
if (vecDiff.Length() > 0.01 && CVAR_GET_FLOAT("sohl_mwdebug"))
```

The `cvar_t mw_debug` variable is already registered and available in
`game.cpp`. Accessing its `.value` field is a direct memory read with no
string matching.

**Fix:** Added `extern cvar_t mw_debug;` to `game.h` and replaced both
`CVAR_GET_FLOAT("sohl_mwdebug")` calls with `mw_debug.value`.

---

### P3. `MAX_MOVEWITH_DEPTH` Defined After First Use (Fixed)

**File:** `SpiritSource/dlls/movewith.cpp`

**Problem:**  
`#define MAX_MOVEWITH_DEPTH 100` appeared at line ~539, well after
`HandlePostAssist` (line ~53) and `AssistChildren` (line ~130) began using it
following the depth-limit fixes. This relied on the compiler seeing only later
uses, but is confusing and potentially fragile.

**Fix:** Moved the `#define` to the top of the file, immediately after the
`#include` block.

---

### P4. O(n) Insertion in `UTIL_AddToAssistList` (Documented, Not Changed)

**Impact:** Every `UTIL_AddToAssistList` call traverses the entire assist list
from `g_pWorld` to find the tail before appending. With many active entities
this grows quadratic in the worst case.

**Recommended fix (not applied – structural change):**  
Add a `CBaseEntity* g_pAssistListTail` global tail pointer. Maintain it in
`UTIL_AddToAssistList` (set to the newly appended node) and reset it to
`g_pWorld` during `CheckAssistList` after any node is removed. This reduces
all insertions to O(1).

---

### P5. `vecDiff.Length()` Square Root Computed Before Debug Check (Documented)

**Impact:** `Vector::Length()` computes a square root, which is relatively
expensive. Both calls are guarded afterwards by `mw_debug.value`, so when
debug output is disabled the root computation is wasted.

**Recommended fix:**  
Replace `vecDiff.Length() > 0.01` with `vecDiff.LengthSquared() > 0.0001`
(i.e. `0.01²`) to avoid the `sqrt` call. The `LengthSquared` method (or an
inline dot-product) avoids the expensive square root when the result is only
being compared against a threshold.

---

## Summary of Changes Made

| # | Category | Description |
|---|---|---|
| 1 | **Bug Fix** | Null-pointer guard added in `ApplyDesiredSettings` for `LF_DESIRED_INFO` |
| 2 | **Bug Fix** | `g_doingDesired` reset to `FALSE` before early return in `CheckDesiredList` |
| 3 | **Bug Fix** | Corrected error message in `UTIL_SetMoveWithAvelocity` |
| 4 | **Safety** | Added depth limit to `HandlePostAssist` recursion |
| 5 | **Safety** | Added depth limit to `AssistChildren` recursion |
| 6 | **Performance** | Removed dead counting loop from `CheckAssistList` |
| 7 | **Performance** | Replaced `CVAR_GET_FLOAT("sohl_mwdebug")` with `mw_debug.value` |
| 8 | **Correctness** | Moved `MAX_MOVEWITH_DEPTH` `#define` before its first use |
| 9 | **Header** | Added `extern cvar_t mw_debug;` to `game.h` |
