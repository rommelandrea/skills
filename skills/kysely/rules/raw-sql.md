---
name: raw-sql
description: Using the sql template tag safely, parameterization, and sql.raw dangers
metadata:
  tags: kysely, raw-sql, parameterization, sql-tag, security
---

# Raw SQL

## The sql Template Tag

Use the `sql` template tag for any SQL that Kysely's query builder does not cover. Template expressions are automatically parameterized:

```typescript
import { sql } from 'kysely'

// Parameters are safely escaped — no SQL injection risk
const users = await sql<{ id: number; name: string }>`
  SELECT id, name FROM "user" WHERE email = ${email}
`.execute(db)

// Use in a query builder
const results = await db.selectFrom('user')
  .select(['id', 'name'])
  .where(sql`lower(email)`, '=', email.toLowerCase())
  .execute()
```

## Type the Result

Pass a type parameter to `sql<T>` to type the result:

```typescript
interface CountResult {
  count: number
}

const result = await sql<CountResult>`
  SELECT count(*)::int as count FROM "user"
`.execute(db)

const count = result.rows[0].count
```

## sql in SELECT, WHERE, and ORDER BY

```typescript
// Computed column in select
const users = await db.selectFrom('user')
  .select([
    'id',
    sql<string>`concat(first_name, ' ', last_name)`.as('full_name'),
  ])
  .execute()

// In WHERE
const recent = await db.selectFrom('post')
  .selectAll()
  .where(sql`created_at > now() - interval '7 days'`)
  .execute()

// In ORDER BY
const sorted = await db.selectFrom('product')
  .selectAll()
  .orderBy(sql`price * (1 - discount_rate)`, 'asc')
  .execute()
```

## NEVER Concatenate User Input

```typescript
// DANGEROUS — SQL injection vulnerability
const users = await sql`
  SELECT * FROM "user" WHERE name = '${name}'
`.execute(db)

// SAFE — parameterized
const users = await sql`
  SELECT * FROM "user" WHERE name = ${name}
`.execute(db)
```

The difference: `${name}` inside a `sql` template literal becomes a parameterized placeholder (`$1`). String concatenation or manual quoting bypasses parameterization.

## sql.raw() — Use With Extreme Caution

`sql.raw()` inserts a string directly into the query without parameterization. Only use it with trusted, hardcoded values:

```typescript
// OK — hardcoded column name
const order = sortColumn === 'name' ? 'name' : 'created_at'
const users = await db.selectFrom('user')
  .selectAll()
  .orderBy(sql.raw(order), 'asc')
  .execute()

// NEVER do this — user input goes directly into SQL
const users = await db.selectFrom('user')
  .selectAll()
  .orderBy(sql.raw(req.query.sort), 'asc')  // SQL injection!
  .execute()
```

## sql.ref() for Dynamic Column References

Use `sql.ref()` to safely reference columns by name:

```typescript
const column = 'email'  // validated against allowed columns
const users = await db.selectFrom('user')
  .select([sql.ref(column).as('value')])
  .execute()
```

## sql.table() for Dynamic Table References

```typescript
const tableName = 'user'
const rows = await sql`
  SELECT * FROM ${sql.table(tableName)}
`.execute(db)
```

## sql.lit() for Literal Values

`sql.lit()` inserts a literal value (number, string, boolean) directly. Only use with trusted values:

```typescript
// Safe — hardcoded literal
const users = await db.selectFrom('user')
  .select([
    'id',
    sql.lit('active').as('status'),
  ])
  .execute()
```

## Using sql with Fragments

Compose reusable SQL fragments:

```typescript
function fullName() {
  return sql<string>`concat(first_name, ' ', last_name)`
}

const users = await db.selectFrom('user')
  .select([
    'id',
    fullName().as('full_name'),
  ])
  .execute()
```

## Pitfalls

- **`sql.raw()` is the #1 source of SQL injection in Kysely** — whitelist values before passing them.
- **Template expressions in `sql` are parameterized** — do not add quotes around `${value}`.
- **`sql` must call `.execute(db)` or be used inside a query builder** — it does not execute by itself.
- **Type `sql<T>` explicitly** — without a type parameter, the result type is `unknown`.
