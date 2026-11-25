# Technical Design Document: *Emoji Warlords* (Working Title)

## 1. Overview & Architectural Principles
* **Target Platform:** Mobile-first web browsers (portrait orientation primary, responsive touch controls).
* **Packaging:** Standalone, single-file `.html` containing all CSS, JavaScript, game data tables, rendering engine, AI, and embedded unit test suite. Zero external runtime dependencies.
* **Design Philosophy:**
  * **Data-Driven Tables:** All unit definitions, faction rosters, terrain movement costs, terrain combat modifiers, and production schedules are stored in declarative configuration tables (`CONFIG.*`).
  * **Event / Hook System:** Game mechanics communicate via a lightweight publish/subscribe hook system (`Hooks.on(event, fn)` / `Hooks.emit(event, payload)`), keeping core combat, movement, AI, UI rendering, and test assertions cleanly decoupled.
  * **Deterministic & Pure Core:** All state mutations operate on a single serializable `gameState` object, enabling trivial JSON save/load, undo/replay debugging, and headless unit testing.

---

## 2. Core Data Tables (`CONFIG`)

### 2.1 Terrain Definitions (`CONFIG.TERRAIN`)
Each map tile has a `terrain` key, movement cost (in MP), and visual representation.

| ID | Name | Base Color | Move Cost (MP) | Combat Notes |
| :--- | :--- | :--- | :--- | :--- |
| `PLAINS` | Plains | `#8bc34a` | `2` | Baseline terrain; cavalry bonus. |
| `WOODS` | Woods | `#388e3c` | `3` | High defense for Elves/Beasts; cavalry penalty. |
| `MOUNTAINS` | Mountains | `#78909c` | `5` | High defense; impassable to heavy cavalry unless flying. |
| `SNOW` | Snow | `#eceff1` | `4` | Movement penalty for non-adapted units; Undead/Frost bonus. |
| `TOWN` | Town | `#d7ccc8` | `2` | Overlay structure on a tile. Defender always strikes first. |

### 2.2 Factions & Unit Rosters (`CONFIG.FACTIONS` & `CONFIG.UNITS`)
All units use single-codepoint people/creature emojis. Each faction has 5 unit tiers (Scout, Core Infantry, Heavy/Elite, Cavalry, Apex/Flyer).

#### Faction A: The Crown Empire (`EMPIRE`) — Color: `#1976d2`
| Unit ID | Emoji | Name | Cost (Buy Prod.) | Build Turns | Max HP | ATK | DEF | Max MP | Traits / Hooks |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `EMP_SCOUT` | 🥷 | Shadow Scout | `100` | `1` | `10` | `3` | `2` | `12` | `WOODS_STRIDE` (MP cost -1 in Woods) |
| `EMP_GUARD` | 💂 | Royal Guard | `150` | `1` | `14` | `4` | `5` | `8` | `TOWN_DEFENDER` (+2 DEF in Towns) |
| `EMP_PALADIN` | 🦸 | Paladin | `350` | `2` | `20` | `7` | `7` | `8` | `HOLY_AURA` (+1 ATK to band allies vs Undead) |
| `EMP_KNIGHT` | 🏇 | Heavy Knight | `400` | `3` | `22` | `8` | `6` | `12` | `CHARGE` (+2 ATK on Plains), `NO_MOUNTAIN` |
| `EMP_EAGLE` | 🦅 | Giant Eagle | `600` | `4` | `24` | `9` | `6` | `16` | `FLYING` (All terrain MP cost = 2) |

