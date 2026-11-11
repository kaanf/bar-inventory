# Bar Inventory

Agentic software development project.

## Platforms
- Android / iOS: Kotlin Multiplatform
- Backend: Kotlin + Spring Boot
- Database: PostgreSQL

## Branch strategy
- main: Release branch
- dev: Integration branch
- feat/*: Feature branches

## Development
Product design, PRD, architecture and implementation
are managed through an agentic workflow.

## Tooling
- BMAD Method v6.12.4 (planning): installed in this repository.
  - `_bmad/`: BMAD configuration and shared scripts
  - `.claude/skills/bmad-*`: Claude Code skills (start with `bmad-help`)
  - `_bmad-output/`: planning and implementation artifacts
  - `docs/`: long-term project knowledge
- Archon (orchestration): installed locally, not in any repository.
  See [docs/orchestration/archon-multi-repo.md](docs/orchestration/archon-multi-repo.md).
