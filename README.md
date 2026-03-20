# A.F.L.I. — Afield Intelligence

Adds a dialogue option to ask NPCs for intel about the current map. NPCs report on faction presence and activity at tracked smart terrains based on live ALife simulation data. What they know — and whether they'll share it — depends on their faction, rank, and disposition.

---

## Features

- Ask any non-companion NPC for a situational report on the current level
- NPC responses are generated from live SIMBOARD data at the moment of conversation
- Intel accuracy scales with NPC rank
- NPCs have faction-based behavioral profiles that determine whether they share freely, ask for payment, or refuse entirely
- Payment unlocks full intel and persists for the duration of the conversation session
- Intel entries expire after a configurable in-game time period and are refreshed on next contact
- The smart terrain the player is currently inside is excluded from the report
- The player's own faction is referenced as "we" in the NPC's report when applicable
- All string variants are randomised to reduce repetition across conversations
- MCM support for expiration time and debug mode

---

## Requirements

- DLTX
- MCM (optional — fallback defaults are defined in `aflimod_mcm.script`)

---

## Installation

Drop the contents of the archive into your Anomaly directory and enable in Mod Organizer 2 or equivalent. Load order is not significant.

---

## Technical Overview

### Script: `aflimod.script`

Main script. Handles all logic, dialog preconditions, actions, and string building.

#### Intel Data Flow

Intel is generated once per NPC per session and cached in `intel_npcs`, a module-level table keyed by NPC alife ID.

```
actor_on_update()
  └─ is_talking() → init_npc_intel(npc)
        └─ intel_npcs[npc_id] exists? → skip
        └─ determine behavior (free / paid / refuse)
        └─ determine accuracy from rank + behavior offset
        └─ get_current_level_intel()
              └─ get_smarts_for_level(level_name)   ← reads LTX
              └─ get_current_tracked_smart()         ← excludes player location
              └─ for each smart: count_smart_squads(), faction, contested
        └─ trim intel by accuracy roll
        └─ store in intel_npcs[npc_id]
```

#### Smart Terrain Population Counting

Vanilla `SIMBOARD.smarts[id].population` excludes squads with an active script target, which means scripted story squads (rookie village guards, military outposts, etc.) are not counted. The mod counts squads directly from `SIMBOARD.smarts[id].squads`, filtering out monster communities via the global `is_squad_monster` table:

```lua
local function count_smart_squads(smart_id)
    local squads = SIMBOARD.smarts[smart_id] and SIMBOARD.smarts[smart_id].squads
    for squad_id, _ in pairs(squads) do
        local squad = alife_object(squad_id)
        if squad and not is_squad_monster[squad.player_id] then
            count = count + 1
        end
    end
end
```

#### Player Location Exclusion

Before building the intel results table, the function checks all tracked smarts for proximity to the player. If the player is within 75 units of a tracked smart, that smart is excluded from the report:

```lua
local function get_current_tracked_smart()
    local actor_pos = db.actor:position()
    for _, smart_name in ipairs(smarts_for_level) do
        local smart = SIMBOARD:get_smart_by_name(smart_name)
        if smart and actor_pos:distance_to(smart.position) < 75 then
            return smart_name
        end
    end
    return nil
end
```

This uses distance against the tracked smart list rather than `nearest_to_actor_smart` to avoid matching unrelated simulation smarts.

#### Intel Expiration

A throttled loop in `actor_on_update` runs every 1000ms engine time when the player is not in dialogue. It compares each cached entry's `generated_at` CTime against the current game time and nils expired entries:

```lua
local current_time = game.get_game_time()
for npc_id, data in pairs(intel_npcs) do
    if data.generated_at then
        local time_passed = current_time:diffSec(data.generated_at)
        if time_passed > intel_expire_time then
            intel_npcs[npc_id] = nil
        end
    end
end
```

Expiration time is in in-game seconds and is configurable via MCM (default: 3600, one in-game hour).

#### NPC Behavior Profiles

Each faction community has a weighted profile defining the probability of three behaviors:

| Behavior | Effect |
|----------|--------|
| `free` | Intel shared immediately at no cost |
| `paid` | NPC requests payment before sharing; accuracy bonus applied |
| `refuse` | NPC declines, no intel generated |

Behavior is rolled once at `init_npc_intel` time and stored in the cache entry. Paid NPCs receive a +70 accuracy offset on top of their rank-based base accuracy, making payment meaningfully better than free intel from the same NPC.

#### Intel Accuracy

Each NPC rank maps to a base accuracy value (30–100). During trimming, each smart terrain entry is kept or discarded via a `math.random(100) <= accuracy` roll. A novice NPC may only know about 30% of tracked locations; a master knows almost all of them.

#### String Building

`build_intel_string()` groups intel entries by faction, then builds natural language sentences by concatenating randomised string fragments from the XML string table.

Smarts are grouped into three buckets: faction-controlled, contested, and empty. Faction sentences are assembled first with a single opener prefix on the first sentence only. Empty and contested sentences follow without an opener. All sentence fragments are randomised independently from pools of 2–5 variants.

The `capitalize_sentences()` function handles post-assembly capitalisation, splitting on `\\n` separators and capitalising the first character of each line independently.

---

### Config: `aflimod_notable_smarts.ltx`

Located at `gamedata/configs/misc/`. Defines which smart terrains are tracked per level. Each section header is a level name as returned by `alife():level_name()`. Values are smart terrain names as registered in SIMBOARD.

Read at script load time via `ini_file()`. Parsed with `r_line()` iterating line count per section.

To add or remove tracked smarts, edit this file. No script changes are required.

---

### Config: `mod_system_aflimod_gameplay.ltx`

Registers `dialogs_aflimod.xml` into the DLTX dialog system via the `![dialogs]` patch directive.

---

### Dialog: `dialogs_aflimod.xml`

The dialog tree has the following structure:

```
[0] Actor asks for intel
 ├─ [1] precondition: free intel OR already paid → build_intel_string()
 ├─ [2] precondition: paid behavior AND not yet paid
 │    ├─ [20] precondition: can afford → actor_pay_intel() → build_intel_string()
 │    └─ [21] nevermind
 └─ [3] precondition: refuse behavior → static refusal line
```

The dialog is gated at the top level by two preconditions: `is_not_actor_companion` (from vanilla) and `is_actor_same_community`, which restricts access to NPCs of the same faction as the player, plus traders and non-monster neutrals.

---

### String Table: `st_aflimod.xml`

All user-facing strings. Randomised variants are indexed numerically (e.g. `st_aflimod_opener_1` through `_5`) and selected at runtime with `math.random(n)`. Smart terrain display names that are missing from vanilla string tables are defined in `st_aflimod_missing_smarts.xml`.

---

## Known Limitations

- Intel reflects ALife state at generation time, not at dialogue time. If a territory changes hands between when the player first talks to an NPC and when they read the intel, the data will be stale until expiration.
- `smart.faction` can be nil on smarts with no current occupants and no `default_faction` defined. These entries are silently dropped from the report.
- The `is_actor_same_community` precondition uses `character_community(npc)` which may not behave identically to `npc:character_community()` in all contexts. If the dialog fails to appear for expected NPCs, this is the first place to investigate.
- Intel is not persisted across save/load. All cached entries are lost on reload and regenerated on next contact.
