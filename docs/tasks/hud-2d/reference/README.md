# roam-game-ui-v4-02 - ROAM HUD - Twin pods - 2D (baked) - hand-off for the game

## Prompt for the integrating agent

**See it:** the UI Lab, https://roam-game-ui.astansler.dev/v4/#v2-cruise (tick "2D baked"; EN / RU switch next to it). **Reference screenshot** of the cab exactly as the lab shows it (3D, Cruise, over a game picture): `assets/screenshot.jpg` in this folder, also https://roam-game-ui.astansler.dev/v4/flat/baked/roam-game-ui-v4-02/screenshot.jpg. The 2D bundle must look like that.

You are putting the ROAM / Road Machiners driving HUD "Twin pods" into the game. This bundle is the HUD as plain 2D web code: a picture of the cab baked once from a fixed camera, and every live part (screens, dials, keys, toggles, LEDs, lamps, menu devices) drawn on top with DOM, canvas 2D and CSS transforms. There is no WebGL and no three.js at run time.

Do:
1. Put the folder (e.g. `src/hud/roam-game-ui-v4-02/`), keeping the file names. All imports are relative ES modules.
2. Run `index.html` from that folder over any static server to see it working with the sample state. It must look like the lab's "2D baked" mode.
3. Mount it over the game view with `mountHud(viewElement, { state })` (see API). `viewElement` is the 16:9 box that holds the game picture; the HUD scales with its width.
4. Feed it the game state: build an object with the fields in "State contract" from the game's own data (an adapter function), pass it in, and call `hud.update(state)` whenever the game state changes. Per-frame motion runs by itself (or call `hud.tick(dt)` from the game loop with `loop: false`).
5. Wire the controls: the click and hotkey handlers (`ACT` and `press` in layout.js, the menu handlers in menus.js) edit the state object directly, as the lab mock does. Replace their bodies with the game's commands (the table "Controls" says what each one is), and let the game push the new state back with `hud.update()`.
6. Pass world clicks through: only the live parts take the pointer; for clicks on the cab's still hardware call `hud.covers(clientX, clientY)` and do not send those to the world.
7. Load the fonts (Anonymous Pro, Roboto Slab, Caveat, see index.html) before mounting, or the first canvases draw with fallback fonts until `document.fonts.ready` redraws them.
8. Keep the `assets/` folder next to bake.js and screens.js (both find it relative to themselves), or move it and change `BAKE.dir` in bake.js and the icon URL in screens.js.
9. Language: `mountHud(el, { state, lang: 'ru' })` or `hud.setLang('en' | 'ru')` switches every word the HUD itself prints (panel labels, label tape, keys, dial and screen words; section "Languages"). Text that comes in the state (log lines, names, radio, contracts, the context action) must arrive from the game already in the player's language.
10. Screens can host the game's real interface: `hud.slot('log.screen1')` returns an element laid on that screen in its perspective (section "Slots"). Use it for the Log device (the road log list) and anywhere the game already has a working UI.

Do not:
- put any filter, tint or overlay on the game picture, or a constant filter over the whole UI;
- use sans-serif fonts (all text must cover Latin and Cyrillic); call the currency M / M's (М in Russian), never money;
- move or restyle parts by hand in the baked pictures: change the layout in the lab (`variants.js`), re-bake with `v4/flat/bake.html`, and copy a new bundle;
- add three.js or WebGL for the HUD.

## How it works

Design space: 2560 × 1440 px, scaled to the host width as one piece (`FlatStage`). Each layer (`FlatLayer`) is, bottom to top:

1. **still picture** (`<img>`, WebP with alpha): the cab rendered once by three.js from the lab's camera (camera z 24, vertical FOV 21.24°), without levers, knob bodies and needles, with screens, key tops and dials blank and no label text: there is no text in the pictures except makers' plates and round logos;
2. **colour masks** (atlas PNG, used as CSS `mask-image`, coloured with `background-color`): toggle lamps, warning lamps, LED cells;
3. **cut-out frames** (atlas PNG as `background-image`): toggle lever poses (5 angles), knob positions, the sunk pose of each key, each with its shadow; one frame shows at a time;
3a. **live text** (DOM spans on the panel planes, see "Languages"), then the **lever and knob bodies** alone over it, so a lever in front of its label hides it as in 3D;
4. **faces** (`<canvas>` with a CSS `matrix3d` that is the exact perspective of that face): screens, key tops, dial faces. They are drawn by the same functions as the 3D cab (`screens.js`, the key and dial code in flat.js); a face partly hidden by hardware in front gets a mask from the atlas;
5. **needles**: one flat needle sprite laid into the dial plane with `matrix3d`, turned with `rotate()`;
6. **hit shapes**: invisible polygons for everything clickable (`pointer-events: auto` only there).

