# Polestar memory audit

Phase 1 baseline for `upstream/next`, 2026-10-15. This is measurement and
analysis only; no addon, compiler, pack-format, or guide changes are included.

## Method and limits

The test harness in `tests/helpers/lua.js` runs the real addon TOC/XML file
lists under Lua 5.1 with `tests/lua/wow_stub.lua` and
`tests/lua/stub_world.lua`. Each profile sets the character before loading
Polestar, Polestar_Guides, and the local, load-on-demand Polestar_QuestDB,
fires login, forces a full collection, and reads `collectgarbage("count")`.
Module attribution records retained Lua KB after each chunk, collecting
between chunks. Guide counts traverse `Polestar.Guides.List()` and `Get()`;
compatibility uses the addon's `Guides.GuideMatches` predicate.

QuestDB was built from the existing `data/forever` using
`node tools/polestar.js questdb build --json`; it reported 5,408 quests,
5,179 NPCs, 982 objects, 2,722 items, and 80,151 spawn points clustered to
36,746 records. The build emitted 1,401,632 Lua bytes and estimated 7,267,922
bytes of in-game memory. I did not run `data build`.

Harness numbers are Lua-state baselines, not replacements for the supplied
in-game `UpdateAddOnMemoryUsage` figures. Staged differences attribute
retained Lua data to an addon; the harness has different stubs and allocations
from the real client. Module numbers are per-TOC-chunk retained memory before
login. The chunk totals leave about 0.57 MiB attributable to ADDON_LOADED and
login work across the loaded addons, which this profile cannot assign to
individual modules. Small differences between character totals are Lua
allocator noise.

To repeat, use Lua 5.1 (set `LUA` if it is not on PATH), build QuestDB as
above, and run the diagnostic scratch drivers in this task's
`C:\Users\thmas\Documents\Codex\2026-10-15\you-re-working-in-a-clone\work`
folder. `memory-profile.cjs` and `module-profile.cjs` use the clone's existing
harness and TOC expansion helpers (the clone path is set in each driver).
The first reports addon-stage totals and guide/goal structure counts; the
second reports retained memory by TOC chunk. These scratch drivers are not
addon or test code and are not in the repository.

## Baseline

In-game measurements supplied for this audit:

| AddOn | In-game reading |
| --- | ---: |
| Polestar_Guides | 32.0 MiB |
| Polestar | 29.2 MiB |
| Polestar_QuestDB | 8.6 MiB |
| **Total after all three are loaded** | **69.8 MiB** |

Lua harness memory after full collection:

| Loaded through login | Alliance Dwarf | Horde Orc | Increment attributed |
| --- | ---: | ---: | ---: |
| Polestar | 4.34 MiB | 4.34 MiB | 4.34 MiB |
| plus Polestar_Guides | 27.97 MiB | 27.95 MiB | 23.75 MiB |
| plus Polestar_QuestDB | 34.19 MiB | 34.17 MiB | 6.22 MiB |

Harness values are MiB (1024 KiB). The addon-specific increments are more
useful than comparing the harness aggregate to the in-game readings. The
guide increment is close in scale to its 5.5 MB generated Lua source; loading
that source expands it into many live Lua tables.

## Guides data shape

Both characters registered the same complete set: **220 guides, 12,017
goals, and 72,285 reachable tables** in guide definitions and nested data.
Only 106 guides match Alliance Dwarf and 104 match Horde Orc. The registry
still includes all 220 so the picker can list other factions, races, and
classes.

| Field / structure | Occurrences |
| --- | ---: |
| `x`, `y` values | 46,702 each |
| `map` | 24,346 |
| `r` | 23,193 |
| rows in `spots` arrays (in 1,599 goal fields) | 22,356 |
| `npcName` | 19,972 |
| rows in `candidates` arrays (in 429 goal fields) | 14,461 |
| `faction` | 14,684 |
| `alt` | 12,635 |
| goals with `t` | 12,017 |
| `title` | 11,814 |
| `note` | 7,769 |
| `itemName` | 2,647 |
| `preTitles` string pairs | 2,234 |

Most frequent goal verbs are `accept` (2,775), `turnin` (2,741), `collect`
(1,608), `goto` (960), `explore` (825), and `kill` (693). Spots and
candidates are especially table-heavy: their 36,817 rows alone account for
just over half of the guide graph's tables. The 22,356 spot rows have `x`,
`y`, and often `r`/`alt` fields; candidate rows carry coordinates and
optional restrictions.

The guide graph has 14,028 distinct string values and 433,416 references to
already-seen string values. This is a reference count, not 433,416 separately
allocated strings: Lua 5.1 interns strings. For the common fields measured,
unique string payload is about 559 KiB total (`note` 427 KiB, `title` 78 KiB,
`itemName` 25 KiB, `npcName` 29 KiB; per-field byte counts can overlap when a
value occurs in multiple fields). Distinct guide text matters, but the large
repeat count does not mean a manual string-intern table can reclaim the same
payload.

