# SmartHome x Living Worlds

Animated pixel-art palette-cycling backgrounds behind the SmartHome mini dashboard,
tuned to actually survive on a **Samsung Galaxy Ace GT-S5830** (ARMv6 800 MHz,
278 MB RAM, 320x480, Firefox for Android).

The SmartHome UI is untouched. Living Worlds is only a background layer:
it reads the clock and the weather that SmartHome already has, and paints
one `<canvas>` behind the existing `.dash-box` widgets.

```
+---------------------------------------+
| Дома                          17:05   |   <- SmartHome UI, unchanged
| 25°C                                  |
|          Облачно с прояснениями       |
|                31°C                   |
|                                       |
|   [ September - Seascape - Cloudy ]   |   <- Living Worlds, z-index 0
+---------------------------------------+
```

## What this repository is

This is an **overlay**, not a fork. It contains the engine, the tooling and the
docs. It deliberately does **not** contain:

- the SmartHome sources (they are yours — the installer patches them in place);
- the 22 scenes (~6.8 MB of binary data, generated locally from the original
  Living Worlds ZIP in one command).

```
lw/living-worlds.js        the whole engine, one ES5 file, no dependencies
lw/scenes/*.lwb            generated scene data (git-ignored)
tools/convert-scenes.js    images/*.js  ->  .lwb
tools/install-mini.js      patches "index mini.html" (idempotent, reversible)
tools/verify-engine.js     proves the port is byte-identical to the original
LIVING_WORLDS.md           full technical reference
```

## Install

You need Node (any version >= 12) once, for the conversion. The runtime itself
is plain ES5 in the browser.

```bash
# 1. put the original Living Worlds project somewhere, e.g. vendor/
unzip living-worlds-master.zip -d vendor

# 2. copy this overlay into your SmartHome folder, then:
node tools/convert-scenes.js vendor/living-worlds-master lw/scenes
node tools/install-mini.js "index mini.html"
```

That is all. `server.js` already does `express.static(__dirname)`, so `lw/`
is served with no server change.

To undo everything:

```bash
node tools/install-mini.js "index mini.html" --uninstall
```

The uninstall is byte-exact — verified by `cmp` against the pristine file.

## What the installer changes

Exactly 60 added lines in **one** file, `index mini.html`, all of them wrapped
in `LW:BEGIN` / `LW:END` markers:

| Where | What |
|---|---|
| before `</head>` | `<style>` for canvas layering and the quality panel |
| first child of `#page-clock` | `<canvas id="lw-canvas">` |
| top of `updateClock()` | `LW.tick(new Date())` |
| top of `showPage(n)` | `LW.setActive(n === 0)` |
| top of `updateWeather()` | `LW.setWeather(weatherData)` |
| before `</body>` | `<script src="lw/living-worlds.js">` + `LW.init()` |

Nothing else is touched: no server code, no API, no other screen, no CSS of
the existing widgets, no swipe handling, no Tuya logic.

## No new data sources

| Living Worlds needs | Comes from |
|---|---|
| time, date, month | the `new Date()` that `updateClock()` already creates, every second |
| weather | the `weatherData` object that `fetchData()` already polls from `/api/mini/data` |

Zero new timers. Zero new network requests. Zero audio.

## Scene selection

22 scenes exist. Coverage is uneven, so the engine never invents one:

| Month | clear | cloudy | rain | snow |
|---|---|---|---|---|
| January | Winter Forest | — | — | **Winter Forest Snow** |
| February | Mountain Stream | **Mountain Stream Cloudy** | — | — |
| March | Monolith Plains | — | — | — |
| April | Deep Forest | — | **Deep Forest Rain** | — |
| May | Jungle Waterfall | **Jungle Waterfall Cloudy** | **Jungle Waterfall Rain** | — |
| June | Crystal Caves | — | — | — |
| July | Desert | **Desert Cloudy** | — | — |
| August | Aquarius | — | — | — |
| September | Seascape | **Seascape Cloudy** | — | — |
| October | Haunted Ruins (early / late) | — | **Haunted Ruins Rain** | — |
| November | Mirror Pond | — | **Mirror Pond Rain** | — |
| December | Winter Manor | — | — | — |

If the exact variant is missing, the mode degrades along a chain rather than
jumping straight to clear:

```
snow -> cloudy -> clear
rain -> cloudy -> clear
cloudy -> clear
```

So December + rain -> Winter Manor Clear, but May + snow -> Jungle Waterfall Cloudy.

## Weather mapping

SmartHome stores OpenWeatherMap's `weather[0].main`, lowercased. Note that OWM
never emits `"cloudy"` — the value is `"clouds"`, which is why a naive mapping
would silently never show a single cloudy scene. The icon code is used to tell
light cloud cover apart from overcast:

| SmartHome value | Living Worlds mode |
|---|---|
| `clear` | clear |
| `clouds` + icon `01`/`02` | clear |
| `clouds` + icon `03`/`04` | cloudy |
| `rain`, `drizzle`, `thunderstorm`, `squall` | rain |
| `snow` | snow |
| `mist`, `fog`, `haze`, `smoke`, `dust`, `sand`, `ash` | cloudy |
| anything unknown | clear |

## Quality levers

The Ace could run the original but died after ~10 minutes. Everything that
costs frames is adjustable, at runtime, and persisted in `localStorage`.

| Preset | fps | render scale | blend shift | light step | notes |
|---|---|---|---|---|---|
| `high` | 30 | 1 (640x480) | on | 15 s | desktop only |
| `medium` | 20 | 2 (320x240) | on | 30 s | |
| `low` | 15 | 2 (320x240) | off | 60 s | **default** |
| `ace` | 10 | 3 (213x160) | off | 120 s | safest |
| `off` | — | — | — | — | background disabled |

Three ways to change it:

- **long-press the clock** for one second -> on-screen panel;
- URL: `index mini.html?lw=ace`, or individually
  `?lwfps=12&lwscale=3&lwtod=120&lwblend=0&lwfit=contain&lwdebug=1`;
- console: `LW.setPreset('ace')`, `LW.set({ fps: 12, scale: 3 })`.

A watchdog also demotes the preset automatically if frames keep overrunning
their budget, so the picture degrades instead of the tab dying.

## Fidelity

`tools/verify-engine.js` loads the **original, unmodified** `oop.js`,
`palette.js` and `bitmap.js` in a `vm` context, drives them the way `main.js`
does, and compares the result against this port:

```
palette comparisons : 1694, failed 0
frame comparisons   : 36, failed 0

RESULT: the port is byte-identical to the original engine.
```

Same constants, same integer rounding, same blend-shift, same time-of-day
interpolation — water still moves, clouds still drift, castle windows still
light up at night. No GIF, no video, no static image.

## Credits

Scene artwork and the original Canvas Cycle engine: **Mark Ferrari** and
**Joseph Huckaby** (LGPL v3). The scene data is not redistributed here.

See **LIVING_WORLDS.md** for the technical details: file format, the exact
mapping between original and ported functions, the eleven original bugs found
along the way, and why the tab used to crash.
