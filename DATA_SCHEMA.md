# Looking Glass: `world-data.json` schema (v1)

Live file: https://tradeshieldlabs.github.io/looking-glass/world-data.json. The game fetches it on every start.

## How the game loads it
- On start the game fetches `world-data.json?cb=<random>` with `cache: "no-store"` and a 4 s timeout.
- If the fetch works and the file passes validation, it is merged over the copy built into the game. The menu then shows `● LIVE`.
- If the fetch fails (offline, 404, timeout, bad JSON, invalid content), or if the file's `updated` is older than the built-in copy's, the game keeps the built-in copy. The menu then shows `○ BUILT-IN COPY`. The game always works.
- How the merge works:
  - Objects merge key by key. Arrays and plain values replace.
  - The top-level maps `start`, `status` and `situation` are replaced whole, so removing an entry removes it.
  - Keys starting with `_` are ignored.
- Timing matters. `start`, `start_extra`, `status` and `readiness` apply when a new game starts. Games already in progress keep their units. `ticker`, `situation`, `whats_new` and `briefs` show immediately. `events` feed the world-event deck each round.

## Top-level fields
| field | type | used for |
|---|---|---|
| `schema` | number | `1`. The game rejects a file whose schema is newer than it understands. |
| `updated` | ISO time with offset, e.g. `2026-09-27T07:00:00+09:00` | Freshness guard. **Bump it on every edit.** |
| `as_of` | `YYYY-MM-DD` (required) | Shown on the menu as "INTEL CURRENT AS OF …". |
| `sources` | string[] | General source list. |
| `note` | string | Free text. |
| `start` | `{ "<TYPE>@<loc>": "name" \| ["name", …] }` | Real name for each seeded unit slot (HF9). `TYPE` is a unit code from `src/units.js` (CSG, DDG, SSN, B21, THAAD, PATRIOT, PLANCV, YASEN, GORSH, SSK…). `loc` is a node id from `src/theater.js` (phs, cpa, nep, ars, guam…). |
| `start_extra` | `[{ t, side, loc, nat, name, hold?, why?, src? }]` (max 8) | Extra real units placed at game start, e.g. carriers tied down in the Arabian Sea. `hold` (0–6) = turns the unit starts COMMITTED; `why` is shown on its card. |
| `status` | `{ "<unit name>": { text, date?, src?, ready?, hold?, why? } }` | Real-world note shown on the unit card (📡). `ready: true` = never starts in random maintenance. `hold` = starts COMMITTED for n turns. |
| `readiness` | `{ as_of, note, types: { TYPE: { mc: 0..1, note, src } } }` | Mission-capable rates (HF10). Only the MC share of each type is ready on day one. |
| `situation` | `{ as_of, summary, flashpoints: [...], fleet: [...] }` | World situation panel (menu 🌐 and the INTEL tab). |
| `situation.flashpoints[]` | `{ id, region, title, text, date, level 1-4, src }` | One card per real flashpoint. `level` = severity bars. |
| `situation.fleet[]` | `{ name, where, status, date, src }` | Where real ships and groups are. |
| `whats_new` | `[{ date, text, src }]` | 🆕 What's new panel: list the newest first. |
| `ticker` | `[{ text, date, src }]` or strings | Real-world ticker lines. Shown during the ELITE CONSTELLATION exercise and whenever there is no war news. Use short ALL-CAPS headlines. |
| `briefs` | `[{ text, date, src }]` | Up to 4 lines appended to the day-one OPORD / EXORD as "REAL-WORLD SITUATION". |
| `events` | `[{ id, title, text, date, src, fx, w?, min_turn? }]` | World events based on real reporting. Each fires at most once per game. About 40% of rounds draw from this list while unused ones remain. |
| `events[].fx` | `{ key: number }` (each within ±30) | Allowed keys: `tension`, `oil`, `food`, `chips` (price shocks), `money_blue`, `money_red` ($B), `will_blue`, `will_red` (support/control), `pac3`, `sm6` (Blue interceptor stocks). |
| `events[].w` | number (default 1) | Draw weight. |
| `events[].min_turn` | number (default 2) | Earliest turn the event can fire. |
| `media` | `[{ url, caption, credit }]` | Optional images shown lazily in the situation panel. Use repo-relative paths (e.g. `media/foo.jpg`) or https URLs. |
| `weather` | machine-written (3c.2) | Morning weather snapshot: `{ at, src, days[7], n: { zone: [[code, gust, rain, cloud, wave, wind] ×7] }, now, storms: [{ name, lat, lon, kmh, … }] }` from Open-Meteo + GDACS. Written by `tools/wx_snapshot.mjs` (run automatically by `publish_data.py`). The game fetches live weather itself and only uses this snapshot when the live fetch fails. **Do not edit by hand.** |
| `pdb` | machine-written (3c.2) | Daily Brief voice map `{ as_of, role, clips: { hash: { f: "pdb-N.mp3?v=…", d } } }`. Written by `tools/pdb_render.py` (run automatically by `publish_data.py`), which also renders `audio/pdb-N.mp3`. **Do not edit by hand.** |
| `intros` | `{ file }` | Intro films are listed in `media/intros.json` (also editable without a rebuild). |

`src` is a list of `{ pub, date, title?, url }`. The game shows "pub, date". Keep the URL for audit.

## Rules for editors
- **Real:** assets, hull numbers, places, reported events, readiness figures. **Fiction:** leaders, the social feed, the war itself. Do not name real heads of state in in-game text. Write "U.S.–China summit", not the leaders' names.
- Every item needs a `date` and at least one `src`. Do not invent positions: if reporting is unclear, say so ("verify") or leave the seed alone.
- Unit names in `start`, `start_extra` and `status` must match the roster names in `src/oob.js` exactly, e.g. `USS Abraham Lincoln (CVN-72)`.
- Keep the file small (under 100 KB). Validate before publishing.

## Updating without a code change (daily routine)
1. Edit `/workspace/game/src/world-data.json` (the canonical copy; the next code build embeds it too). Bump `updated` and `as_of`.
2. `node tools/validate_world_data.mjs`. This checks the schema plus unit types and locations against the game tables.
3. `.venv/bin/python tools/publish_data.py "Data: <what changed>"`. This validates, copies the file to the root `world-data.json`, uploads only that file to GitHub Pages, and waits until the live file matches.
4. Commit `src/world-data.json` + `world-data.json` in `/workspace/game`.
`publish_data.py` also (best effort, never blocking) refreshes the `weather` snapshot and re-renders the 📰 Today's Brief voice (`audio/pdb-N.mp3`, uploaded in the same commit). Today's Brief is built from `situation` (summary, top 3 flashpoints by `level`, `fleet`) + the weather, so a good `situation.summary` and flashpoint `title`s make a good brief.
Players get the new data the next time the game starts. No game rebuild or version bump is needed.