The menu devices (Log, Truck) are layers of their own, baked in place over this cab (their shadow on the cab is in their picture). They slide up from below when `state.open` is `"log"` or `"truck"`.

Cost per frame: no draw calls at all; each live part only checks its timer, and a canvas is redrawn only when its `every` is due (or on `hud.update`). Needle and lever motion changes one CSS transform or one `display`.

## Files

| file | what |
|---|---|
| roam-hud.js | `mountHud()`: the public API (mount, update, tick, covers, open, destroy), click and hotkey routing |
| flat.js | the 2D runtime: `FlatStage` (design space and scale), `FlatLayer` (builds the DOM for one baked layer and drives it from the state) |
| layout.js | this layout: `VARIANT.build()` lists the live parts in order (same code as the 3D cab), `ACT` (control handlers), `HOTKEYS` |
| menus.js | the Log and Truck devices (same code as in 3D) |
| kit-flat.js | the cab kit API without three.js: records the live parts, colour helpers, screen palettes |
| screens.js | what every screen draws: `(ctx, W, H, state, time, palette)` functions |
| bake.js | `BAKE`: pictures, atlases, and per part: canvas size, `matrix3d`, frame and mask rects, hit polygons |
| sample-state.js | one full state (Cruise) to start from |
| i18n.js | the HUD's own words in Russian (`LANGS`, `makeTr`); add a language there |
| fonts.js, three-stub.js | font names; a no-op stand-in for the few THREE calls left in layout code |
| index.html | demo page: the game picture with the HUD on top |

Parts in `VARIANT.build().parts` and `BAKE.layers.hud.els` match one to one, in order. `FlatLayer` throws if they do not (re-bake).

## API

```js
import { mountHud } from './roam-hud.js';
const hud = mountHud(viewEl, {
  state,            // the state object (see State contract)
  keys: true,       // listen for the hotkeys on window
  loop: true,       // run its own requestAnimationFrame; false → call hud.tick(dt) yourself
  onChange: s => {},// after a click or hotkey changed the state
  history: [],      // older log lines for the Log device: [time, text, tone]
  lang: 'en',       // 'en' | 'ru': the HUD's own words (i18n.js)
  tr: null,         // or your own s => translated
});
hud.update(nextState);          // new state (or the same object, changed): redraws everything that reads it
hud.tick(dtSeconds);            // with loop: false
hud.covers(clientX, clientY);   // true over HUD hardware or a control
hud.setLang('ru');              // switch language live
hud.translate(s => dict[s] ?? s); // or your own translator
hud.slot('log.screen1');        // a screen as a slot for the game's own UI (see Slots)
hud.screens();                  // [{ id, draw, width, height }]
hud.pick(clientX, clientY);     // the handler under the point, or null
hud.open('log' | 'truck' | null);
hud.destroy();
```

## State contract

Fields this HUD reads (sample values in sample-state.js):

