---
name: migrations
description: Migration setup, FileMigrationProvider, up/down, and schema builder API
metadata:
  tags: kysely, migration, schema, ddl, create-table, alter-table
---

# Migrations

## Migration File Structure

Each migration is a TypeScript file exporting `up` and `down` functions:

```typescript
// migrations/001_create_user_table.ts
import type { Kysely } from 'kysely'

export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .createTable('user')
    .addColumn('id', 'serial', (col) => col.primaryKey())
    .addColumn('email', 'varchar(255)', (col) => col.notNull().unique())
    .addColumn('name', 'varchar(255)', (col) => col.notNull())
    .addColumn('created_at', 'timestamptz', (col) =>
      col.notNull().defaultTo('now()')
    )
    .execute()
}

export async function down(db: Kysely<any>): Promise<void> {
  await db.schema.dropTable('user').execute()
}
```

## FileMigrationProvider

Point the migrator at a directory of migration files:

```typescript
import { promises as fs } from 'node:fs'
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { Migrator, FileMigrationProvider } from 'kysely'

const __dirname = path.dirname(fileURLToPath(import.meta.url))

const migrator = new Migrator({
  db,
  provider: new FileMigrationProvider({
    fs,
    path,
    migrationFolder: path.join(__dirname, 'migrations'),
  }),
})
```

## Run Migrations Programmatically

```typescript
async function migrateToLatest() {
  const { error, results } = await migrator.migrateToLatest()

  for (const result of results ?? []) {
    if (result.status === 'Success') {
      console.log(`Migration "${result.migrationName}" executed successfully`)
    } else if (result.status === 'Error') {
      console.error(`Migration "${result.migrationName}" failed`)
    }
  }

  if (error) {
    console.error('Migration failed:', error)
    process.exit(1)
  }
}

migrateToLatest()
```

Create a standalone script to run migrations:

```typescript
// scripts/migrate.ts
import { db } from '../src/db/index.js'
import { migrator } from '../src/db/migrator.js'

const { error, results } = await migrator.migrateToLatest()

for (const result of results ?? []) {
  console.log(`${result.status}: ${result.migrationName}`)
}

if (error) {
  console.error(error)
  process.exit(1)
}

await db.destroy()
```

## Always Implement down()

Every migration must have a `down()` that reverses the `up()`. This enables rollback during development and in emergencies:

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .alterTable('user')
    .addColumn('avatar_url', 'text')
    .execute()
}

export async function down(db: Kysely<any>): Promise<void> {
  await db.schema
    .alterTable('user')
    .dropColumn('avatar_url')
    .execute()
}
```

## Schema Builder API

### Create Table

```typescript
await db.schema
  .createTable('post')
  .addColumn('id', 'serial', (col) => col.primaryKey())
  .addColumn('user_id', 'integer', (col) =>
    col.notNull().references('user.id').onDelete('cascade')
  )
  .addColumn('title', 'varchar(255)', (col) => col.notNull())
  .addColumn('body', 'text', (col) => col.notNull())
  .addColumn('published', 'boolean', (col) => col.notNull().defaultTo(false))
  .addColumn('created_at', 'timestamptz', (col) =>
    col.notNull().defaultTo('now()')
  )
  .execute()
```

### Create Index

```typescript
await db.schema
  .createIndex('idx_post_user_id')
  .on('post')
  .column('user_id')
  .execute()

// Unique index
await db.schema
  .createIndex('idx_user_email')
  .on('user')
  .column('email')
  .unique()
  .execute()

// Composite index
await db.schema
  .createIndex('idx_post_user_published')
  .on('post')
  .columns(['user_id', 'published'])
  .execute()
```

### Alter Table

```typescript
// Add column
await db.schema
  .alterTable('user')
  .addColumn('bio', 'text')
  .execute()

// Drop column
await db.schema
  .alterTable('user')
  .dropColumn('bio')
  .execute()

// Rename column
await db.schema
  .alterTable('user')
  .renameColumn('name', 'full_name')
  .execute()

// Alter column type
await db.schema
  .alterTable('user')
  .alterColumn('name', (col) => col.setDataType('text'))
  .execute()

// Add not null constraint
await db.schema
  .alterTable('user')
  .alterColumn('email', (col) => col.setNotNull())
  .execute()
```

### Drop Table

```typescript
// Drop if exists
await db.schema.dropTable('temp_data').ifExists().execute()
```

## Migration Naming

Name migrations with a numeric prefix for ordering. Use descriptive names:

```
001_create_user_table.ts
002_create_post_table.ts
003_add_user_avatar_url.ts
004_create_comment_table.ts
```

## Pitfalls

- **Use `Kysely<any>` in migration files** — migrations must work regardless of the current Database interface shape.
- **Never modify existing migration files** — create a new migration to alter the schema.
- **Always test `down()` locally** — broken rollbacks are worse than no rollbacks.
- **Run `kysely-codegen` after migrating** to keep TypeScript types in sync.
