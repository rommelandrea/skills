---
name: type-safety
description: Database type definitions, Generated columns, codegen, and ColumnType
metadata:
  tags: kysely, types, typescript, codegen, generated, column-type
---

# Type Safety

## Database Interface

The `Database` interface maps table names to row types. Kysely uses this to type-check every query at compile time:

```typescript
import type { Generated, ColumnType } from 'kysely'

interface Database {
  user: UserTable
  post: PostTable
  comment: CommentTable
}
```

## Generated Columns

Use `Generated<T>` for columns with defaults (optional on INSERT, present on SELECT). Use `GeneratedAlways<T>` for columns the database always generates (identity columns, computed columns — cannot be inserted or updated):

```typescript
import type { Generated, GeneratedAlways, ColumnType } from 'kysely'

interface UserTable {
  // Generated: optional on INSERT, present on SELECT and UPDATE
  id: Generated<number>
  email: string
  name: string
  // Generated: defaults to current timestamp, optional on INSERT
  created_at: Generated<Date>
  // GeneratedAlways: NEVER provided on INSERT or UPDATE
  row_version: GeneratedAlways<number>
}

// INSERT — id and created_at are optional
await db.insertInto('user')
  .values({ email: 'a@b.com', name: 'Alice' })
  .returningAll()
  .executeTakeFirstOrThrow()

// SELECT — id and created_at are always present
const user = await db.selectFrom('user')
  .selectAll()
  .executeTakeFirstOrThrow()
// user.id: number, user.created_at: Date
```

## ColumnType for Asymmetric Types

Use `ColumnType<SelectType, InsertType, UpdateType>` when a column has different types for read vs write:

```typescript
import type { ColumnType, Generated } from 'kysely'

interface OrderTable {
  id: Generated<number>
  // SELECT: Date, INSERT: string or undefined, UPDATE: never (immutable)
  created_at: ColumnType<Date, string | undefined, never>
  // SELECT: number, INSERT: number, UPDATE: number
  amount: number
  // SELECT: string, INSERT: string, UPDATE: string | undefined
  status: ColumnType<string, string, string | undefined>
}
```

Shorthand helpers:

```typescript
import type { Insertable, Selectable, Updateable } from 'kysely'

// Derive concrete types from the table interface
type User = Selectable<UserTable>
type NewUser = Insertable<UserTable>
type UserUpdate = Updateable<UserTable>

// Use in function signatures
async function createUser(data: NewUser): Promise<User> {
  return db.insertInto('user')
    .values(data)
    .returningAll()
    .executeTakeFirstOrThrow()
}

async function updateUser(id: number, data: UserUpdate): Promise<User> {
  return db.updateTable('user')
    .set(data)
    .where('id', '=', id)
    .returningAll()
    .executeTakeFirstOrThrow()
}
```

## Automatic Type Generation with kysely-codegen

Generate types directly from the database schema:

```bash
npm install --save-dev kysely-codegen
npx kysely-codegen --dialect postgres --url "$DATABASE_URL"
```

This produces a `node_modules/kysely-codegen/dist/db.d.ts` (or a custom output path). Configure the output:

```bash
npx kysely-codegen --dialect postgres --out-file src/db/generated-types.ts
```

Use the generated types:

```typescript
import type { DB } from './generated-types.js'
import { Kysely, PostgresDialect } from 'kysely'

const db = new Kysely<DB>({
  dialect: new PostgresDialect({ pool }),
})
```

Regenerate after every migration to keep types in sync.

## Avoid `any`

Never use `any` in your Database interface. If a column type is unknown, use `unknown` and narrow at the call site:

```typescript
// Bad
interface EventTable {
  id: Generated<number>
  payload: any  // loses all type safety
}

// Good
interface EventTable {
  id: Generated<number>
  payload: unknown
}

// Narrow when reading
const event = await db.selectFrom('event')
  .select(['id', 'payload'])
  .executeTakeFirstOrThrow()

const payload = event.payload as { type: string; data: Record<string, unknown> }
```

## JSON/JSONB Columns

Type JSONB columns explicitly:

```typescript
import type { ColumnType, Generated } from 'kysely'

interface SettingsJson {
  theme: 'light' | 'dark'
  notifications: boolean
  language: string
}

interface UserTable {
  id: Generated<number>
  email: string
  // JSONB: stored as JSON string on insert, parsed on select
  settings: ColumnType<SettingsJson, string, string>
}
```

## Pitfalls

- **Do not define tables directly in the Database interface** — use named interfaces for each table to keep things readable and reusable with `Selectable<T>` / `Insertable<T>`.
- **Keep generated types in sync** — run `kysely-codegen` in CI or as a pre-commit hook after migrations.
- **`Generated<T>` only affects insert types** — the column is always present on select and update.
- **`GeneratedAlways<T>` forbids insert AND update** — use it for identity columns and database-computed columns, not for columns with simple defaults.
