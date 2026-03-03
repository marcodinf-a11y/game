# Modding Architecture

The game supports modding from day one. The architecture is designed in layers so that casual modders can tweak data files while advanced modders can extend the game with new mechanics.

## Design Principles

1. **Data-driven by default** — all game data lives in external JSON files, never hardcoded. The base game is itself structured like a mod.
2. **Base game as a mod** — the vanilla game data is loaded through the same system mods use. There is no special path for "built-in" content.
3. **Layered override system** — mods override or extend base data. Multiple mods can be active with a dependency-aware load order.
4. **Two tiers of modding** — data overrides for casual modders, plugin API for advanced modders.

## Modding Tiers

### Tier 1: Data Overrides (Casual)

Modders create JSON files that override or extend base game data. No code required.

**What can be modded via data:**
- Economic parameters (tax rates, markup rates, wage floors, etc.)
- Sector definitions (new sectors, modified production chains)
- Goods and resources (new goods types, modified properties)
- Household class definitions (new classes, modified needs hierarchies)
- Scenario definitions (new scenarios with custom objectives and fail conditions)
- Agent behavior parameters (consumption propensities, investment thresholds, etc.)
- Map data (provinces, resources, terrain)
- UI text and localization strings
- Bond properties, interest rate rules
- Time lag durations

**How it works:**
1. Base game data lives in `data/base/`
2. Mods live in `mods/<mod-name>/`
3. A mod can include any data file that exists in `data/base/`
4. The mod's version of a file overrides or extends the base version
5. Load order determines which mod wins when multiple mods change the same data

### Tier 2: Plugin API (Advanced)

