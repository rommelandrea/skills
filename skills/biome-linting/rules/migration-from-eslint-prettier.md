---
name: migration-from-eslint-prettier
description: Migrate from ESLint and/or Prettier to Biome
metadata:
  tags: biome, eslint, prettier, migration
---

Use this when a project currently uses ESLint, Prettier, or both, and you want to migrate to Biome.

## Biome migration tool

Biome provides a built-in migration command that reads your existing config and generates an equivalent `biome.json`:

```bash
npx @biomejs/biome migrate eslint --write
npx @biomejs/biome migrate prettier --write
```

These commands translate rules from your existing config into Biome equivalents and write the result to `biome.json`.

## Manual migration flow

1. Install Biome:

```bash
npm install --save-dev --save-exact @biomejs/biome
```

2. Initialize config:

```bash
npx @biomejs/biome init
```

3. Replace lint/format scripts in `package.json`:

```json
{
  "scripts": {
    "check": "biome check --write .",
    "check:ci": "biome ci ."
  }
}
```

4. Run Biome and compare issue count with previous linter output.

5. Cleanup old setup once parity is verified:

```bash
npm uninstall eslint prettier eslint-config-prettier eslint-plugin-prettier
```

Then remove `.eslintrc*`, `.prettierrc*`, `.eslintignore`, `.prettierignore`, and any ESLint/Prettier plugin packages.

## Rules not covered by Biome

Some ESLint plugin rules have no Biome equivalent. For these:
- Check the [Biome rules reference](https://biomejs.dev/linter/rules/) for equivalent rules
- For TypeScript type-checking rules, rely on `tsc --noEmit` instead
- For import order rules, Biome handles import sorting natively via `biome check`

## Behavioral differences to expect

- Biome is significantly faster than ESLint + Prettier
- Biome handles formatting and linting in a single pass
- Some rule names differ; the `migrate` command handles most mappings
- Biome import sorting replaces `eslint-plugin-import` sort rules

## Pitfalls to avoid

- Migrating config and rule semantics in a single large change without validation
- Forgetting to remove old config files (causes confusion with editor extensions)
- Running both ESLint/Prettier and Biome simultaneously in CI
- Assuming all ESLint plugin rules have Biome equivalents — verify coverage first
