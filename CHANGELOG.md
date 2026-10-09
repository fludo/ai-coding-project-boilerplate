# Changelog

All notable changes to this project will be documented in this file.

## [1.0.0] - 2026-10-09

Consolidated baseline. Earlier incremental history has been retired into this single v1.0 entry.

Forked from [shinpr/ai-coding-project-boilerplate](https://github.com/shinpr/ai-coding-project-boilerplate), maintained at [fludo/ai-coding-project-boilerplate](https://github.com/fludo/ai-coding-project-boilerplate).

### Added

- **Scaffolding** — `create-ai-project` sets up a TypeScript repo with Biome, Vitest, and a managed Claude Code setup (`CLAUDE.md`, commands, agents, skills); `update` refreshes it while preserving source and settings.
- **Workflow commands, agents, and skills** — the full command set (`/implement`, `/task`, `/design`, `/plan`, `/build`, `/review`, and frontend variants), the subagent roster (requirements through review and verification), and reusable skills for TypeScript, testing, specs, and orchestration.
- **CodeIgniter 4 + MariaDB backend skills** — `codeigniter-rules`, `codeigniter-testing`, and `mariadb-data-access`.
- **Workflow modes** — Normal (default) and Lite, set via `node scripts/set-workflow-mode.js lite|normal`.
- **Guides** — Quick Start, Use Cases & Commands, and Skills Editing under `docs/guides/en/`.

### Notes

- English is the only supported language.
- Requires Node.js 24.15+, pnpm, and Claude Code.
