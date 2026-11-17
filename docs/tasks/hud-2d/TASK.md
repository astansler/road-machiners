# Driving HUD: the "Twin pods" cab, 2D baked

Replace the driving HUD with the Twin pods cab from the ROAM HUD Lab. It is a 3D cab rendered once from a fixed camera and shown in 2D: one still picture per device, live parts on top in DOM, canvas 2D and CSS transforms. There is no WebGL for the HUD and no per-frame render. A working reference is in `reference/` here.

- Lab (switch "2D baked" on, EN / RU beside it): https://roam-game-ui.astansler.dev/v4/#v2-cruise
- Target look: `screenshots/2d-en-2560x1440.jpg` and `screenshots/2d-ru-2560x1440.jpg` (the reference bundle as shipped), `screenshots/3d-lab-en-2560x1440.jpg` (the 3D original it was baked from).

## Read first

The reference's JavaScript is stored as `*.js.txt`, and its middle dots as `\u00b7` escapes, so it stays outside the quality gate. The baked art is in `reference/assets/`. To run it, copy it into `tmp/`, which the gate and git ignore, and drop the `.txt`:

```bash
mkdir -p tmp/hud-2d && cp -R docs/tasks/hud-2d/reference/. tmp/hud-2d/ && for f in tmp/hud-2d/*.js.txt; do mv "$f" "${f%.txt}"; done
```

1. `tmp/hud-2d/README.md`: how the 2D cab works, its API (`mountHud`), the state contract, every live part, the controls, languages and slots.
2. `tmp/hud-2d/index.html`: serve `tmp/hud-2d/` over any static server and open it (`?lang=ru` for Russian) to see the reference running with a sample state.
3. `tmp/hud-2d/roam-hud.js`, `flat.js`, `layout.js`, `bake.js`: the runtime, this cab's live parts, and where each part sits (2560 x 1440 design px, CSS `matrix3d` per face).

## What to build

1. Port the runtime into the game as TypeScript, e.g. `game/src/ui/cab/`. Keep its structure: a design space of 2560 x 1440 scaled to the game view as one piece, one layer per device, and the order picture → colour masks → frames → live text → lever bodies → faces → needles → hit shapes. The baked files (`tmp/hud-2d/assets/*.webp`, `*-atlas.png`, and the layout data in `tmp/hud-2d/bake.js`) are data: move them as they are, e.g. to `game/public/hud/twin-pods/`. Do not redraw or re-crop them.
2. Write an adapter from `World` to the state contract in `tmp/hud-2d/README.md` ("State contract"). Today's HUD (`game/src/ui/hud.ts`, `hud-readout.ts`) already reads most of these values; use the same sources. Fields the game does not have yet stay at a sane default and are listed in the PR.
3. Wire every control to the game's own commands (table "Controls" in `tmp/hud-2d/README.md`): route / manual, auto patch, lights, overdrive, cool, first aid, fire auto / all / arcs, turn speed, end turn, the context action, and the screen keys (driver, truck, map, journal, menu). Hotkeys stay as they are in the game.
4. When the cab is on, it replaces the driving HUD panels it covers (speed, fuel, heat, hull, driver, weapons, switches, radio, contacts, log line, contracts, M's, clock). Screens that are not part of the cab (journal, garage, shop, map, town) stay as they are and open from the cab's keys.
5. Text: every word the cab prints comes from the game's language, English and Russian. `tmp/hud-2d/i18n.js` has the Russian words for the cab's own labels and keys; move them into the game's translation tables. Text from the world (log, radio, names, contracts) is already translated by the game.
6. Clicks: only the cab's controls take the pointer. A click on open view or on the cab's still hardware must not drive the truck (`covers()` in the reference).

## Rules

- **This cab keeps its own type**: Anonymous Pro, Roboto Slab and Caveat, as in the reference. This is a deliberate exception to the fonts of `game/docs/ui.md` for the cab only; note it there. All three cover Latin and Cyrillic. Load them like the game's other fonts.
- Never put a filter, tint or overlay on the game picture.
- No WebGL, canvas 3D or three.js for the HUD. The cab costs no draw calls; a canvas is redrawn only when its timer is due or the state changed.
- The currency is M / M's (М in Russian), never money.
- The game's text rules apply to the port: the reference's middle-dot separators (U+00B7, as between "MOUNT 2" and "EMPTY") become a comma or a colon, and the port carries no source comments.
- Do not restyle, move or redraw the cab's parts. A change to the cab is made in the HUD Lab and baked again there.
- Phones and small screens: the cab scales with the view width. Below the width where its smallest text gets unreadable, keep today's HUD.

## Done when

- At 2560 x 1440 and 1280 x 720 the cab matches `screenshots/2d-en-2560x1440.jpg` (and `2d-ru` in Russian) part for part, with the game's real values.
- Every control works with the mouse and its hotkey and changes the game, not a mock.
- Needles ease to their values, levers flip, keys sink when on, warning keys blink, the radar and vitals screens move.
- Switching the language switches every label, key and screen word of the cab.
- The game's tests, `npm run layout` and the quality gate pass.