| field | type | meaning | read by |
|---|---|---|---|
| `speed` | number | km/h now | cab |
| `limit` | number | top speed allowed now, km/h (drops with low fuel or a worn engine) | cab |
| `heading` | number | compass heading 0–359° | cab |
| `fuel` | number | litres in the tank (warn ≤ 25, bad ≤ 10) | cab, truck |
| `tank` | number | tank size, litres | truck |
| `money` | number | M's the player has (never call it money in the UI) | cab, log |
| `supplies` | number | food and water units | cab, truck |
| `supMax` | number | supply capacity | cab, truck |
| `day` | number | game day | cab, log |
| `time` | string | game clock "h:mm" | cab |
| `eng` | number | engine °C (warn ≥ 95, bad ≥ 112, fire at 120) | cab |
| `health` | number | driver health 0–100 (warn ≤ 60, bad ≤ 30) | cab |
| `load` | number | cargo kg | cab, truck |
| `loadMax` | number | cargo capacity kg | cab, truck |
| `power` | number | power used | cab, truck |
| `powerMax` | number | power available | cab, truck |
| `sw` | object | toggles {route, patch, lights, overdrive}: 1 on / 0 off | cab |
| `locked` | object | controls that cannot be used now {overdrive, cool, aid, …}: 1 = locked | cab |
| `hot` | object, optional | controls that are on and causing trouble {overdrive}: 1 = hot | cab |
| `parts` | array | truck parts [{k, n, hp 0–100, pos}] (ok ≥ 70, damaged 40–69, critical < 40) | cab, truck |
| `weapons` | array | up to 3 slots [{n, ic, ammo, mag, st, cyc, stat, tgt?, util?} \| null] | cab |
| `fire` | object | fire toggles {auto, all, show}: 1 / 0 | cab |
| `radio` | string | latest radio line (pager) | cab |
| `log` | array | road log lines [time, text, tone: "" \| good \| bad \| warn \| dim], newest first | cab, log |
| `objective` | object | {t: title, lines: [[text, detail]], ctx: [key, label] \| null}: contracts and the context action | cab, log |
| `turn` | number, optional (default 2) | turn speed knob index 0–2 (1×, 2×, 4×) | cab |
| `pressed` | string, optional | id of a screen key pressed just now (lab flash, 260 ms) | cab |
| `ui` | object, made by the menus | menu UI state {filter, scroll, sel, tab, say} | log, truck |
| `cargo` | object, made by the Truck menu | bed cargo {good: crates} | truck |

## Controls

Hotkeys (`HOTKEYS` in layout.js): R, P, L, O, G, U, Space, C, I, M, J, Esc, E.

| control | hotkey | handler | in the game |
|---|---|---|---|
| Route / Manual toggle | R | `ACT.sw('route')` | Route follows roads and drives around obstacles; Manual drives straight at the point |
| Auto Patch toggle | P | `ACT.sw('patch')` | while parked and out of a fight the driver patches damaged parts (spends spare parts) |
| Lights toggle | L | `ACT.sw('lights')` | see farther at night; others see you farther |
| Overdrive toggle | O | `ACT.sw('overdrive')` | faster, engine heats quickly; locked when the engine is too worn |
| Cool | G | `ACT.cool` | engine −30 °C, costs 1 supply; locked when the engine is cool |
| First aid | U | `ACT.aid` | driver +25 health, costs 1 supply; locked when the driver is fine |
| Auto / All / Arcs fire keys | (mouse) | `ACT.fire('auto' \| 'all' \| 'show')` | auto fire, all weapons together, show firing arcs |
| Turn speed knob | (mouse) | `ACT.turn` | cycles 1× / 2× / 4× |
| End turn | Space | `ACT.end` | ends the turn (the mock just moves the clock and burns fuel) |
| Context key | E | `press('ctx')` | does `objective.ctx` (e.g. Enter Bowl opens the town screen) |
| Screen keys DRIVER / TRUCK / MAP / JOURNAL / MENU | C, I, M, J, Esc | `press('navC' …)` | open the game's screens; in this bundle J opens the Log device, I the Truck device, Esc closes |

## Live parts

#### Cab (30 live parts)

