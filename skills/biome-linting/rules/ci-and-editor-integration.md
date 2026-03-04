---
name: ci-and-editor-integration
description: Integrate Biome with CI pipelines, pre-commit hooks, and editors
metadata:
  tags: ci, linting, formatting, biome, pre-commit, vscode
---

## CI guidance

- Run `biome ci` in CI as a required step:

```bash
npx biome ci .
```

- `biome ci` is purpose-built for CI: it checks lint, format, and import sorting, produces detailed diagnostics, and exits non-zero on any issue.
- Do not use `--write` in CI — it should only validate, never modify.
- Keep CI Node.js version aligned with local development and project engines.

## Pre-commit hook pattern

Use `lint-staged` (or equivalent) to check only changed files before commit:

```json
{
  "lint-staged": {
    "*.{js,mjs,cjs,ts,mts,cts,jsx,tsx,json,css}": [
      "biome check --write --no-errors-on-unmatched"
    ]
  }
}
```

The `--no-errors-on-unmatched` flag prevents errors when staged files include types Biome doesn't handle.

## VS Code integration

- Install the **Biome** extension (`biomejs.biome`) from the marketplace
- Set Biome as the default formatter for supported languages

Recommended workspace settings (`.vscode/settings.json`):

```json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },
  "[javascript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[typescript]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[json]": {
    "editor.defaultFormatter": "biomejs.biome"
  },
  "[css]": {
    "editor.defaultFormatter": "biomejs.biome"
  }
}
```

## Other editors

- **IntelliJ/WebStorm:** Install the Biome plugin from JetBrains marketplace
- **Neovim:** Use `nvim-lspconfig` with `biome` LSP or `none-ls` integration
- **Zed:** First-party Biome support built in

## Disabling conflicting tools

When migrating to Biome, disable or remove competing formatter/linter extensions:
- Disable Prettier extension (or set Biome as default formatter per language)
- Disable ESLint extension for languages Biome covers
- Remove `editor.defaultFormatter: esbenp.prettier-vscode` from settings
