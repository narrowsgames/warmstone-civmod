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
