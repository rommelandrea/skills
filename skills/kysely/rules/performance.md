---
name: performance
description: Connection pooling, cursor-based pagination, batch operations, and query analysis
metadata:
  tags: kysely, performance, pooling, pagination, batch, explain
---

# Performance

## Connection Pooling

Create a single Kysely instance with a properly sized connection pool:

```typescript
import { Kysely, PostgresDialect } from 'kysely'
import { Pool } from 'pg'

const db = new Kysely<Database>({
  dialect: new PostgresDialect({
    pool: new Pool({
      connectionString: process.env.DATABASE_URL,
      max: 20,                     // Max connections in pool
      idleTimeoutMillis: 30_000,   // Close idle connections after 30s
      connectionTimeoutMillis: 5_000, // Fail if no connection in 5s
    }),
  }),
})
```

Pool sizing rule of thumb: set `max` to 2-3x the number of CPU cores available to your application. Too many connections overwhelm the database; too few cause queuing.

## Cursor-Based Pagination

Avoid OFFSET for large datasets — it forces the database to scan and discard rows. Use cursor-based (keyset) pagination:

```typescript
// Offset pagination — slow on large tables
const page = await db.selectFrom('post')
  .selectAll()
  .orderBy('id')
  .limit(20)
  .offset(10000)  // Scans 10,000 rows then discards them
  .execute()

// Cursor pagination — fast regardless of position
async function getPostsAfter(cursor: number | null, limit: number) {
  let query = db.selectFrom('post')
    .select(['id', 'title', 'created_at'])
    .orderBy('id', 'asc')
    .limit(limit)

  if (cursor) {
    query = query.where('id', '>', cursor)
  }

  const rows = await query.execute()

  return {
    data: rows,
    nextCursor: rows.length === limit ? rows[rows.length - 1].id : null,
  }
}
```

For multi-column cursors (e.g., sort by date then ID):

```typescript
async function getPostsByDate(
  cursor: { createdAt: string; id: number } | null,
  limit: number,
) {
  let query = db.selectFrom('post')
    .select(['id', 'title', 'created_at'])
    .orderBy('created_at', 'desc')
    .orderBy('id', 'desc')
    .limit(limit)

  if (cursor) {
    query = query.where((eb) =>
      eb.or([
        eb('created_at', '<', cursor.createdAt),
        eb.and([
          eb('created_at', '=', cursor.createdAt),
          eb('id', '<', cursor.id),
        ]),
      ])
    )
  }

  return query.execute()
}
```

## Batch Inserts

Insert many rows in a single statement:

```typescript
// Single statement — much faster than individual inserts
await db.insertInto('event')
  .values(events)  // Array of objects
  .execute()
```

For very large batches (10,000+ rows), chunk them to avoid exceeding parameter limits:

```typescript
const BATCH_SIZE = 1000

for (let i = 0; i < rows.length; i += BATCH_SIZE) {
  const chunk = rows.slice(i, i + BATCH_SIZE)
  await db.insertInto('event').values(chunk).execute()
}
```

## Select Only What You Need

Narrow selects reduce data transfer and allow index-only scans:

```typescript
// Bad — transfers all columns, prevents index-only scans
const users = await db.selectFrom('user').selectAll().execute()

// Good — only fetches what's needed
const users = await db.selectFrom('user')
  .select(['id', 'email'])
  .execute()
```

## Inspect Generated SQL with compile()

Use `compile()` to see the exact SQL and parameters without executing:

```typescript
const query = db.selectFrom('user')
  .select(['id', 'email'])
  .where('active', '=', true)

const { sql, parameters } = query.compile()
console.log(sql)        // select "id", "email" from "user" where "active" = $1
console.log(parameters) // [true]
```

This is invaluable for debugging slow queries and verifying index usage.

## Use EXPLAIN ANALYZE

Inspect query plans to catch slow queries. Use raw SQL to run EXPLAIN:

```typescript
import { sql } from 'kysely'

const plan = await sql`
  EXPLAIN ANALYZE
  SELECT id, title FROM post
  WHERE user_id = ${userId}
  ORDER BY created_at DESC
  LIMIT 20
`.execute(db)

for (const row of plan.rows) {
  console.log((row as any)['QUERY PLAN'])
}
```

Look for:
- **Seq Scan** on large tables → add an index
- **Sort** with high cost → add an index matching the ORDER BY
- **Nested Loop** with many iterations → consider a hash or merge join

## Index-Aware Query Patterns

Write queries that can use indexes:

```typescript
// Indexable — equality on indexed column
db.selectFrom('user').selectAll().where('email', '=', email)

// Indexable — range on indexed column
db.selectFrom('post').selectAll().where('created_at', '>', cutoff)

// Not indexable — function on indexed column prevents index use
db.selectFrom('user').selectAll().where(sql`lower(email)`, '=', email)
// Fix: create a functional index on lower(email)
```

## Pitfalls

- **Do not create multiple Kysely instances** — each one creates a separate pool, multiplying connections.
- **OFFSET pagination degrades linearly** — cursor pagination stays constant regardless of page depth.
- **Batch inserts have a parameter limit** — PostgreSQL supports ~65,535 parameters per query. Chunk large inserts.
- **`selectAll()` prevents covering index optimization** — the database must visit the heap for every row.
