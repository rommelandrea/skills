---
name: plugins
description: CamelCasePlugin, DeduplicateJoinsPlugin, and writing custom Kysely plugins
metadata:
  tags: kysely, plugins, camel-case, custom-plugin, transform
---

# Plugins

## CamelCasePlugin

Map between snake_case database columns and camelCase TypeScript properties automatically:

```typescript
import { Kysely, PostgresDialect, CamelCasePlugin } from 'kysely'

const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
  plugins: [new CamelCasePlugin()],
})

// Database column: created_at
// TypeScript property: createdAt
interface UserTable {
  id: Generated<number>
  firstName: string       // maps to first_name in DB
  lastName: string        // maps to last_name in DB
  createdAt: Generated<Date>  // maps to created_at in DB
}

const user = await db.selectFrom('user')
  .select(['id', 'firstName', 'lastName'])
  .executeTakeFirstOrThrow()
// user.firstName, user.lastName — camelCase in TypeScript
```

Queries use camelCase; the plugin translates to snake_case for SQL:

```typescript
// Write camelCase
await db.selectFrom('user')
  .select(['firstName'])
  .where('createdAt', '>', cutoff)
  .orderBy('createdAt', 'desc')
  .execute()

// Generated SQL uses snake_case:
// SELECT "first_name" FROM "user" WHERE "created_at" > $1 ORDER BY "created_at" DESC
```

## DeduplicateJoinsPlugin

Prevents the same table from being joined multiple times in complex queries:

```typescript
import { Kysely, PostgresDialect, DeduplicateJoinsPlugin } from 'kysely'

const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
  plugins: [new DeduplicateJoinsPlugin()],
})
```

Useful when composing queries from multiple sources that may each add the same join.

## ParseJSONResultsPlugin

Automatically parse JSON/JSONB columns from strings to objects:

```typescript
import { Kysely, PostgresDialect, ParseJSONResultsPlugin } from 'kysely'

const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
  plugins: [new ParseJSONResultsPlugin()],
})

// Without plugin: settings is a JSON string
// With plugin: settings is a parsed object
const user = await db.selectFrom('user')
  .select(['id', 'settings'])
  .executeTakeFirstOrThrow()
// user.settings is already an object, not a string
```

## Combining Plugins

Pass multiple plugins in the array — they execute in order:

```typescript
const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
  plugins: [
    new CamelCasePlugin(),
    new ParseJSONResultsPlugin(),
  ],
})
```

## Writing a Custom Plugin

Implement the `KyselyPlugin` interface with `transformQuery` and `transformResult`:

```typescript
import type {
  KyselyPlugin,
  PluginTransformQueryArgs,
  PluginTransformResultArgs,
  RootOperationNode,
  QueryResult,
  UnknownRow,
} from 'kysely'

class QueryTimingPlugin implements KyselyPlugin {
  transformQuery(args: PluginTransformQueryArgs): RootOperationNode {
    // Called before query execution — can modify the query AST
    return args.node  // Return unmodified
  }

  async transformResult(
    args: PluginTransformResultArgs,
  ): Promise<QueryResult<UnknownRow>> {
    // Called after query execution — can modify results
    return args.result
  }
}
```

Example — a plugin that adds `updated_at` to every UPDATE:

```typescript
import {
  OperationNodeTransformer,
  UpdateQueryNode,
  ColumnUpdateNode,
  ColumnNode,
  ValueNode,
} from 'kysely'
import type {
  KyselyPlugin,
  PluginTransformQueryArgs,
  PluginTransformResultArgs,
  RootOperationNode,
  QueryResult,
  UnknownRow,
} from 'kysely'

class AutoUpdatedAtPlugin implements KyselyPlugin {
  private transformer = new AutoUpdatedAtTransformer()

  transformQuery(args: PluginTransformQueryArgs): RootOperationNode {
    return this.transformer.transformNode(args.node)
  }

  async transformResult(
    args: PluginTransformResultArgs,
  ): Promise<QueryResult<UnknownRow>> {
    return args.result
  }
}

class AutoUpdatedAtTransformer extends OperationNodeTransformer {
  protected transformUpdateQuery(node: UpdateQueryNode): UpdateQueryNode {
    const transformed = super.transformUpdateQuery(node)
    return {
      ...transformed,
      updates: [
        ...(transformed.updates ?? []),
        ColumnUpdateNode.create(
          ColumnNode.create('updated_at'),
          ValueNode.create(new Date().toISOString()),
        ),
      ],
    }
  }
}
```

## Pitfalls

- **CamelCasePlugin affects the entire Database interface** — define all column names in camelCase when using it.
- **Plugin order matters** — transformations are applied left to right for queries, right to left for results.
- **Custom plugins work with the internal AST** — the node types are stable but not documented as public API. Pin your Kysely version if relying on them heavily.
- **Do not use CamelCasePlugin halfway** — either all tables use camelCase in the interface or none do.