#### Faction B: Sylvan Covenant (`SYLVAN`) — Color: `#2e7d32`
| Unit ID | Emoji | Name | Cost (Buy Prod.) | Build Turns | Max HP | ATK | DEF | Max MP | Traits / Hooks |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `SYL_PIXIE` | 🧚 | Forest Pixie | `100` | `1` | `8` | `3` | `3` | `14` | `FLYING` |
| `SYL_ELF` | 🧝 | Sylvan Archer | `160` | `1` | `12` | `5` | `4` | `10` | `WOODS_MASTER` (+1 ATK, +1 DEF in Woods) |
| `SYL_TROLL` | 🧌 | Moss Troll | `360` | `2` | `26` | `7` | `6` | `8` | `REGENERATE` (+2 HP at start of owner's turn) |
| `SYL_UNICORN` | 🦄 | Unicorn | `420` | `3` | `20` | `8` | `7` | `14` | `WOODS_STRIDE`, `PLAINS_BONUS` (+1 ATK) |
| `SYL_DRAGON` | 🐉 | Emerald Dragon | `750` | `5` | `32` | `11` | `9` | `16` | `FLYING`, `MOUNTAIN_HOME` (+2 DEF in Mountains) |

#### Faction C: Blight Legion (`UNDEAD`) — Color: `#6a1b9a`
| Unit ID | Emoji | Name | Cost (Buy Prod.) | Build Turns | Max HP | ATK | DEF | Max MP | Traits / Hooks |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `UND_BAT` | 🦇 | Dread Bat | `90` | `1` | `8` | `3` | `2` | `14` | `FLYING` |
| `UND_SKEL` | 💀 | Skeleton | `120` | `1` | `12` | `4` | `4` | `8` | `SNOW_WALKER` (MP cost = 2 in Snow) |
| `UND_GHOUL` | 🧟 | Rotting Ghoul | `300` | `2` | `22` | `7` | `5` | `8` | `SNOW_BONUS` (+2 ATK, +2 DEF in Snow) |
| `UND_WOLF` | 🐺 | Dire Wolf | `350` | `2` | `18` | `8` | `5` | `14` | `PACK_HUNTER` (+1 ATK per friendly wolf in band) |
| `UND_DEMON` | 👹 | Abyssal Oni | `700` | `4` | `30` | `11` | `8` | `10` | `MOUNTAIN_STRIDE` (MP cost = 2 in Mountains) |

### 2.3 Terrain Modifier Table (`CONFIG.TERRAIN_MODS`)
Applied dynamically during movement and combat calculations via table lookup, overridden or extended by unit trait hooks.

```javascript
TERRAIN_MODS: {
  PLAINS:    { defaultAtk: 0, defaultDef: 0 },
  WOODS:     { defaultAtk: 0, defaultDef: 1 },
  MOUNTAINS: { defaultAtk: 0, defaultDef: 2 },
  SNOW:      { defaultAtk: -1, defaultDef: 0 },
  TOWN:      { defaultAtk: 0, defaultDef: 2, firstStrike: true }
}
```

### 2.4 XP & Veterancy Table (`CONFIG.XP_LEVELS`)
Units gain XP for surviving combat (`+10 XP` per battle participated, `+15 XP` bonus for landing a killing blow).

| Level | XP Threshold | Stat Bonus | Badge Indicator |
| :--- | :--- | :--- | :--- |
| `1` | `0` | Base Stats | None |
| `2` | `25` | `+1 ATK`, `+1 DEF`, `+2 MaxHP` | `•` |
| `3` | `60` | `+2 ATK`, `+2 DEF`, `+4 MaxHP` | `••` |
| `4` | `110` | `+3 ATK`, `+3 DEF`, `+6 MaxHP` | `★` |

---

## 3. Game Mechanics & Rules Engine

### 3.1 Procedural Map Generation (`MapGen`)
* **Grid Size:** Configurable (Default: `16 x 22` tiles, optimized for vertical mobile scrolling/panning).
* **Generation Pipeline:**
  1. Seeded 2D value/simplex noise generates elevation (`MOUNTAINS` > `0.75`, `PLAINS`/`WOODS` middle, `SNOW` at extreme north/south latitudes or high elevation).
  2. **Town Placement:** Places `12–16` towns across valid non-mountain tiles with a minimum Manhattan/Euclidean distance constraint (`minDist = 4`) so towns are evenly distributed.
  3. **Starting Positions:** Selects 3 maximally separated towns (farthest-point sampling) for the 3 factions. Each starting town begins with `1` Tier-1 Scout and `1` Tier-2 Core Infantry unit, and starts with Tier-2 Core Infantry already installed in its production queue.
  4. **Connectivity Guarantee:** Runs a flood-fill/A* check from each starting town to ensure no town is isolated by impassable terrain.

### 3.2 Movement, Bands & Stacking (`MovementSystem`)
* **Tile Capacity:** Up to `6` friendly units can occupy a single tile.
* **Banding & Disbanding:**
  * Each unit has an individual `bandId` (or `null` if ungrouped).
  * **Band Action:** Groups selected friendly units on the same tile into a shared `bandId`. When a band moves, it moves together at the **lowest remaining MP** among its members, paying the **highest terrain MP cost** required by any member on each step.
  * **Disband Action:** Clears `bandId` for selected units (or splits a subset out), allowing individual units with remaining MP to move independently.
* **Pathfinding & Next-Unit Cycling:**
  * Tapping a unit/band computes reachable tiles via Dijkstra's algorithm and highlights valid destination tiles.
  * Tapping a reachable tile moves the unit/band step-by-step along the shortest path.
  * **Next Unit Button:** Cycles focus to the next friendly unit/band with `mp > 0` that has not been marked `skip/wait` for the turn.

### 3.3 Combat Resolution (`CombatSystem`)
* **Initiation:** Long-pressing an adjacent or reachable hostile unit/band or enemy/neutral town automatically moves to the best adjacent tile (if needed) and initiates combat if the attacker has `mp >= 1` (attacking costs `2 MP` or all remaining MP if `< 2`).
* **Town Defense Rule:** Neutral towns automatically spawn `1` neutral militia unit (`NEU_MILITIA`) at map generation so capturing towns always involves tactical combat. Towns themselves do not have HP.
* **Turn-Based Dice Simulation:**
  1. **Combatants Lineup:** Attacker stack (`A[]`) and Defender stack (`D[]`) are ordered by unit role (frontline infantry/cavalry first, heroes/flyers/ranged behind).
  2. **Initiative & Simultaneous Exchange:** Emit `onCombatInit` hook. If Defender is inside a `TOWN` tile, Defender gets **First Strike on Round 1 only** (attacker counter-strikes if it survives). All open-field rounds and Round 2+ in Towns resolve **simultaneously**.
  3. **Dice Exchange Loop (1v1 Duel until one unit dies, then next unit steps up on the next round):**
     * Compute effective `ATK` and `DEF` each round by querying `CONFIG.TERRAIN_MODS` and firing `onCalculateCombatStats` hooks (`CHARGE` applies on Round 1 only; `PACK_HUNTER` and `HOLY_AURA` scale from living `hp > 0` allies).
     * Roll `1d6` per point of `ATK` (hits on `4+`) vs `1d6` per point of `DEF` (blocks on `5+`), dealing `Math.max(1, atkHits - defBlocks)` damage per strike so combat never stalemates (no round cap).
     * Emit `onCombatRound` hook (records dice rolls for the battle replay modal).
  4. **Resolution:**
     * Surviving units receive XP via `onUnitGainXP` hook.
     * If all defenders are defeated and at least one attacker survives, the surviving attacking unit/band advances onto the defender's tile (and captures the town if it was a town tile).

### 3.4 Economy & Town Production (`EconomySystem`)
* **Income:** Each owned town generates `+25 Gold` per turn (plus optional variance `20–35 Gold` assigned per town at map generation).
* **Production Slots:** Each town has **1 active production slot** (`producingUnitId`, `turnsRemaining`).
* **Buying & Switching Production:**
  * The player opens a town's management sheet, views the faction's 5 available unit blueprints, and pays a **one-time Unlock Cost** (`unit.cost` in Gold) to permanently unlock that blueprint for that town (`town.unlockedUnits`).
  * Once unlocked, switching between any unlocked blueprints in that town is **free**. If a town is captured and later recaptured, the faction's unlocked blueprints in that town are preserved (only the active queue resets on capture).
  * **Queued Switching:** If the town is idle, production starts immediately. If the town is already mid-production, switching queues the new blueprint (`nextProdUnit`) so the current unit finishes and spawns first before switching automatically.
  * Once active, the town automatically produces `1` unit of that type every `unit.buildTurns` turns indefinitely (until the player switches to another blueprint).
  * **Spawn Placement & Overflow:**
    * If the town tile has `< 6` units (`MAX_STACK`), the produced unit spawns directly on the town tile and `turnsRemaining` resets to `unit.buildTurns`.
    * **Overflow:** If the town tile is full (`6` units), the engine scans the 8 adjacent tiles (top-left to bottom-right) and spawns the unit on the first in-bounds tile that is not `MOUNTAINS`, contains no hostile/neutral units, and has `< 6` friendly units.
    * **Blocked / Deferred:** If both the town tile and all valid adjacent tiles are full or blocked, spawning is deferred with `turnsRemaining = 0` (the active blueprint is preserved and no progress is lost) and retried automatically at the start of the owner's next turn until space clears.

### 3.5 Victory & Elimination (`WinConditionSystem`)
* Checked immediately after any town capture (`onTownCaptured` hook):
  * If a faction reaches `0` owned towns, that faction is immediately eliminated: all of its remaining units on the map are destroyed (`onFactionEliminated` hook).
  * If only `1` faction remains with towns, trigger the Victory / Defeat end-of-game screen.

---

## 4. Hook Architecture (`Hooks`)
To keep code modular and avoid sprawling `if/else` chains, all systems register handlers on a central `Hooks` registry:

```javascript
const Hooks = {
  _listeners: {},
  on(event, fn) { (this._listeners[event] ??= []).push(fn); },
  emit(event, ctx) {
    for (const fn of (this._listeners[event] || [])) fn(ctx);
    return ctx;
  }
};
```

**Core Hook Events:**
* `onTurnStart` (`{ state, factionId }`) — Income collection, production ticks, `REGENERATE` trait healing, MP reset.
* `onMoveCost` (`{ unit, tile, cost }`) — Trait hooks (`WOODS_STRIDE`, `FLYING`, `SNOW_WALKER`) modify `ctx.cost`.
* `onCalculateCombatStats` (`{ unit, tile, allies, enemies, atk, def }`) — Trait hooks (`HOLY_AURA`, `PACK_HUNTER`, `CHARGE`, `WOODS_MASTER`) modify `ctx.atk` / `ctx.def`.
* `onCombatInit` (`{ attackerBand, defenderBand, defenderTile, defenderStrikesFirst }`) — Sets `defenderStrikesFirst = true` when `defenderTile.hasTown`.
* `onUnitDeath` (`{ unit, killer, tile }`) — Awards kill XP, cleans up empty bands.
* `onTownCaptured` (`{ town, oldOwner, newOwner, state }`) — Resets town production, checks faction elimination and victory condition.

---

## 5. AI Opponent Controller (`AIController`)
Executed sequentially for the two PC factions during their turn with a `240ms` step delay and `200ms` unit glide animations so players can watch enemy movements and attacks:
1. **Production Phase:** For each owned town with no production (or when sitting on excess gold > `450`), purchase the highest-tier affordable unit blueprint (weighted toward Flyers/Cavalry when towns are far apart, or Infantry when enemies are within 4 tiles).
2. **Banding Phase:** Group co-located idle units into bands of `2–4` units before moving.
3. **Target Scoring & Movement:** Each unit/band evaluates visible/known targets using a utility score:
   * Unprotected neutral or enemy towns within reach (`+100` priority).
   * Weakened enemy bands (`+60` priority).
   * Nearest uncaptured town (`+30` priority).
   * Moves via A* toward the highest-scoring target and initiates combat if adjacent.

---

## 6. Mobile UI / UX Layout
Designed for one-handed portrait play (`100dvh` viewport, no page bounce):

1. **Top Status Bar (48px):**
   * Current Faction Badge & Color | Gold Balance (`💰 240 (+75/t)`) | Town Count (`🏰 3/14`) | Menu Button (`☰`).
2. **Interactive Canvas Map (Flexible Center):**
   * HTML5 `<canvas>` rendering crisp terrain colors, subtle grid lines, town icons (`🏰` tinted with owner faction badge), unit emojis, stack count badges (`×3`), HP bars, `200ms` smooth movement animations, and a high-contrast reachable-tile overlay (cyan border + dimmed unreachable tiles).
   * Supports single-finger pan, Google Maps-style pinch-to-zoom anchored to the finger midpoint, **short-tap** to select/inspect (friendly units, enemy stacks, towns), and **long-press (`~450ms`)** to move/stack or move-and-attack.
3. **Bottom Contextual Action Drawer (132px+):**
   * **When Unit/Band/Tile Selected:** Displays unit emoji(s), HP, ATK/DEF, XP, remaining MP, a horizontally scrollable `#stack-strip` of clickable unit & `🏰` town chips (for both friendly and enemy/neutral tiles), and touch-friendly buttons: `[Band / Disband]`, `[Wait]`, `[Next Unit ⏭️]`, `[End Turn]`.
   * **When Town Selected:** Displays town name, gold yield, current production progress (`🐉 2/5 turns`), and `[Buy Production]` button opening a clean modal list of faction units.
   * **Battle Modal:** Compact animated dice-roll log showing each duel round, First Strike indicator (if Town defense), damage dealt, and XP gained, with `[Skip]` / `[Close]`.
4. **Main / Pause Menu Modal (`☰`):**
   * `[New Game (Pick Faction)]`
   * `[Save Game]` / `[Load Game]` (uses `localStorage` with multiple slots + JSON export/import text fallback)
   * `[Run Unit Tests 🧪]`

---

## 7. Save / Load Serialization (`StorageSystem`)
* **State Schema:**
  ```json
  {
    "version": 1,
    "turn": 14,
    "activeFaction": "EMPIRE",
    "humanFaction": "EMPIRE",
    "factions": { "EMPIRE": { "gold": 320, "alive": true }, ... },
    "map": { "width": 16, "height": 22, "tiles": [...] },
    "towns": [ { "id": "t1", "x": 4, "y": 5, "owner": "EMPIRE", "prodUnit": "EMP_GUARD", "turnsLeft": 1 } ],
    "units": [ { "id": "u1", "type": "EMP_KNIGHT", "owner": "EMPIRE", "x": 4, "y": 6, "hp": 22, "mp": 12, "xp": 10, "bandId": "b1" } ]
  }
  ```
* **Persistence:** Saves to `localStorage` under `emoji_warlords_save_1..3` plus an automatic `autosave` at the start of every human turn.

---

## 8. Embedded Unit Test Suite (`TestRunner`)
Accessible directly from the in-game menu (`Run Unit Tests 🧪`), rendering a pass/fail diagnostic overlay inside the single HTML file without needing external test frameworks.

### Required Test Cases:
1. **`test_map_generation_connectivity`:** Generates a map, verifies 3 distinct starting towns, verifies minimum spacing between towns, and runs pathfinding to confirm all towns are reachable.
2. **`test_terrain_movement_and_traits`:** Verifies base MP deduction across Plains/Woods/Mountains/Snow, and verifies trait hooks (`FLYING` flat cost, `WOODS_STRIDE` discount, `NO_MOUNTAIN` block).
3. **`test_banding_and_disbanding`:** Places 3 units with different MPs (`8`, `12`, `16`) on one tile, bands them, verifies band MP equals `8` (minimum), moves 1 tile, disbands, and verifies remaining units retain their unspent individual MP.
4. **`test_town_first_strike_combat`:** Places an attacker outside a town and a defender inside a town, hooks `onCombatRound`, and asserts the defender executes the first attack roll of the battle.
5. **`test_xp_and_leveling`:** Simulates combat victory, verifies XP award, triggers level-up threshold, and asserts increased `ATK`/`DEF`/`MaxHP`.
6. **`test_economy_and_production_loop`:** Sets town gold, buys a `2-turn` unit blueprint, advances 2 turns via `endTurn()`, and asserts gold deduction, turn income addition, and unit spawning on the town tile.
7. **`test_faction_elimination_and_victory`:** Captures a faction's last remaining town and asserts that all surviving units belonging to that faction are immediately removed and victory triggers when 1 faction remains.
8. **`test_save_load_roundtrip`:** Serializes a mid-game state to JSON, mutates the live state, reloads from JSON, and asserts deep equality of map, towns, units, bands, and gold.
9. **`test_mixed_band_reachability_and_stack_cap`:** Verifies a `FLYING` + `NO_MOUNTAIN` band is blocked by Mountains and cannot enter a tile that would exceed `MAX_STACK` (`6`).
10. **`test_ai_turn_execution`:** Verifies `AIController.runTurn` executes cleanly.

---

## 9. Suggested Implementation Order for the Coding Agent
1. **Scaffold Single HTML + CSS Layout:** Mobile viewport, canvas container, bottom HUD drawer, and modal containers.
2. **Implement `CONFIG` Tables & `Hooks` Bus:** Populate all terrain, faction, unit, and trait definitions.
3. **Build Pure State & `MapGen`:** Implement deterministic state factory and procedural map/town generator.
4. **Build `MovementSystem` & Banding Logic:** Pathfinding, MP validation, banding/disbanding, and next-unit selector.
5. **Build `CombatSystem`:** Dice resolution, Town First Strike hook, terrain/trait stat modifiers, and XP progression.
6. **Build `EconomySystem` & `WinConditionSystem`:** Gold ticks, production queue buying/spawning, and town-capture faction wipeout.
7. **Implement Canvas Renderer & Touch Input:** Pan/zoom, tile rendering, unit stacking badges, and contextual HUD binding.
8. **Implement `AIController` & `StorageSystem`:** AI turn loop and `localStorage` save/load.
9. **Implement `TestRunner` & Run All 8 Unit Tests:** Verify all tests pass green from the menu.
