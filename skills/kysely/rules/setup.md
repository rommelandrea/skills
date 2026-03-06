---
name: setup
description: Installing Kysely and configuring the PostgreSQL dialect
metadata:
  tags: kysely, setup, postgres, dialect, configuration
---

# Setup

## Install Kysely with PostgreSQL

```bash
npm install kysely pg
npm install --save-dev @types/pg
```

For MySQL:

```bash
npm install kysely mysql2
```

For SQLite:

```bash
npm install kysely better-sqlite3
npm install --save-dev @types/better-sqlite3
```

## Define the Database Interface

Create a single file exporting your database type. Every table is a key, every value describes the row shape:

```typescript
// src/db/types.ts
import type { Generated, ColumnType } from 'kysely'

interface Database {
  user: UserTable
  post: PostTable
}

interface UserTable {
  id: Generated<number>
  email: string
  name: string
  created_at: ColumnType<Date, string | undefined, never>
  updated_at: ColumnType<Date, string | undefined, never>
}

interface PostTable {
  id: Generated<number>
  user_id: number
  title: string
  body: string
  published: boolean
  created_at: ColumnType<Date, string | undefined, never>
  updated_at: ColumnType<Date, string | undefined, never>
}

export type { Database }
```

## Create the Kysely Instance

Create one instance and reuse it across the application:

```typescript
// src/db/index.ts
import { Kysely, PostgresDialect } from 'kysely'
import { Pool } from 'pg'
import type { Database } from './types.js'

const db = new Kysely<Database>({
  dialect: new PostgresDialect({
    pool: new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 10,
    }),
  }),
})

export { db }
```

## Other Dialects

```typescript
// MySQL
import { Kysely, MysqlDialect } from 'kysely'
import { createPool } from 'mysql2'

const db = new Kysely<Database>({
  dialect: new MysqlDialect({
    pool: createPool({
      uri: process.env.DATABASE_URL,
    }),
  }),
})

// SQLite
import { Kysely, SqliteDialect } from 'kysely'
import Database from 'better-sqlite3'

const db = new Kysely<Database>({
  dialect: new SqliteDialect({
    database: new Database('app.db'),
  }),
})
```

Third-party dialects are available for PlanetScale, Cloudflare D1, Neon, and others — install the dialect package and pass it to `new Kysely()`.

## Graceful Shutdown

Always destroy the Kysely instance on process exit to close the connection pool:

```typescript
async function shutdown() {
  await db.destroy()
  process.exit(0)
}

process.on('SIGTERM', shutdown)
process.on('SIGINT', shutdown)
```

In Fastify, use the `onClose` hook:

```typescript
fastify.addHook('onClose', async () => {
  await db.destroy()
})
```

## Logging

Enable query logging in development:

```typescript
import { Kysely, PostgresDialect } from 'kysely'

const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
  log(event) {
    if (event.level === 'query') {
      console.log(event.query.sql)
      console.log(event.query.parameters)
    }
  },
})
```

For production, log only errors or slow queries:

```typescript
const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
  log(event) {
    if (event.level === 'error') {
      console.error('Query failed:', event.error)
    }
    if (event.level === 'query' && event.queryDurationMillis > 1000) {
      console.warn('Slow query:', event.query.sql)
    }
  },
})
```

## Introspection

Query database metadata at runtime:

```typescript
const tables = await db.introspection.getTables()

for (const table of tables) {
  console.log(table.name)
  for (const column of table.columns) {
    console.log(`  ${column.name}: ${column.dataType}`)
  }
}
```

Useful for codegen scripts, admin tools, and dynamic schema exploration.

## Pitfalls

- **Do not create multiple Kysely instances** for the same database. Each instance creates its own connection pool.
- **Do not import `pg` Pool types** as a runtime dep in your types file — keep types separate from runtime code.
- **Always set `max` on the pool** — the default (10) may be too high or too low for your workload.
