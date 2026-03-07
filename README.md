# Macrostate

An open source economic simulation game that models [Modern Monetary Theory](https://en.wikipedia.org/wiki/Modern_Monetary_Theory) (MMT) ideas through gameplay. Players take the role of a sovereign currency-issuing government, setting fiscal policy and observing how money flows through the economy.

## Vision

A Victoria 3-inspired real-time economic simulation where the underlying economic model is built on MMT rather than mainstream economics. Players learn MMT concepts — not through lectures, but by experiencing how a sovereign currency system actually works.

## Key Features (Planned)

- **Sovereign currency mechanics** — money is created by government spending and destroyed by taxation
- **Agent-based economy** — households, firms, and banks interact to produce emergent economic behavior
- **Job Guarantee** — implement MMT's signature policy proposal as an automatic stabilizer
- **Sectoral balances** — observe how government, private, and foreign sector balances interact
- **Real resource constraints** — inflation arises from spending beyond productive capacity, not from "printing money"
- **Multiple economies** — AI-controlled nations with their own monetary systems react to your policies
- **Geographic provinces** — 2D top-down map with regions, resources, and populations
- **Scenario-based objectives** — guided challenges alongside sandbox experimentation

## Tech Stack

- **Engine:** [Godot](https://godotengine.org/) (open source, MIT licensed)
- **Language:** C#
- **Platforms:** Linux, Windows, macOS

## In-Game Console

A full-power command line console (like Paradox games) available in all builds. Query any variable, manipulate the economy, run scripts, inspect balance sheets. Operates through the SFC model to maintain accounting consistency. See [docs/technical/CONSOLE.md](docs/technical/CONSOLE.md).

## Modding

The game is designed for modding from day one. All game data lives in external JSON files — the base game itself loads through the same system mods use.

- **Data mods** — tweak economic parameters, add sectors/goods/scenarios via JSON. No code required.
- **Plugin mods** — extend game mechanics, add new agent types, custom AI, new UI panels via C# plugins.
- **Combinable** — multiple mods with dependency resolution and load ordering.

See [docs/technical/MODDING.md](docs/technical/MODDING.md) for the full modding architecture.

## Current Status

**Requirements gathering phase.** See [docs/design/GAME-DESIGN.md](docs/design/GAME-DESIGN.md) for the game design document, [docs/requirements/PRD.md](docs/requirements/PRD.md) for requirements, [docs/requirements/MVP-SCOPE.md](docs/requirements/MVP-SCOPE.md) for MVP scope, and [docs/design/ECONOMIC-MODEL.md](docs/design/ECONOMIC-MODEL.md) for the simulation model.

## License

TBD