Modders write code (C# or GDScript) that hooks into the game's event system and can extend or replace game mechanics.

**What can be done via plugins:**
- New agent types (e.g., trade unions, NGOs, international organizations)
- New economic mechanics (e.g., stock market, derivatives, cryptocurrency)
- New policy levers for the player
- Custom AI behavior for nations
- New UI panels and visualizations
- Custom scenario logic and events
- Modified simulation tick phases
- New map overlays and interactions

**How it works:**
1. Mods include a plugin entry point (a C# class implementing a mod interface)
2. The game discovers and loads plugin assemblies from mod folders
3. Plugins register event handlers and extensions through a mod API
4. The API provides hooks at key points in the simulation loop

## Mod Structure

```
mods/
└── my-mod/
    ├── mod.json              # Mod manifest (required)
    ├── data/                 # Data overrides (Tier 1)
    │   ├── sectors.json      # Override/extend sector definitions
    │   ├── goods.json        # Override/extend goods
    │   ├── scenarios/        # New scenarios
    │   │   └── my-scenario.json
    │   └── ...
    ├── scripts/              # Plugin scripts (Tier 2)
    │   └── MyModPlugin.cs
    └── assets/               # Custom assets (textures, sounds, etc.)
        └── ...
```

### Mod Manifest (`mod.json`)

```json
{
  "id": "com.author.my-mod",
  "name": "My Economic Mod",
  "version": "1.0.0",
  "description": "Adds stock market mechanics",
  "authors": ["Author Name"],
  "gameVersionMin": "0.1.0",
  "gameVersionMax": "1.x",
  "dependencies": [
    {
      "id": "com.author.other-mod",
      "versionMin": "2.0.0"
    }
  ],
  "conflicts": [
    {
      "id": "com.someone.incompatible-mod",
      "reason": "Both modify the banking system in incompatible ways"
    }
  ],
  "loadOrder": {
    "loadAfter": ["com.author.other-mod"],
    "loadBefore": ["com.someone.later-mod"]
  },
  "tier": "data"
}
```

**Fields:**
- `id` — unique identifier (reverse domain notation)
- `name` — human-readable name
- `version` — semantic versioning
- `description` — what the mod does
- `authors` — list of authors
- `gameVersionMin` / `gameVersionMax` — compatible game versions
- `dependencies` — mods that must be present and loaded before this one
- `conflicts` — mods that are incompatible with this one
- `loadOrder` — explicit ordering hints
- `tier` — "data" for Tier 1 mods, "plugin" for Tier 2 mods

## Base Game Data Structure

The base game is structured as data files that the mod system can override:

```
data/
└── base/
    ├── economy/
    │   ├── sectors.json          # Sector definitions (agriculture, industry, services)
    │   ├── goods.json            # Goods and resource definitions
    │   ├── production.json       # Production chains (inputs → outputs per sector)
    │   └── parameters.json       # Global economic parameters
    ├── agents/
    │   ├── households.json       # Household class definitions and behavior parameters
    │   ├── firms.json            # Firm behavior parameters
    │   ├── banks.json            # Banking parameters (spreads, creditworthiness thresholds)
    │   └── government.json       # Government defaults and constraints
    ├── scenarios/
    │   ├── sandbox.json          # Sandbox mode configuration
    │   └── full-employment.json  # Example scenario
    ├── map/
    │   └── default-map.json      # Province and resource layout
    └── config/
        ├── simulation.json       # Tick rate, lag durations, speed settings
        └── ui.json               # UI layout and chart configurations
```

### Example: `sectors.json`

```json
{
  "sectors": [
    {
      "id": "agriculture",
      "name": "Agriculture",
      "description": "Produces food and raw materials",
      "inputs": [
        { "resource": "labor", "weight": 0.7 },
        { "resource": "land", "weight": 0.3 }
      ],
      "outputs": [
        { "good": "food", "category": "survival" },
        { "good": "raw_materials", "category": "intermediate" }
      ],
      "baseMarkup": 0.15,
      "baseProductivity": 1.0,
      "capitalDepreciationRate": 0.02
    },
    {
      "id": "industry",
      "name": "Industry",
      "description": "Produces manufactured goods and capital goods",
      "inputs": [
        { "resource": "labor", "weight": 0.4 },
        { "resource": "raw_materials", "weight": 0.3 },
        { "resource": "capital_goods", "weight": 0.3 }
      ],
      "outputs": [
        { "good": "manufactured_goods", "category": "comfort" },
        { "good": "capital_goods", "category": "investment" }
      ],
      "baseMarkup": 0.20,
      "baseProductivity": 1.0,
      "capitalDepreciationRate": 0.03
    }
  ]
}
```

### Example: `households.json`

```json
{
  "classes": [
    {
      "id": "low_income",
      "name": "Low Income",
      "populationShare": 0.40,
      "needs": [
        { "good": "food", "priority": 1, "monthlyUnits": 10, "elasticity": 0.1 },
        { "good": "housing", "priority": 2, "monthlyUnits": 1, "elasticity": 0.2 },
        { "good": "manufactured_goods", "priority": 3, "monthlyUnits": 2, "elasticity": 0.7 },
        { "good": "services", "priority": 4, "monthlyUnits": 1, "elasticity": 0.9 }
      ],
      "reservationWageMultiplier": 0.7,
      "maxDebtToIncomeRatio": 0.5,
      "savingsTargetMonths": 1
    }
  ]
}
```

## Data Override Mechanics

### Merge Strategies

When a mod provides a data file, it is merged with the base (or previous mod's) data:

- **Replace**: the mod's value completely replaces the base value
- **Extend**: the mod adds new entries to a list (e.g., new sectors, new goods)
- **Patch**: the mod modifies specific fields of existing entries

Each data file specifies its merge strategy:

```json
{
  "_modAction": "patch",
  "sectors": [
    {
      "id": "agriculture",
      "baseMarkup": 0.25
    }
  ]
}
```

- `"replace"` — this file replaces the entire base file
- `"extend"` — new entries are appended, existing entries untouched
- `"patch"` — matched by `id`, only specified fields are overwritten

## Mod Load Order and Dependencies

### Resolution Algorithm

1. Read all `mod.json` manifests from `mods/` directory
2. Check game version compatibility — disable incompatible mods with warning
3. Check for declared conflicts — report errors if conflicting mods are both active
4. Build dependency graph from `dependencies` and `loadOrder` fields
5. Topological sort to determine load order (dependencies load first)
6. Detect circular dependencies — report error
7. Load mods in resolved order, applying data overrides sequentially

### Conflict Handling

- **Missing dependency**: mod is disabled with a clear error message
- **Version mismatch**: mod is disabled with version requirements shown
- **Declared conflict**: both mods cannot be active; user must choose
- **Undeclared conflict** (same field modified by two mods): last in load order wins (user can reorder)

## Plugin API (Post-MVP)

The plugin API will provide these hook points:

### Simulation Hooks
- `OnTickStart(TickContext)` — before any tick processing
- `OnGovernmentPhase(GovernmentContext)` — during government spending/taxation
- `OnProductionPhase(ProductionContext)` — during production decisions
- `OnMarketPhase(MarketContext)` — during buying/selling
- `OnFinancialPhase(FinancialContext)` — during bank lending/repayment
- `OnTickEnd(TickContext)` — after all processing, before UI update

### Game Event Hooks
- `OnScenarioStart(ScenarioContext)`
- `OnScenarioEnd(ScenarioResult)`
- `OnPolicyChange(PolicyChangeEvent)`
- `OnAgentCreated(AgentContext)`
- `OnAgentRemoved(AgentContext)`

### Extension Points
- `RegisterSector(SectorDefinition)` — add new economic sectors
- `RegisterGood(GoodDefinition)` — add new goods/resources
- `RegisterAgentType(AgentTypeDefinition)` — add new agent types
- `RegisterPolicy(PolicyDefinition)` — add new player policy levers
- `RegisterScenario(ScenarioDefinition)` — add new scenarios
- `RegisterUIPanel(UIPanelDefinition)` — add new UI panels

## MVP Scope

For the MVP, the modding infrastructure is limited to the **data-driven foundation**:

**In MVP:**
- All game data loaded from JSON files in `data/base/`
- No hardcoded economic parameters, sector definitions, or scenario data
- Clean separation between data and logic in the codebase
- Data schema validated on load
- Base game structured as if it were a mod (same file format, same loading path)

**Not in MVP:**
- Mod loader (modders can edit `data/base/` directly to test)
- Mod manifest system
- Override/merge mechanics
- Dependency resolution
- Plugin API and event hooks
- Mod management UI

The key MVP deliverable is the **architecture**: the game must be designed so that adding the mod loader later is a matter of adding a new data source, not restructuring the entire codebase.
