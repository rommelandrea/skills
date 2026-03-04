This file provides guidance to AI coding agents like Claude Code (claude.ai/code), Cursor AI, Codex, Gemini CLI, GitHub Copilot, and other AI coding assistants when working with code in this repository.

## What this repository is

- This is a **skills/prompt library** for AI-assisted development (not a typical application service).
- Primary content lives under `skills/` as Markdown skill definitions and rule documents.
- `src/index.ts` is minimal and only exports `version`; it is not where product logic lives.

## High-level architecture

### 1) Skill packages (`skills/<skill-name>/`)

Each skill directory is a self-contained package of guidance:

- `SKILL.md`: entrypoint for the skill (metadata, when-to-use, instructions, links)
- `rules/*.md`: detailed rule documents referenced from `SKILL.md`

Current top-level skills include:

- `biome-linting` — Linting and formatting workflows with Biome (install, config, migration from ESLint/Prettier, CI/editor integration)
- `fastify` — Comprehensive Fastify best practices (plugins, routes, schemas, hooks, auth, testing, deployment, and more)

### 2) Minimal TypeScript package surface (`src/`)

- `src/index.ts` exports package version only.
- TypeScript config (`tsconfig.json`) is strict and `noEmit`; this repo uses TS checks, not a compile output pipeline.

### 3) Documentation-driven behavior

- Most “logic” is instruction text in Markdown.
- Internal cross-references in `SKILL.md` files (for example `rules/...`) are part of the public skill structure and should stay valid.

## Repository structure

```
skills/
  biome-linting/
    SKILL.md                              # Skill entrypoint
    rules/
      biome-setup.md                      # Install, initialize, CLI usage
      biome-config.md                     # biome.json configuration
      migration-from-eslint-prettier.md   # Migration guide
      ci-and-editor-integration.md        # CI, pre-commit, editor setup
  fastify/
    SKILL.md                              # Skill entrypoint
    rules/
      plugins.md, routes.md, schemas.md,  # 19 rule files covering
      hooks.md, testing.md, ...           # all Fastify patterns
src/
  index.ts                                # Package version export only
```

## Common commands

Run from repository root.

- Install deps: `npm install`
- Typecheck: `npm run typecheck`
- Lint: `npm run lint`
- Run tests: `npm test` (alias for `node --test`)
- Run a single test file: `node --test path/to/file.test.ts`
- Run tests matching a name: `node --test --test-name-pattern “pattern”`

## Editing rules for this repo

1. **Treat `skills/*/SKILL.md` as an index contract.**
   - Every `rules/*.md` file must be explicitly mentioned and linked from that skill’s main `SKILL.md`.
   - If you add/rename/remove any `rules/*.md`, update the corresponding links in `SKILL.md` in the same change.

2. **Preserve skill metadata/frontmatter format.**
   - Skill and rule docs use YAML frontmatter (`name`, `description`, `metadata.tags`).
   - Frontmatter is required for both `SKILL.md` and every `rules/*.md` file.

3. **Keep naming consistent with existing directory structure.**
   - Add new guidance under an existing skill’s `rules/` unless you are intentionally creating a new skill.
   - Use lowercase kebab-case for directory and file names.

4. **Validate changes with project scripts.**
   - At minimum run `npm run typecheck` and `npm test` after substantial edits.

## Best practices for writing skills

### Skill structure

- Each skill must have a `SKILL.md` with: YAML frontmatter (`name`, `description`, `metadata.tags`), “When to use”, “How to use” (with links to all rules), and “Core principles” sections.
- Each rule must have a `rules/<rule-name>.md` with: YAML frontmatter (`name`, `description`, `metadata.tags`) and actionable content.
- Keep rules focused on a single concern. Prefer multiple small rule files over one large document.

### Content quality

- Provide concrete, copy-pasteable code examples (install commands, config snippets, package.json scripts).
- Show the canonical way to do things first, then alternatives.
- Include common pitfalls and things to avoid.
- Reference official documentation URLs when relevant.
- Use TypeScript in code examples where applicable.

### Consistency

- Follow the same section patterns across skills (frontmatter, install, config, usage, best practices).
- Use imperative tone in instructions (“Install Biome”, “Create config”, not “You should install”).
- Keep code examples minimal — show only what is needed to illustrate the point.

### Migration rules

- When a skill replaces another tool, include a dedicated migration rule file.
- Provide step-by-step migration flows with before/after comparisons.
- List behavioral differences the user should expect.
- Include cleanup steps (uninstall old packages, remove old config files).

### CI and editor integration

- Every skill that involves a CLI tool should have a `ci-and-editor-integration.md` rule.
- CI guidance should always specify the non-modifying command (e.g., `biome ci`, `eslint .` without `--fix`).
- Editor guidance should cover at minimum VS Code settings.
- Include `lint-staged` / pre-commit hook patterns when applicable.

## Existing agent/tooling constraints found in repo

- `.claude/settings.local.json` allows `npx githuman:*` commands for Claude-specific workflows.
- `.githuman/` is intentionally gitignored project tooling state.
