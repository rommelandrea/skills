---
name: insert-update-delete
description: DML operations including inserts, updates, deletes, upserts, and returning clauses
metadata:
  tags: kysely, insert, update, delete, upsert, returning, dml
---

# Insert, Update, Delete

## Insert with Returning

Always use `returning()` on PostgreSQL to get the inserted row back without a second query:

```typescript
const user = await db.insertInto('user')
  .values({
    email: 'alice@example.com',
    name: 'Alice',
  })
  .returningAll()
  .executeTakeFirstOrThrow()
// user.id is populated by the database

// Return specific columns
const { id } = await db.insertInto('user')
  .values({ email: 'bob@example.com', name: 'Bob' })
  .returning('id')
  .executeTakeFirstOrThrow()
```

## Batch Inserts

Insert multiple rows in a single query with `values([...])`:

```typescript
const newUsers = await db.insertInto('user')
  .values([
    { email: 'a@b.com', name: 'Alice' },
    { email: 'c@d.com', name: 'Bob' },
    { email: 'e@f.com', name: 'Charlie' },
  ])
  .returningAll()
  .execute()
// newUsers: User[]
```

## Update

```typescript
// Update with where clause
const updated = await db.updateTable('user')
  .set({ name: 'Alice Smith' })
  .where('id', '=', userId)
  .returningAll()
  .executeTakeFirstOrThrow()

// Update multiple columns
await db.updateTable('post')
  .set({
    title: 'Updated Title',
    published: true,
  })
  .where('id', '=', postId)
  .execute()

// Expression builder with ref and fn
await db.updateTable('user')
  .set(({ ref, fn }) => ({
    login_count: ref('login_count').add(1),
    last_login_at: fn.now(),
  }))
  .where('id', '=', userId)
  .execute()
```

## Delete

```typescript
// Delete with returning
const deleted = await db.deleteFrom('post')
  .where('id', '=', postId)
  .returningAll()
  .executeTakeFirstOrThrow()

// Delete with multiple conditions
await db.deleteFrom('session')
  .where('expires_at', '<', new Date().toISOString())
  .execute()

// Check how many rows were deleted
const result = await db.deleteFrom('session')
  .where('user_id', '=', userId)
  .executeTakeFirst()
// result.numDeletedRows: bigint
```

## Upsert with onConflict

Handle INSERT-or-UPDATE in a single atomic operation:

```typescript
// Insert or update on conflict
const user = await db.insertInto('user')
  .values({
    email: 'alice@example.com',
    name: 'Alice',
  })
  .onConflict((oc) =>
    oc.column('email').doUpdateSet({
      name: 'Alice',
    })
  )
  .returningAll()
  .executeTakeFirstOrThrow()

// Insert or do nothing on conflict
await db.insertInto('user')
  .values({ email: 'alice@example.com', name: 'Alice' })
  .onConflict((oc) => oc.column('email').doNothing())
  .execute()

// Upsert with composite unique constraint
await db.insertInto('user_role')
  .values({ user_id: 1, role_id: 2 })
  .onConflict((oc) =>
    oc.columns(['user_id', 'role_id']).doNothing()
  )
  .execute()

// Reference excluded values in the update
await db.insertInto('product')
  .values({ sku: 'ABC', price: 29.99, stock: 100 })
  .onConflict((oc) =>
    oc.column('sku').doUpdateSet((eb) => ({
      price: eb.ref('excluded.price'),
      stock: eb.ref('excluded.stock'),
    }))
  )
  .execute()
```

## Insert with Complex Values

Use the expression builder for computed values, references, and subqueries:

```typescript
import { sql } from 'kysely'

const user = await db.insertInto('user')
  .values(({ ref, selectFrom, fn }) => ({
    email: 'alice@example.com',
    name: 'Alice',
    display_name: sql<string>`concat('Ms. ', 'Alice')`,
    department_id: selectFrom('department')
      .select('id')
      .where('name', '=', 'Engineering'),
  }))
  .returningAll()
  .executeTakeFirstOrThrow()
```

## INSERT INTO ... SELECT

Insert rows from a query result:

```typescript
await db.insertInto('archive_post')
  .columns(['title', 'body', 'archived_at'])
  .expression((eb) =>
    eb.selectFrom('post')
      .select([
        'post.title',
        'post.body',
        eb.val(new Date().toISOString()).as('archived_at'),
      ])
      .where('post.published', '=', false)
  )
  .execute()
```

## MERGE (PostgreSQL 15+, MSSQL)

Use `mergeInto` for conditional insert/update/delete based on row existence:

```typescript
const result = await db
  .mergeInto('product as target')
  .using('product_import as source', 'source.sku', 'target.sku')
  .whenMatchedAnd('source.price', '>', 0)
  .thenUpdateSet((eb) => ({
    price: eb.ref('source.price'),
    stock: eb.ref('source.stock'),
  }))
  .whenNotMatched()
  .thenInsertValues((eb) => ({
    sku: eb.ref('source.sku'),
    price: eb.ref('source.price'),
    stock: eb.ref('source.stock'),
  }))
  .executeTakeFirstOrThrow()
```

## Using Insertable/Updateable Types

Type function parameters with the derived helper types:

```typescript
import type { Insertable, Updateable } from 'kysely'

type NewPost = Insertable<PostTable>
type PostUpdate = Updateable<PostTable>

async function createPost(data: NewPost) {
  return db.insertInto('post')
    .values(data)
    .returningAll()
    .executeTakeFirstOrThrow()
}

async function updatePost(id: number, data: PostUpdate) {
  return db.updateTable('post')
    .set(data)
    .where('id', '=', id)
    .returningAll()
    .executeTakeFirstOrThrow()
}
```

## Pitfalls

- **Always use `returning()` on PostgreSQL** — avoids a second query to fetch the inserted/updated row.
- **`numDeletedRows` / `numUpdatedRows` are `bigint`** — compare with `BigInt(0)` or convert with `Number()`.
- **Batch inserts share the same column set** — every object in the array must have the same keys.
- **`onConflict` requires a unique constraint or index** on the target columns.
- **`mergeInto` is not supported on SQLite or older PostgreSQL (<15)** — use `onConflict` for upserts on those platforms.