| # | part | what | where (x × y of the screen) | reads | click / hotkey changes | refresh and motion |
|---|---|---|---|---|---|---|
| 1 | screen | truckScreen - green | 1.9–15.1% × 70.9–85.8% | `health` `parts` `parts[].hp` `parts[].k` |  | every 0.5 s |
| 2 | screen | vitals - green | 15.7–23.7% × 73.6–87.4% | `health` `load` `loadMax` `power` `powerMax` `supMax` `supplies` |  | every 0.1 s |
| 3 | screen | weaponsRow - amber | 1.4–18.1% × 84.7–94.0% | `weapons` `weapons[].ammo` `weapons[].cyc` `weapons[].ic` `weapons[].mag` `weapons[].st` `weapons[].stat` |  | every 0.5 s |
| 4 | key | AUTO | 18.9–22.8% × 87.8–91.1% | `fire` `fire.auto` | `fire.auto` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 5 | key | ALL | 18.8–22.7% × 91.2–94.5% | `fire` `fire.all` | `fire.all` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 6 | key | ARCS | 18.7–22.5% × 94.7–97.9% | `fire` `fire.show` | `fire.show` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 7 | screen | contracts - lcd | 3.9–14.3% × 63.5–73.1% | `money` `objective` `objective.lines` `objective.lines[]` `objective.t` |  | every 2 s |
| 8 | gauge | KM/H | 76.3–85.7% × 72.7–89.7% | `limit` `speed` |  | face redrawn on change; needle eases to the value (rate 5/s) and quivers ±0.23° (CSS) |
| 9 | gauge | FUEL L | 86.5–92.2% × 71.9–82.0% | `fuel` |  | face redrawn on change; needle eases to the value (rate 5/s) and quivers ±0.23° (CSS) |
| 10 | gauge | ENG °C | 92.2–98.2% × 70.8–81.0% | `eng` |  | face redrawn on change; needle eases to the value (rate 5/s) and quivers ±0.23° (CSS) |
| 11 | screen | custom - amber | 86.9–92.8% × 82.5–86.7% | `heading` |  | every 0.5 s |
| 12 | screen | clock - red | 92.9–98.5% × 81.5–85.7% | `day` `time` |  | every 0.5 s |
| 13 | toggle | ROUTE [R] | 77.1–78.5% × 90.1–97.8% | `hot` `locked` `locked.route` `sw` `sw.route` | `hot.route` `sw.route` | lever eases (12/s) through 5 baked frames; lamp colour on / hot blink 2.2 Hz / locked / off |
| 14 | toggle | PATCH [P] | 80.2–81.6% × 89.7–97.4% | `hot` `locked` `locked.patch` `sw` `sw.patch` | `hot.patch` `sw.patch` | lever eases (12/s) through 5 baked frames; lamp colour on / hot blink 2.2 Hz / locked / off |
| 15 | toggle | LIGHTS [L] | 83.3–84.7% × 89.1–96.9% | `hot` `locked` `locked.lights` `sw` `sw.lights` | `hot.lights` `sw.lights` | lever eases (12/s) through 5 baked frames; lamp colour on / hot blink 2.2 Hz / locked / off |
| 16 | toggle | OVERDR [O] | 86.5–87.9% × 88.6–96.5% | `locked` `locked.overdrive` | `hot.overdrive` `sw.overdrive` | lever eases (12/s) through 5 baked frames; lamp colour on / hot blink 2.2 Hz / locked / off |
| 17 | key | COOL [G] | 90.8–93.9% × 89.7–94.5% | `locked` `locked.cool` | `eng` `locked.cool` `log[]` `supplies` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 18 | key | AID [U] | 94.1–97.2% × 89.1–94.0% | `locked` `locked.aid` | `health` `log[]` `supplies` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 19 | key | C | 28.1–30.3% × 90.2–93.8% | `pressed` | `pressed` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 20 | key | I | 30.5–32.7% × 90.2–93.8% | `pressed` | `open` `pressed` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 21 | key | M | 32.9–35.1% × 90.2–93.8% | `pressed` | `pressed` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 22 | key | J | 35.3–37.5% × 90.2–93.8% | `pressed` | `open` `pressed` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 23 | key | ESC | 37.7–39.8% × 90.2–93.8% | `pressed` | `open` `pressed` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 24 | screen | ms - lcd | 30.5–39.6% × 95.1–97.8% | `money` |  | every 2 s |
| 25 | screen | pager - lcd | 40.1–55.5% × 89.7–94.8% | `radio` |  | every 2 s |
| 26 | screen | logLine - lcd | 40.0–55.5% × 95.6–98.1% | `log` `log[]` `log[][]` |  | every 2 s |
| 27 | screen | custom - amber | 56.5–58.6% × 90.8–94.3% | `objective` `objective.ctx` `objective.ctx[]` |  | every 2 s |
| 28 | key | ENTER BOWL | 59.1–66.7% × 90.6–94.3% | `objective` `objective.ctx` `objective.ctx[]` | `open` `pressed` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 29 | knob | knob (1 / 2 / 4) | 70.0–71.8% × 91.5–94.9% | `turn` | `turn` | one baked frame per position, switched at once |
| 30 | key | END TURN [SPACE] | 57.3–66.9% × 94.7–97.9% |  | `fuel` `heading` `time` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |

