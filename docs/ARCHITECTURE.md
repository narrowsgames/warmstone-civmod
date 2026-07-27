# Warmstone Civ VI Mod — Technical Architecture

Internal engineering notes for continuing this mod. Not player-facing, not a
Workshop/community page — see the AI content boundary below before writing
anything else in this repo.

## AI content boundary

This project's draft AI policy restricts AI-assisted work here to software/mechanics:
game-database XML/SQL, modifier/effect wiring, `.modinfo` structure, and internal
technical docs like this one. It excludes any shipped art, any player-visible copy
(names, ability/civilopedia text, city lists, quotes), and any community-facing
documentation.

**Practical mechanism used throughout this repo:**
- Every `Name`/`Description` attribute that would normally hold a `LOC_*` key with
  matching localized English text instead holds the **literal internal Type ID**
  (e.g. `Name="FEATURE_WARMSTONE_CANOPY"`). Civ6 displays an unresolved string
  verbatim, so this shows up in-game as an obvious placeholder, not as authored copy.
- No new icon/texture files are shipped. New game elements either have no `Icon`
  set yet (game falls back to a default look) or explicitly reuse an **existing**
  base-game icon (e.g. `IMPROVEMENT_WARMSTONE_VENT_TAP` reuses
  `ICON_IMPROVEMENT_OIL_WELL`).
- When a human author is ready to add real names/flavor text and art, search this
  repo for the literal Type IDs used as placeholder `Name`/`Description` values and
  replace them with real `LOC_*` keys plus a matching `Text/en_US/*.xml` file; add
  real icon atlases the same way the base game's DLC does (see
  `Australia_Icons_Civilizations.xml` in the installed game for the pattern).

## Repo layout

```
Warmstone.modinfo
Data/Systems/          -- cross-civ mechanics: resources, terrain/features, governments
Data/Civilizations/     -- one folder per civ (Phase 2+), not yet populated
Data/FutureEra/         -- teleportation tech + reskinned victory chain (Phase 5), not yet populated
Text/en_US/             -- reserved for real localized text once a human author writes it; empty for now
docs/                   -- this file and other internal technical notes
```

## Grounding sources

- `CIVILIZATION 6 MODDING GUIDE.pdf` (Lee "LeeS" Shipp) — general modding conventions,
  `.modinfo` schema, the Modifiers system.
