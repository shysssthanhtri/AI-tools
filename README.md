# AI IDE Reusable Configurations

A curated collection of reusable configurations, prompts, and skills for AI-assisted IDE workflows (GitHub Copilot, Cursor, and similar tools).

This repository is intended as a practical reference and starter kit for teams and individuals who want consistent AI coding workflows across projects.

## Purpose

- Centralize reusable AI IDE configuration assets.
- Make proven prompts/skills easy to discover and reuse.
- Keep a shared, versioned source of truth for AI workflow setup.

## Repository Structure

- `.github/skills/`: Skill definitions used by GitHub Copilot Chat customizations.
- `skills/`: Mirrored or working skill directories for reuse and maintenance.

Current skill packs include topics such as:

- Architecture blueprint generation
- Feature and test breakdown
- Conventional commits
- README/specification/implementation-plan generation
- Documentation writing
- Excalidraw diagram generation
- TypeScript MCP server scaffolding

## How To Reuse

1. Pick a skill or configuration folder from this repo.
2. Copy it into the target repository (commonly under `.github/skills/`).
3. Adjust wording, constraints, and examples to match your project.
4. Validate behavior in your IDE assistant before broad rollout.

> [!TIP]
> Keep your local customizations small and composable. It is usually better to combine a few focused skills than to maintain one very large, generic prompt.

## Source and Attribution

Many configurations in this repository are adapted from well-known community and official resources, including:

- https://github.com/github/awesome-copilot

Additional sources may be included over time where relevant.

> [!IMPORTANT]
> This repository is a curated collection for reuse and learning. Please review the license and attribution requirements of any upstream source before redistributing or publishing derivative content.

## Notes

- This project is not an official distribution of upstream repositories.
- Content may be modified to fit practical usage in real projects.
- Prefer linking back to original sources when sharing externally.