Exact per-guide heap size was not isolated without unloading definitions from
the registry. A repeatable size proxy is each guide's goals, spots, candidates,
and goal-text bytes. The largest by goal count are:

| Guide | Goals | Spot rows | Candidate rows |
| --- | ---: | ---: | ---: |
| Horde Skyborne: Zephras Isle | 338 | 340 | 315 |
| Mining | 321 | 0 | 133 |
| Alliance Skyborne: Zephras Isle | 310 | 340 | 402 |
| Herbalism | 307 | 0 | 48 |
| Eastern Kingdoms 1–30 exploration | 306 | 0 | 0 |
| Skinning | 285 | 0 | 111 |
| Alliance Dwarf/Gnome: Dun Morogh | 269 | 260 | 409 |
| Alliance Human: Elwynn Forest | 262 | 590 | 476 |

Across 220 guides, the mean is 54.6 goals and about 110 KiB of retained data
per guide (23.75 MiB divided by 220). Sizes vary substantially; the table is
a content-volume ranking, not an exact per-guide heap ranking.

## Polestar module allocations

Retained Lua memory attributed to each Polestar TOC module's chunks before
login (both characters are the same to rounding):

| Module | KiB |
| --- | ---: |
| UI | 1,162 |
| Core | 580 |
| Nav, excluding generated travel data | 351 |
| Gear | 247 |
| QoL | 247 |
| Theme | 252 |
| Rares | 175 |
| Lookup | 168 |
| Maps | 167 |
| Quests | 166 |
| `Nav/TravelData.lua` | 137 |
| Dungeons | 95 |

The chunks account for about 3.66 MiB before login; the full Polestar
post-login harness total is about 4.34 MiB. UI is the largest own-module
group. TravelData is about 0.13 MiB and Dungeons about 0.09 MiB in this
harness. Laziness in those data modules alone cannot approach the guide
pack's footprint.

## QuestDB and repeated data

Loading the local QuestDB TOC adds about 6.22 MiB in the harness; the game
measurement is 8.6 MiB. The generated addon contains compact flat quest,
NPC, object, and item records. Its spawn data is substantial (80,151 source
points reduced to 36,746 clusters), so coordinates and per-record tables are
likely its leading structures.

Quest titles, NPC names, and guide goal text can repeat across the two packs.
Identical Lua strings are interned in the shared VM, so repeated values refer
to the same string object. Numeric coordinates are inline values, not
separately allocated strings; avoiding duplicated numbers would require
changing record shapes or referencing QuestDB records by id. The harness did
not assign in-game AddOn memory ownership for cross-addon string references.

## Candidate fixes, ranked

Estimates are rough savings against the 69.8 MiB in-game total; they need
validation in game. Ranges overlap and must not be added together.

| Rank | Candidate | Estimated saving | Risk / notes |
| ---: | --- | ---: | --- |
| 1 | Keep compact headers for every guide, but load a guide body only when opened. | **18–24 MiB** while most guide bodies are unopened. | High. Picker browsing, search, and details use guide metadata and step facts. Needs a metadata index and on-demand behavior that preserves menu access and community packs. |
| 2 | Keep all headers but initially register only character-compatible guide bodies; load other bodies when explicitly opened. | **10–13 MiB** for these profiles, using the 114/116 non-matching guides as a rough proxy. | High. Faction filters intentionally expose other guides, and a player may inspect guides for another character. |
| 3 | Compact goal, spot, and candidate records with positional fields/shared schemas; start with 36,817 spot/candidate rows and 12,017 goal tables. | **2–5 MiB**. | Medium-high. Compiler, loader, pack-format docs/version/refusal logic must change together, with a legacy path for third-party format-2 packs. The current hash shape is consumed throughout engine and UI code. |
| 4 | Defer selected Polestar UI/module tables or frame construction until the feature opens. | **0.2–0.8 MiB** in the addon Lua harness; game result uncertain. | Medium. UI is the largest own-module group at 1.13 MiB. Check login listeners, menus, and options registration. This is a smaller return than Guides. |
| 5 | Replace repeated strings with an explicit intern table. | **Near 0 MiB** expected. | Low value. Lua already interns strings; repeated names are references, not duplicate payload. Avoid unless another profiler identifies a source of duplicate string storage. |
| 6 | Deduplicate guide/QuestDB titles, NPC names, and coordinates across addons. | **0–1 MiB** expected from this evidence. | High. Strings already share storage; coordinates are numbers. Cross-pack references would tie standalone community packs to QuestDB and complicate compatibility. |

The best next experiment is a lazy-guide-body prototype that keeps every
menu row and detail header available. Compact records have the next-largest
plausible savings, but require a format migration. No candidate is implemented
in this phase.
