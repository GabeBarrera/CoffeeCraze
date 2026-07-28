# CafeCraze — Pixel Brew

A single-file, mobile-shaped browser game: run an espresso bar, take customer
orders, grind / tamp / pull / steam each drink, and beat the ticket timers before
patience (5 hearts) runs out. Days run 6:00 AM → 3:00 PM of in-game time; cash
carries over and buys supplies.

Everything is one Design Component (`index.html`) plus flat PNG sprite sheets.
No build step, no dependencies — open `index.html` in a browser.

---

## File structure

```
.
├── index.html                        # the whole game (template + logic)
├── support.js                        # Design Component runtime (generated — do not edit)
├── README.md
└── assets/
    ├── characters/
    │   ├── barista_gabe.png          # menu / settings portraits
    │   └── barista_daniella.png
    ├── scene/
    │   ├── cafe-order-scene.png      # backdrop (behind NPCs)
    │   └── cafe-order-scene-top.png  # counter overlay (occludes NPC torsos)
    └── sprites/
        ├── npc-sheet.png             # 8 customers, uniform cells, 1 row
        ├── items-sheet.png           # tamp, milk, espresso, cup, beans, cocoa, glasses, note
        ├── drinks-sheet.png          # espresso, americano, latte, cortado, mocha
        ├── grinder-sheet.png         # 6 grinder states
        ├── machine-sheet.png         # 6 espresso-machine states
        └── portafilter-sheet.png     # clean / grounds / tamped
```

All artwork is downscaled to ~1.5–2× its largest on-screen size and rendered with
`image-rendering: pixelated`, keeping the pixel look while holding the whole
asset set under ~9 MB.

---

## Sprite sheets

Two packing schemes, two helpers. Both compute pure percentages, so a sheet can
be rescaled wholesale without touching the code.

**Uniform sheets** — every frame the same cell size, laid out in one row
(`grinder`, `machine`, `portafilter`, `npc`):

```js
spriteBg(url, frameCount, index, aspectRatio, extraCss)
// background-size: frameCount*100% 100%
// background-position: index/(frameCount-1)*100% 0
```

**Strip-packed sheets** — frames of differing native sizes, top-aligned, with a
metadata table of `{w, h, left}` per frame (`items`, `drinks`):

```js
const ITEMS_SHEET = { totalW, totalH, frames: { beans:{w,h,left}, … } };
spriteBgN(sheet, url, frameKey, extraCss)
```

NPCs use the uniform path: `npc-sheet.png` is 8 cells of 313×932 (each source
figure centred horizontally, top-aligned), so an order only stores a frame index
`0–7` and `npcSprite(idx, extra)` renders it.

Sprites are painted as `background-image` on `<div>`s — never `<img src>` — so
one HTTP request serves every state and swapping state is just a changed
`background-position`.

---

## Code structure (`index.html`)

```
<x-dc>                          — template: markup with {{ value }} holes
  helmet                        — fonts, resets, @keyframes
  MAIN MENU                     — barista pick, START SHIFT
  GAME
    stage (1536×2752 aspect)    — backdrop → NPCs → counter overlay → equipment
      order tickets             — drink icon, name, draining timer bar
      grinder / beans / cups    — drag sources & drop targets (data-drag / data-drop)
      espresso machine          — portafilter dock, steam wand, water nozzle
      portafilter / tamp / trash
      drink stack               — finished drinks awaiting a customer
    HUD                         — day, clock, cash, hearts, stock, OPEN, ☰ menu
  SHOP · DAY END · GAME OVER    — full-screen states
  modals                        — instructions, recipes, highscores, settings, note
  #dragGhost                    — follows the pointer while dragging
</x-dc>

<script data-dc-script>         — logic
  A                             — asset paths (scene, characters, 6 sheets)
  *_SHEET / *_FRAMES            — sprite metadata + frame maps
  spriteBg / spriteBgN          — sheet → CSS background
  npcSprite / itemSprite / drinkSprite / orderIcon
  RECIPES, NPCS, NAMES, DRINKS, PRICE, DLABEL, TUT
  tuning consts                 — ORDER_TIME, SPAWN, BREW_T, GRIND_T, STEAM_RATE, DAY_START/END
  fresh()                       — a new run's state
  save/load                     — localStorage: cafecraze_savegame, pixelBrewHighscores

  class Component extends DCLogic
    componentDidMount           — pointer listeners, 100 ms tick, restore save, preload
    tick(dt)                    — grind, brew, steam, water mini-game, clock, spawns, misses
    grab / onMove / onUp        — pointer drag; ghost styled from a sheet frame
    handleDrop(item, targets)   — every crafting & serving rule
    detect(cup)                 — ingredients → espresso | americano | latte | cortado | mocha
    actions                     — brew, stopBrew, takeShot, steam, water pour, shop, menus
    renderVals()               — every {{ hole }} the template reads
}
```

### State model

A drink is a small ingredient record — `{esp, water, milk, cocoa, short}` (or
`{plainWater:true}`) — and `detect()` maps it to a menu item at serve time, so
recipes are combinations rather than hard-coded buttons. Machine visuals derive
from state via `machineKey(state)` rather than being set imperatively.

A lone `short` shot displays as **RISTRETTO** in the drink stack (still the
espresso sprite); combined with milk it becomes a **CORTADO** instead of a latte.

### Brewing: full shot vs. cortado half-shot

The brew bar (`brewProgress`, 0→1 over `BREW_T`) has a highlighted hitbox zone
(`HITBOX_START`–`HITBOX_END`, centered on `HITBOX_MID`). Stopping the brew:

- **before** the hitbox → pauses (`machine:'paused'`); BREW resumes it in place.
- **inside** the hitbox → finalizes a half shot (`brewShort:true`), sets
  `puckState:'short'`, and leaves the portafilter docked.
- **past** the hitbox (or letting it run to completion) → a full shot,
  `puckState:'spent'`.

A `'shot'` sitting on the machine can be dragged to a customer/queue, or simply
tapped/clicked (a `data-drop="serve"` overlay that only exists once
`machine==='shot'`, so it never blocks the portafilter dock while a shot is
still brewing). Dropping a shot anywhere except a matching order ticket sends
it to the drink queue automatically instead of doing nothing.

With `puckState:'short'`, pressing BREW again with a fresh cup reuses the same
puck: the bar runs and auto-stops exactly at `HITBOX_MID`, producing a second
ristretto — after which `puckState` becomes `'spent'` and the portafilter is
forced off the machine. Either way, once `puckState==='spent'` the machine
won't brew again until a freshly tamped portafilter is docked.

Two ristretto shots (one in the machine and one queued, or two queued) can be
merged: dragging/dropping a ristretto onto another queued ristretto (a
`data-drop="ristretto"` target added only to ristretto entries) turns the
target into a full espresso and consumes the dragged one.

### Persistence

- `cafecraze_savegame` — SAVE GAME in the ☰ menu; restored on load (legacy saves
  that stored NPC image paths are migrated to frame indices).
- `pixelBrewHighscores` — top 5 runs by day reached, then earnings.

---

## Tuning

Gameplay knobs sit together near the top of the logic block: `ORDER_TIME` (ticket
seconds), `SPAWN` (seconds between customers), `BREW_T`, `GRIND_T`, `STEAM_RATE`,
`MIN_PER_SEC` (clock speed), `DAY_START` / `DAY_END`, the `PRICE` table, and
`HITBOX_START` / `HITBOX_END` / `HITBOX_MID` (the cortado half-shot window).
