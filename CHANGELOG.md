# Changelog

All notable changes to this project will be documented in this file.

## [1.0.0] - 2026-10-09

Consolidated baseline. This entry synthesizes the current state of the starter kit; earlier incremental history has been retired.

### Added

- **Project scaffolding** — `create-ai-project` sets up a TypeScript repository preconfigured with Biome formatting and linting and Vitest, plus a project-level `CLAUDE.md`, commands, agents, and skills so Claude Code follows the repository's rules. `npx create-ai-project update` refreshes the managed Claude Code setup while preserving source code, package settings, and the saved workflow mode.
- **Workflow commands** — `/implement`, `/task`, `/design`, `/plan`, `/build`, `/review`, `/diagnose`, `/reverse-engineer`, `/add-integration-tests`, `/project-inject`, `/create-skill`, `/refine-skill`, and `/update-doc`, with dedicated frontend variants `/front-design`, `/front-plan`, `/front-build`, and `/front-review`.
- **Specialized agents** — A full set of subagents covering requirements, design, planning, execution, review, and verification, including requirement-analyzer, prd-creator, technical-designer (+ frontend), work-planner, task-decomposer, task-executor (+ frontend), quality-fixer (+ frontend), code-reviewer, code-verifier, document-reviewer, security-reviewer, integration-test-reviewer, acceptance-test-generator, design-sync, codebase-analyzer, ui-analyzer, ui-spec-designer, scope-discoverer, investigator, solver, verifier, skill-creator, and skill-reviewer.
- **Skills** — Reusable guidance for TypeScript rules and testing, frontend TypeScript rules/testing, frontend technical spec, coding standards, technical spec, implementation approach, integration/E2E testing, documentation criteria, requirement convergence, project context, LLM-friendly context, subagents orchestration, and skill optimization.
- **CodeIgniter 4 + MariaDB backend skills** — `codeigniter-rules`, `codeigniter-testing`, and `mariadb-data-access` for PHP/MariaDB backend work.
- **Workflow modes** — Normal Mode (default) runs independent design and security checks and per-task quality runs; Lite Mode reduces independent checks and agent calls, consolidating quality into one final run per layer before code review, while keeping focused checks, required test reviews, and approval gates. Set the project default with `node scripts/set-workflow-mode.js lite|normal`; explicit session selections take precedence and updates retain the saved mode.
- **Guides** — Quick Start, Use Cases & Commands, and Skills Editing guides under `docs/guides/en/`.

### Notes

- English is the only supported language.
- Requires Node.js 24.15+, pnpm, and Claude Code.
