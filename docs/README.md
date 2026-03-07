# Documentation

An open source economic simulation game built on Modern Monetary Theory (MMT).

## Reading Order

If you're new to the project, read in this order:

1. **[design/GAME-DESIGN.md](design/GAME-DESIGN.md)** — what the game is and why it exists
2. **[design/ECONOMIC-MODEL.md](design/ECONOMIC-MODEL.md)** — the MMT-based simulation model
3. **[requirements/PRD.md](requirements/PRD.md)** — detailed functional and non-functional requirements
4. **[technical/ARCHITECTURE.md](technical/ARCHITECTURE.md)** — system design and project structure
5. **[technical/MVP-IMPLEMENTATION-PLAN.md](technical/MVP-IMPLEMENTATION-PLAN.md)** — phased build plan with TDD

## Document Index

| Document | Directory | Role | Audience |
|---|---|---|---|
| [GAME-DESIGN.md](design/GAME-DESIGN.md) | design/ | Game vision, mechanics, and design rationale | Everyone |
| [ECONOMIC-MODEL.md](design/ECONOMIC-MODEL.md) | design/ | MMT economic model specification | Designers, economists |
| [PRD.md](requirements/PRD.md) | requirements/ | Functional and non-functional requirements | Developers, testers |
| [MVP-SCOPE.md](requirements/MVP-SCOPE.md) | requirements/ | MVP scoping: definition of done, simplifications, deferrals | Developers, testers |
| [ARCHITECTURE.md](technical/ARCHITECTURE.md) | technical/ | System architecture, project structure, design patterns | Developers |
| [MVP-IMPLEMENTATION-PLAN.md](technical/MVP-IMPLEMENTATION-PLAN.md) | technical/ | Phased implementation plan with TDD methodology | Developers |
| [CONSOLE.md](technical/CONSOLE.md) | technical/ | In-game console commands and scripting | Developers, players |
| [MODDING.md](technical/MODDING.md) | technical/ | Modding architecture and data format | Modders, developers |
| [UI-TESTING-ROADMAP.md](technical/UI-TESTING-ROADMAP.md) | technical/ | Path toward automated UI testing | Developers |

## Decisions

The `decisions/` directory contains [Architecture Decision Records](decisions/README.md) (ADRs) — lightweight documents capturing key design decisions and their rationale.

## Reviews

The `reviews/` directory contains audit artifacts — point-in-time assessments of document quality and accuracy. These are not reference documentation.

| Review | Purpose |
|---|---|
| [prd-review.md](reviews/prd-review.md) | PRD correctness, staleness, completeness review |
