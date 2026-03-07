# Pull Request Guide

This document describes how to create pull requests for this repository. It is intended for both human contributors and AI agents.

## Labels

Apply **all labels that match**. Every PR should have at least one type label and one scope label where applicable.

### Type labels

| Label | Use when |
|-------|----------|
| `bug` | Fixing incorrect behavior |
| `documentation` | Changes to .md files, comments, or documentation only |
| `enhancement` | New feature or improvement to existing functionality |

### Scope labels

| Label | Use when |
|-------|----------|
| `economic-model` | Changes to the core economic model design or parameters |
| `mmt` | MMT-related correctness, accuracy, or theoretical alignment |
| `game-mechanics` | Gameplay systems, UI, player-facing features |
| `infra` | Build system, CI/CD, tooling, project configuration |

## Milestones

| Milestone | Use when |
|-----------|----------|
| `MVP` | Work required for the minimum viable product |

Assign the appropriate milestone to every PR. If no milestone fits, leave it empty and note why in the PR description.

## Branch naming

Use the pattern: `<type>/<short-description>`

| Type | Use when |
|------|----------|
| `fix/` | Bug fixes (e.g., `fix/interbank-settlement`) |
| `feat/` | New features (e.g., `feat/bond-auction`) |
| `docs/` | Documentation only (e.g., `docs/two-layer-money-flows`) |
| `refactor/` | Code restructuring without behavior change |
| `chore/` | Tooling, config, dependencies |

## PR title

- Keep under 70 characters
- Use imperative mood (e.g., "Add interbank payment flow", not "Added" or "Adds")
- Prefix with the type if it aids clarity (e.g., "docs: clarify two-layer money structure")

## PR description

Use this template:

```markdown
## Summary
<1-3 bullet points describing what changed and why>

Closes #<issue number, if applicable>

## Context
<Why this change is needed. Reference issues, audit findings, or review comments if applicable.>

## Test plan
- [ ] <Checklist of verification steps>
```

## Reviewers and assignees

- Assign the PR author as assignee
- Request review from repository maintainers

## When NOT to create a PR

- Trivial typo fixes can be committed directly to `main` if the repository has no branch protection rules
- Do not create a PR for work-in-progress unless marked as draft
