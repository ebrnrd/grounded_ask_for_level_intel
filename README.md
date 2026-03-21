# A.F.L.I. — Afield Intelligence

Adds a dialogue option to ask NPCs for intel about the current map. NPCs report on faction presence and activity at tracked smart terrains, as well as dynamic squad movements, based on live ALife simulation data. What they know — and whether they'll share it — depends on their faction, rank, disposition, and how much the player has earned their trust.

---

## Features

- Ask any non-companion NPC of the same faction for a situational report on the current level
- Intel covers faction-controlled locations, contested areas, empty smarts, and dynamic squad movements
- NPC responses are generated from live SIMBOARD data at the moment of conversation
- Intel accuracy and cap scale with NPC rank
- NPCs have faction-based behavioral profiles that determine whether they share freely, ask for payment, or refuse entirely
- Behavior is further adjusted by rank delta between player and NPC, and by faction goodwill
- NPCs can give incorrect intel — wrong faction, wrong population, wrong contested state — with higher chance at low confidence
- Three confidence tiers (low / mid / high) reflected in the language the NPC uses
- Payment unlocks full intel and persists for the duration of the conversation session
- Payment cost scales dynamically with how much intel the NPC actually has to offer
- Intel entries expire after a configurable in-game time period and are refreshed on next contact
- The smart terrain the player is currently inside is excluded from the report
- The player's own faction is referenced differently from enemy factions in the report
- Faction-specific refusal lines
- MCM support for expiration time and debug mode

---

## Requirements

- DLTX
- MCM (optional — fallback defaults are defined in `aflimod_mcm.script`)

---

## Installation

Drop the contents of the archive into your Anomaly directory or use Mod Organizer 2. Load order is not significant.

---

## Translations

Russian language support is planned but not yet implemented. The string system has been redesigned with translation in mind — all user-facing text uses complete sentence templates with `%s` slots, making it possible to restructure word order freely for other languages. That said, the mod is still in active development and new strings may be added. Anyone planning a translation should be aware things might change. You've been warned.

---

## Changelog

**v0.0.5**
Introduced a confidence system where lower-rank NPCs can give incorrect intel — wrong faction, wrong population, or wrong contested state. All phrases have been rewritten and split into three confidence tiers (low, mid, high), so the NPC's uncertainty is reflected in how they actually talk. High-rank NPCs sound like they know what they're talking about; novices hedge everything. The mistake system uses per-level plausible faction lists so wrong intel stays believable.

**v0.0.4**
Reworked how dialogue strings are composed. The system now uses full sentence templates with substitution slots, making translation significantly more straightforward. Added many phrase variants.

**v0.0.3**
Dynamic behavior based on player and NPC rank delta and faction goodwill. Higher rank difference increases the chance of free intel. Added faction-specific refusal lines.

**v0.0.2**
Added dynamic squad sightings — moving groups of stalkers and mutants not tied to tracked bases now appear in intel reports with location descriptions.

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
              └─ adjusted by rank delta + goodwill modifier
        └─ determine accuracy and cap from rank + behavior offset
        └─ get_current_level_intel(npc)
              └─ get_smarts_for_level(level_name)        ← reads LTX
              └─ get_current_tracked_smart()             ← excludes player location
              └─ for each smart: count_smart_squads(), faction, contested, confidence
              └─ apply mistake roll per entry
        └─ get_current_level_dynamic_squads_intel(npc)
              └─ iterate SIMBOARD.smarts_by_names
              └─ exclude tracked smarts and nearby smarts
              └─ one sighting per smart, apply faction mistake roll
        └─ trim both intel tables by accuracy + cap
        └─ store in intel_npcs[npc_id]
