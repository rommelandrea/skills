---
name: biome-setup
description: How to install, initialize, and run Biome
metadata:
  tags: biome, linting, formatting, javascript, typescript
---

Biome is a single toolchain for linting, formatting, and import sorting. It replaces ESLint + Prettier with one tool and one config file.

## Install

Pin the exact version for reproducibility:

```bash
npm install --save-dev --save-exact @biomejs/biome
```

Or with other package managers:

```bash
pnpm add --save-dev --save-exact @biomejs/biome
bun add --dev --exact @biomejs/biome
yarn add --dev --exact @biomejs/biome
```

## Initialize config

```bash
npx @biomejs/biome init
```

This creates a `biome.json` with recommended defaults.

## Run commands

- **Check all (lint + format + import sorting):**

```bash
npx biome check --write .
```

- **Lint only:**

```bash
npx biome lint --write .
```

- **Format only:**

```bash
npx biome format --write .
```

- **CI mode (no writes, exits non-zero on issues):**

```bash
npx biome ci .
```

## Package scripts

```json
{
  "scripts": {
    "check": "biome check --write .",
    "check:ci": "biome ci .",
    "lint": "biome lint --write .",
    "format": "biome format --write ."
  }
}
```

## Key CLI flags

- `--write` — apply safe fixes and formatting in place
- `--changed` — only process files changed since last commit (useful for large repos)
- `--diagnostic-level=error` — only report errors, suppress warnings

## Important notes

- Biome works with zero configuration, but `biome.json` gives you control
- The `--write` flag is safe: it only applies safe fixes (no unsafe auto-fixes unless explicitly configured)
- Use `biome check` as the default local command — it covers lint, format, and import organization in one pass