#### Log menu device (9 live parts, slides up when `open === "log"`)

| # | part | what | where (x × y of the screen) | reads | click / hotkey changes | refresh and motion |
|---|---|---|---|---|---|---|
| 1 | screen | custom - paper | 20.8–58.7% × 6.9–60.2% | `day` `log` `log[]` `log[][]` `ui` `ui.filter` `ui.scroll` |  | every 1 s |
| 2 | key | ALL | 62.2–77.8% × 6.9–11.8% | `ui` | `ui` `ui.scroll` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 3 | key | TROUBLE | 62.3–77.8% × 12.9–17.8% | `ui` | `ui` `ui.scroll` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 4 | key | GOOD NEWS | 62.3–77.9% × 18.9–23.8% | `ui` | `ui` `ui.scroll` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 5 | key | CHATTER | 62.3–77.9% × 24.9–29.8% | `ui` | `ui` `ui.scroll` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 6 | key | UP | 62.5–69.9% × 36.7–42.6% |  | `ui` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 7 | key | DOWN | 70.4–77.8% × 36.7–42.6% |  | `ui` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 8 | screen | contracts - lcd | 62.2–78.1% × 44.2–54.0% | `money` `objective` `objective.lines` `objective.lines[]` `objective.t` |  | every 2 s |
| 9 | key | CLOSE [ESC] | 62.4–78.1% × 55.6–60.8% |  | `open` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |

#### Truck menu device (8 live parts, slides up when `open === "truck"`)

| # | part | what | where (x × y of the screen) | reads | click / hotkey changes | refresh and motion |
|---|---|---|---|---|---|---|
| 1 | screen | custom - green - touch zones | 14.9–33.8% × 3.3–60.2% | `parts` `parts[].hp` `parts[].k` `ui` `ui.sel` |  | every 0.5 s |
| 2 | screen | custom - paper | 35.3–60.0% × 3.4–36.0% | `parts` `parts[].hp` `parts[].k` `ui` |  | every 1 s |
| 3 | key | PATCH | 35.4–42.8% × 38.3–43.4% |  | nothing (mock) | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 4 | key | MOVE | 43.7–51.1% × 38.3–43.4% |  | nothing (mock) | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 5 | key | UNFIT | 52.0–59.5% × 38.3–43.4% |  | nothing (mock) | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |
| 6 | screen | custom - amber | 35.2–60.0% × 45.9–58.4% | `load` `loadMax` `parts` `parts[].hp` `power` `powerMax` |  | every 1 s |
| 7 | screen | custom - amber | 62.8–84.3% × 3.3–45.6% | `cargo` `fuel` `supMax` `supplies` `tank` |  | every 1 s |
| 8 | key | CLOSE [ESC] | 63.0–84.3% × 51.6–57.5% |  | `open` | top redrawn every 0.12 s (pressable) / 0.45 s; sinks 0.3 × its height when on; hot blinks 2.2 Hz |

Where = the part's box in % of the game view (left–right × top–bottom).

## Languages

Every word the HUD prints itself goes through one translator (`tr`, from `lang` and i18n.js):
- **panel labels** (control names, hotkeys, knob marks, label tape, handwriting): DOM `<span>`s on the panel plane with the panel's `matrix3d`, in the font and lit colour the 3D cab gives them, masked where hardware stands in front. A longer translation of a plain label narrows to at most 1.25 × the original width. **Dymo label tape** is live as a whole (black strip and letters), so the strip grows with the word. The paper tape stays in the picture; its handwriting is live.
- **keys**: the legend is drawn live, translated, and fitted to the key (two lines if it has a space and would get small, then narrowed).
- **screens and dials**: `T()` in screens.js translates its words and narrows a longer translation to the original's width. Lines made from words and numbers are matched by the patterns in i18n.js.
Makers' plates and round decals are logos and stay in the pictures.