```

#### Confidence System

Each intel entry carries a `confidence` value generated from `math.random(min_conf, max_conf)` where the bounds are determined by NPC rank. When `math.random(100) > confidence`, one of three mistakes is applied: wrong population (flipped), wrong faction (replaced with a plausible level-appropriate faction), or wrong contested state (flipped). The confidence value is stored with the entry and used in `build_intel_string` to select the appropriate phrase tier.

Three tiers are defined by `CONF_LOW` and `CONF_MED` constants:
- **low** — heavy hedging, explicit uncertainty
- **mid** — sourced from someone else, mild qualifier
- **high** — direct, stated as fact

When multiple smarts are grouped into a single line (e.g. all smarts held by the same faction), the minimum confidence across the group determines the tier.

#### Smart Terrain Population Counting

Vanilla `SIMBOARD.smarts[id].population` excludes squads with an active script target, which means scripted story squads are not counted. The mod counts squads directly from `SIMBOARD.smarts[id].squads`, filtering out monster communities via the global `is_squad_monster` table:

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

Both the static and dynamic intel functions check all smarts for proximity to the player. Any smart within `NEARBY_SMART_RADIUS` units is excluded from the report. The static function uses the tracked smart list; the dynamic function uses a distance check against `SIMBOARD.smarts_by_names` directly.

#### NPC Behavior Profiles

Each faction community has a weighted profile defining the probability of three behaviors:

| Behavior | Effect |
|----------|--------|
| `free` | Intel shared immediately at no cost |
| `paid` | NPC requests payment before sharing; accuracy and cap bonus applied |
| `refuse` | NPC declines, no intel generated |

Behavior is rolled once at `init_npc_intel` time. Before rolling, the profile chances are adjusted by:
- **Rank delta**: `actor_rank - npc_rank` multiplied by 5, added to `free_chance` and subtracted from `paid_chance`
- **Goodwill modifier**: `floor(community_goodwill / 500)`, added on top

The adjusted values are renormalized to sum to 100 before rolling.

#### Dynamic Cost

Payment cost is calculated as `count(intel entries) + count(dynamic sightings)` multiplied by `PAID_INTEL_COST`. This means a well-informed NPC costs more than one with partial knowledge.

#### String Building

`build_intel_string()` groups intel entries by faction, then selects complete sentence templates from the XML string table based on confidence tier. All sentence types (faction control, contested, empty, dynamic stalker, own faction, monster) have three sets of variants indexed as `key_low_N`, `key_mid_N`, `key_high_N`. Templates use `%s` slots for faction names and location strings, enabling full restructuring for translation without script changes.

---

### Script: `aflimod_point_desc.script`

Local copy of `dynamic_news_helper.GetPointDescription` with the level name prefix removed. Used to generate location strings like "north of the Mill" for dynamic squad sightings.

---

### Config: `aflimod_notable_smarts.ltx`

Located at `gamedata/configs/misc/`. Defines which smart terrains are tracked per level for static intel. Each section header is a level name as returned by `alife():level_name()`. Values are smart terrain names as registered in SIMBOARD.

Read at script load time via `ini_file()`. To add or remove tracked smarts, edit this file. No script changes are required.

---

### Config: `mod_system_aflimod_gameplay.ltx`

Registers `dialogs_aflimod.xml` into the DLTX dialog system via the `![dialogs]` patch directive.

---

### Dialog: `dialogs_aflimod.xml`

```
[0] Actor asks for intel
 ├─ [1] precondition: free intel OR already paid → build_intel_string()
 ├─ [2] precondition: paid behavior AND not yet paid → build_pay_intel_string()
 │    ├─ [20] precondition: can afford → actor_pay_intel() → build_intel_string()
 │    └─ [21] nevermind
 └─ [3] precondition: refuse behavior → build_refuse_intel_string()
```

Gated at the top level by `is_not_actor_companion` and `is_actor_same_community`. The latter allows traders and non-monster neutrals regardless of faction.

---

### String Table: `st_aflimod.xml`

All user-facing strings. Each intel line type has five variants per confidence tier, selected at runtime with `math.random(5)`. Smart terrain display names missing from vanilla string tables are defined in `st_aflimod_missing_smarts.xml`.

---

## Known Limitations

- Intel reflects ALife state at generation time. If a territory changes hands between generation and when the player reads the intel, the data will be stale until expiration.
- `smart.faction` can be nil on smarts with no current occupants and no `default_faction` defined. These entries are silently treated as empty.
- The `is_actor_same_community` precondition uses `character_community(npc)` as a global function call. If the dialog fails to appear for expected NPCs, this is the first place to investigate.
- Intel is not persisted across save/load. Cached entries are saved and restored via `save_state` / `load_state` callbacks, but CTime serialization behavior should be verified if expiration behaves unexpectedly after a reload.
- Some levels (notably CNPP) may not behave correctly and are under review for removal from the tracked smarts list.