- The installed game itself, `E:\SteamLibrary\steamapps\common\Sid Meier's
  Civilization VI\` — used as ground truth for exact table schemas (`Base\Assets\
  Gameplay\Data\Resources.xml`, `Features.xml`, `Improvements.xml`,
  `Governments.xml`) and a real full-civ mod pattern (`DLC\Australia\Australia.modinfo`
  and its `Data/` files).
- `C:\work\beyond-warmsun-work\docs\` — the Beyond Warmsun canon this mod dramatizes.
  Treated as read-only reference; this mod does not write back to it (user decision,
  2026-07-26).

## Phase 1 systems (current state)

- `Data/Systems/Warmstone_Terrain.xml` — `FEATURE_WARMSTONE_CANOPY` (Stalwart
  closed-canopy rainforest, parallels vanilla `FEATURE_JUNGLE`) and
  `FEATURE_WARMSTONE_VENT_MARSH` (Wilder sulfide wetland, parallels vanilla
  `FEATURE_MARSH`). Base movement/defense/appeal values mirror their vanilla
  parallels; civ-specific interactions (e.g. Wilders should probably ignore the vent
  marsh's movement penalty) are deferred to the civ that makes them relevant
  (Phase 2/3), not baked in here.
- `Data/Systems/Warmstone_Resources.xml` — `RESOURCE_WARMSTONE_CANOPY_SAP` (Bonus,
  on the canopy feature, improved by the existing `IMPROVEMENT_PLANTATION`) and
  `RESOURCE_WARMSTONE_VENT_GAS` (Strategic, on the vent-marsh feature, needs its own
  `IMPROVEMENT_WARMSTONE_VENT_TAP`, modeled directly on vanilla's Oil Well).
- `Data/Systems/Warmstone_Governments.xml` — `GOVERNMENT_WARMSTONE_COMPACT` (postwar
  unity government, Democracy's civic tier) and `GOVERNMENT_WARMSTONE_SOFT_WAR`
  (ideological-rivalry government, Fascism's civic tier). Every Modifier reuses an
  existing `ModifierType`/`GovernmentBonusType` — mods can't add new
  EffectTypes/CollectionTypes/BonusTypes, only recombine existing ones.

## Total-conversion behavior: hiding vanilla leaders

Two files hide vanilla/DLC leaders from the "Choose Civilization" picker, keeping
exactly one (Rome/Trajan) available so AI opponents have something to be assigned
to while only Stalwarts exists:

- `Data/Systems/Warmstone_HideVanillaLeaders_Config.xml` — Configuration database
  (`Players`), wired to **FrontEndActions only**.
- `Data/Systems/Warmstone_HideVanillaLeaders_Gameplay.xml` — Gameplay database
  (`CivilizationLeaders`), wired to **InGameActions only**.

**Revision history — v1 and v2 both failed, for a reason neither diagnosed:**

- **v1 (reverted):** deleted only `Players`. Tested in-game, no change. Concluded
  at the time that `Players` was "display-only metadata". *That conclusion was
  wrong* — see v3.
- **v2 (reverted):** deleted `CivilizationLeaders` instead, keeping the `Players`
  delete alongside it in the same file. Tested in-game, no change: every vanilla
  leader still appeared, **and** the Stalwart leader never appeared either.
- **v3 (current).** The second half of that v2 symptom — Stalwarts *also* missing
  — was the clue that cracked it, because nothing in the hide-leaders logic should
  have affected the mod's own civ. Three independent bugs, all now fixed:

  1. **The logs being read were the wrong ones.** Civ6 on this machine writes its
     live logs to `%LOCALAPPDATA%\Firaxis Games\Sid Meier's Civilization VI\Logs\`.
     The `Documents\My Games\...\Logs\` folder also exists but is stale (last
     written 2023). Every "Database.log is clean, Passed Validation" observation in
     v1/v2 came from that stale file and meant nothing. The live log said:

     ```
     [Configuration] ERROR: no such table: CivilizationLeaders
     [Configuration]: In Query - DELETE FROM CivilizationLeaders;
     ```

  2. **The file mixed both databases.** `Players` is Configuration-only;
     `CivilizationLeaders` is Gameplay-only (confirmed against
     `Configuration/Data/Schema/AdditionalTables.sql` and
     `Gameplay/Data/Schema/01_GameplaySchema.sql`). The single v2 file was loaded
     into *both* databases, so each pass hit a table that doesn't exist in it and
     aborted the whole file — which is why the `Players` delete never committed.
     Worse, `Modding.log` showed the abort also **skipped every later frontend
     action from this mod**, so `Stalwarts_Config.xml` was never loaded at all.
     Hence the missing Stalwart leader. Fix: one file per database, each wired to
     exactly one action group.

  3. **`Domain`, and load order.** The picker does not read `Players` wholesale.
     `RulesetDomainOverrides` (in `DLC/Expansion2/Config/Expansion2_Players.xml`)
     binds the `PlayerLeader` setup parameter to a domain per ruleset —
     `RULESET_EXPANSION_2` → `Players:Expansion2_Players`. Rows with no `Domain`
     default to `Players:StandardPlayers` (per the schema) and are invisible in a
     Gathering Storm game. Every re-added row is now declared once per domain, the
     way `DLC/Australia/Data/Australia_Config.xml` does it. Relatedly,
     `Modding.log` timestamps showed the old `LoadOrder` of -10 running the delete
     at t=…016.120 while `Expansion2_Players.xml` loaded at t=…017.079 and simply
     repopulated the table afterwards.

**Load order.** DLC data loads at the default LoadOrder 0, so anything deleting
vanilla rows must run *later*, not earlier. Both hide-leaders actions now run at
100, and each Warmstone civ at 110+, so civ rows are added after the delete rather
than being wiped by it. Any future civ file must keep a LoadOrder above 100.

This only affects games created while the mod is enabled — Civ6 rebuilds the
database fresh from currently-active content, so existing saves and games made
without the mod are untouched.

**Verification standard for this feature:** a clean log is necessary but not
sufficient, and the log must be the one in `%LOCALAPPDATA%`. The real test is the
picker itself: only Trajan and the Stalwart leader present, nothing else.

## Phase 2 civ (current state): Stalwarts

`Data/Civilizations/Stalwarts/` — the proof-of-concept civ, structurally modeled
directly on the installed game's own `DLC/Australia/` files (confirmed by reading
them, not guessed):

- `Stalwarts_Civilization.xml` — civ, capital/city-name placeholders, unique ability
  (grants existing `ABILITY_IGNORE_TERRAIN_COST` + flat +3 combat strength, both
  reusing confirmed-working vanilla ModifierTypes), unique unit (Warrior replacement),
  unique improvement (buildable on the new Canopy feature and vanilla Jungle).
- `Stalwarts_Leaders.xml` — minimal leader definition.
- `Stalwarts_Config.xml` — frontend (leader-select screen) wiring, all icons reused
  from existing base-game fallbacks (`ICON_CIVILIZATION_UNKNOWN`, `ICON_LEADER_DEFAULT`).

**UNVERIFIED — first thing to check in-game:** the leader intentionally omits
`SceneLayers` (no 3D scene, per the "static 2D leader" decision). Whether the
diplomacy screen renders acceptably with no scene layers declared, or breaks, isn't
something I can check without the game running.

A lightweight self-consistency script (every `*Type=`/`TraitType`/`ModifierId`
reference in this mod's own `WARMSTONE_*` namespace resolves to something actually
declared) was run over `Data/` before committing — it caught one real bug (the
unique unit and improvement Types were referenced throughout but never registered in
the `<Types>` table) which is now fixed. It does not, and cannot, verify vanilla
constant names (`ABILITY_IGNORE_TERRAIN_COST`, `CIVIC_CRAFTSMANSHIP`, etc.) — those
were each individually confirmed by grepping the installed game's own files, not
assumed from memory.

## Known open items for the human author

- No `LOC_*` text or art exists anywhere in this mod yet — see the content boundary
  above.
- Civ-specific movement/hazard interactions with the two new Feature types are not
  yet built (belongs to Phase 2 Stalwarts / Phase 3 Wilders).
- `Resource_Harvests` (one-time Builder-harvest yield) is not defined for either new
  resource — optional, skipped for Phase 1 to keep scope minimal.
- Nobles/Sages/Lyrics have no assigned home biome (open Beyond Warmsun canon
  question) — per user decision, they'll ship with placeholder/generic terrain in
  Phase 4 rather than the mod inventing an answer.