| layer | text | font |
|---|---|---|
| cab | arcs = [A] | bold 23.4px Caveat, cursive |
| cab | ROUTE | bold 15px "Roboto Slab", serif |
| cab | [R] | bold 12.4px "Anonymous Pro", monospace |
| cab | PATCH | bold 15px "Roboto Slab", serif |
| cab | [P] | bold 12.4px "Anonymous Pro", monospace |
| cab | LIGHTS | bold 15px "Roboto Slab", serif |
| cab | [L] | bold 12.4px "Anonymous Pro", monospace |
| cab | OVERDR | bold 15px "Roboto Slab", serif |
| cab | [O] | bold 12.4px "Anonymous Pro", monospace |
| cab | Nose line junk | bold 24px Caveat, cursive |
| cab | 1 | bold 11px "Anonymous Pro", monospace |
| cab | 2 | bold 11px "Anonymous Pro", monospace |
| cab | 4 | bold 11px "Anonymous Pro", monospace |
| log | SHOW | 700 15px "Anonymous Pro", monospace |
| log | ROLL | 700 15px "Anonymous Pro", monospace |
| log | ribbon is nearly dry | bold 21.84px Caveat, cursive |
| truck | FITTED - TOUCH A PART | 700 15px "Anonymous Pro", monospace |
| truck | CARGO | 700 15px "Anonymous Pro", monospace |
| truck | weigh it first | bold 26px Caveat, cursive |

## Slots: the game's own interface in a screen

Any screen can be handed to the game: `hud.slot(id)` stops drawing that canvas and returns an element of the canvas's size (CSS px, below) laid on the screen with its perspective; it takes the pointer, scrolls, and holds any DOM (a React root, the game's log list…). Design its content for that pixel size. Screens of this HUD:

| id | shows now | size |
|---|---|---|
| `hud.screen1` | truckScreen - green | 475 × 280 |
| `hud.screen2` | vitals - green | 300 × 280 |
| `hud.screen3` | weaponsRow - amber | 625 × 140 |
| `hud.screen4` | contracts - lcd | 375 × 150 |
| `hud.screen5` | custom - amber | 215 × 65 |
| `hud.screen6` | clock - red | 200 × 65 |
| `hud.screen7` | ms - lcd | 350 × 55 |
| `hud.screen8` | pager - lcd | 600 × 115 |
| `hud.screen9` | logLine - lcd | 600 × 50 |
| `hud.screen10` | custom - amber | 75 × 75 |
| `log.screen1` | custom - paper | 1320 × 1056 |
| `log.screen2` | contracts - lcd | 575 × 195 |
| `truck.screen1` | custom - green - touch zones | 675 × 1175 |
| `truck.screen2` | custom - paper | 900 × 675 |
| `truck.screen3` | custom - amber | 900 × 250 |
| `truck.screen4` | custom - amber | 775 × 875 |

## Assets

| file | size | url |
|---|---|---|
| hud.webp | 88 KB | assets/hud.webp |
| hud-atlas.png | 142 KB | assets/hud-atlas.png |
| log.webp | 36 KB | assets/log.webp |
| log-atlas.png | 57 KB | assets/log-atlas.png |
| truck.webp | 43 KB | assets/truck.webp |
| truck-atlas.png | 18 KB | assets/truck-atlas.png |
| items.svg (game item icons, used by screens) | 273 KB | assets/items.svg |
| screenshot.jpg (reference: the 3D lab, not used by the code) | 560 KB | assets/screenshot.jpg |

Fonts: Google Fonts `Anonymous Pro` 400/700, `Roboto Slab` 400–900, `Caveat` 700 (see index.html).

## Changing the HUD

The 3D lab is the source: change the layout in `v4/variants.js` (or the menus in `v4/menus.js`, the screens in `v4/screens.js`), open `v4/flat/bake.html` on the bake server (`python3 tools/bake-server.py`), bake the layout, then copy a new bundle from the lab's Code panel. Screen drawing (screens.js) and state logic can change without a re-bake; anything that moves hardware needs one.
