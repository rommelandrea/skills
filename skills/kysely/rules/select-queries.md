---
name: select-queries
description: SELECT patterns including joins, subqueries, CTEs, and narrow selects
metadata:
  tags: kysely, select, query, join, subquery, cte, where
---

# SELECT Queries

## Narrow Selects

Always select specific columns instead of using `selectAll()`. This improves performance and produces precise result types:

```typescript
// Good — narrow select
const users = await db.selectFrom('user')
  .select(['id', 'email', 'name'])
  .execute()
// Type: { id: number; email: string; name: string }[]

// Avoid — selectAll pulls every column
const users = await db.selectFrom('user')
  .selectAll()
  .execute()
```

## WHERE Clauses

```typescript
// Equality
const user = await db.selectFrom('user')
  .select(['id', 'email'])
  .where('email', '=', 'a@b.com')
  .executeTakeFirst()

// Multiple conditions
const posts = await db.selectFrom('post')
  .select(['id', 'title'])
  .where('published', '=', true)
  .where('user_id', '=', userId)
  .execute()

// OR conditions
const posts = await db.selectFrom('post')
  .select(['id', 'title'])
  .where((eb) =>
    eb.or([
      eb('status', '=', 'published'),
      eb('status', '=', 'draft'),
    ])
  )
  .execute()

// IN clause
const users = await db.selectFrom('user')
  .select(['id', 'name'])
  .where('id', 'in', [1, 2, 3])
  .execute()

// IS NULL / IS NOT NULL
const unverified = await db.selectFrom('user')
  .select(['id', 'email'])
  .where('verified_at', 'is', null)
  .execute()
```

## Execute Methods

Choose the right execute method for your use case:

```typescript
// execute() — returns all rows as an array (may be empty)
const rows = await db.selectFrom('user').selectAll().execute()

// executeTakeFirst() — returns first row or undefined
const user = await db.selectFrom('user')
  .selectAll()
  .where('id', '=', 1)
  .executeTakeFirst()
// Type: User | undefined

// executeTakeFirstOrThrow() — returns first row or throws NoResultError
const user = await db.selectFrom('user')
  .selectAll()
  .where('id', '=', 1)
  .executeTakeFirstOrThrow()
// Type: User (no undefined)
```

## Joins

```typescript
// Inner join
const postsWithAuthor = await db.selectFrom('post')
  .innerJoin('user', 'user.id', 'post.user_id')
  .select([
    'post.id',
    'post.title',
    'user.name as author_name',
  ])
  .execute()

// Left join — joined columns become nullable
const usersWithPosts = await db.selectFrom('user')
  .leftJoin('post', 'post.user_id', 'user.id')
  .select([
    'user.id',
    'user.name',
    'post.title',  // Type: string | null
  ])
  .execute()

// Multi-table join
const comments = await db.selectFrom('comment')
  .innerJoin('post', 'post.id', 'comment.post_id')
  .innerJoin('user', 'user.id', 'comment.user_id')
  .select([
    'comment.id',
    'comment.body',
    'post.title as post_title',
    'user.name as author_name',
  ])
  .execute()
```

## Ordering, Grouping, Pagination

```typescript
// Order by
const users = await db.selectFrom('user')
  .select(['id', 'name', 'created_at'])
  .orderBy('created_at', 'desc')
  .execute()

// Group by with aggregate
const postCounts = await db.selectFrom('post')
  .select([
    'user_id',
    (eb) => eb.fn.count<number>('id').as('post_count'),
  ])
  .groupBy('user_id')
  .having((eb) => eb.fn.count('id'), '>', 5)
  .execute()

// Pagination with limit/offset
const page = await db.selectFrom('user')
  .select(['id', 'name'])
  .orderBy('id')
  .limit(20)
  .offset(40)
  .execute()
```

## Distinct

```typescript
// Simple distinct
const names = await db.selectFrom('user')
  .select('name')
  .distinct()
  .execute()

// Distinct on (PostgreSQL only) — one row per unique owner_id
const owners = await db.selectFrom('user')
  .innerJoin('pet', 'pet.owner_id', 'user.id')
  .distinctOn('user.id')
  .selectAll('user')
  .execute()
```

## Expression Builder

Use the expression builder callback for type-safe dynamic expressions:

```typescript
const users = await db.selectFrom('user')
  .select(({ eb, selectFrom, val, lit, fn, or }) => [
    'user.id',
    // Correlated subquery
    selectFrom('post')
      .whereRef('post.user_id', '=', 'user.id')
      .select('post.title')
      .limit(1)
      .as('first_post_title'),
    // Boolean expression
    or([
      eb('name', '=', 'Alice'),
      eb('name', '=', 'Bob'),
    ]).as('is_target'),
    // Static value
    val('constant').as('static_value'),
    // Literal (unparameterized)
    lit(42).as('literal_number'),
    // Aggregate function
    fn.count<number>('id').as('total'),
  ])
  .execute()
```

## Function Calls with fn

```typescript
const stats = await db.selectFrom('user')
  .innerJoin('post', 'post.user_id', 'user.id')
  .select(({ fn, val }) => [
    'user.id',
    fn.count<number>('post.id').as('post_count'),
    fn.max('post.created_at').as('latest_post'),
    fn.agg<string[]>('array_agg', ['post.title']).as('post_titles'),
    fn<string>('concat', [val('User: '), 'user.name']).as('label'),
  ])
  .groupBy('user.id')
  .having((eb) => eb.fn.count('post.id'), '>', 5)
  .execute()
```

## Subqueries

```typescript
// Subquery in select
const usersWithPostCount = await db.selectFrom('user')
  .select([
    'user.id',
    'user.name',
    (eb) =>
      eb.selectFrom('post')
        .whereRef('post.user_id', '=', 'user.id')
        .select(eb.fn.count<number>('post.id').as('count'))
        .as('post_count'),
  ])
  .execute()

// Subquery in where
const activeAuthors = await db.selectFrom('user')
  .select(['id', 'name'])
  .where('id', 'in',
    db.selectFrom('post')
      .select('user_id')
      .where('published', '=', true)
  )
  .execute()

// Subquery join
const result = await db.selectFrom('user')
  .innerJoin(
    (eb) => eb
      .selectFrom('post')
      .select(['user_id as owner', 'title'])
      .where('published', '=', true)
      .as('published_posts'),
    (join) => join.onRef('published_posts.owner', '=', 'user.id'),
  )
  .selectAll('published_posts')
  .execute()
```

## Common Table Expressions (CTEs)

```typescript
const result = await db
  .with('active_users', (qb) =>
    qb.selectFrom('user')
      .select(['id', 'name'])
      .where('active', '=', true)
  )
  .with('user_posts', (qb) =>
    qb.selectFrom('post')
      .innerJoin('active_users', 'active_users.id', 'post.user_id')
      .select([
        'active_users.name',
        'post.title',
      ])
  )
  .selectFrom('user_posts')
  .selectAll()
  .execute()
```

## Dynamic Queries

Build queries conditionally using `$if` or standard conditionals:

```typescript
// Using $if — chainable and concise
function findUsers(filters: {
  name?: string
  email?: string
  active?: boolean
}) {
  return db.selectFrom('user')
    .select(['id', 'name', 'email'])
    .$if(filters.name != null, (qb) =>
      qb.where('name', 'ilike', `%${filters.name}%`)
    )
    .$if(filters.email != null, (qb) =>
      qb.where('email', '=', filters.email!)
    )
    .$if(filters.active !== undefined, (qb) =>
      qb.where('active', '=', filters.active!)
    )
    .execute()
}

// Using let reassignment — better for complex branching
function findUsersAlt(filters: {
  name?: string
  email?: string
  active?: boolean
}) {
  let query = db.selectFrom('user').select(['id', 'name', 'email'])

  if (filters.name) {
    query = query.where('name', 'ilike', `%${filters.name}%`)
  }
  if (filters.email) {
    query = query.where('email', '=', filters.email)
  }
  if (filters.active !== undefined) {
    query = query.where('active', '=', filters.active)
  }

  return query.execute()
}
```

## CTEs with DML (PostgreSQL)

CTEs can contain INSERT/UPDATE/DELETE, enabling multi-step operations in a single statement:

```typescript
const result = await db
  .with('new_user', (db) => db
    .insertInto('user')
    .values({ email: 'alice@example.com', name: 'Alice' })
    .returning('id')
  )
  .with('new_profile', (db) => db
    .insertInto('profile')
    .values({
      user_id: db.selectFrom('new_user').select('id'),
      bio: '',
    })
    .returning('id')
  )
  .selectFrom(['new_user', 'new_profile'])
  .select(['new_user.id as user_id', 'new_profile.id as profile_id'])
  .executeTakeFirstOrThrow()
```

## Pitfalls

- **`selectAll()` breaks when columns change** — narrow selects keep queries stable.
- **Left join columns are nullable** — Kysely correctly types them as `T | null`.
- **`executeTakeFirst()` returns `undefined` on no match** — use `executeTakeFirstOrThrow()` when you expect a result.
- **Aggregate functions need explicit type parameters** — use `eb.fn.count<number>('id')` to avoid `string | number | bigint`.
- **Always qualify columns in joins** — `select('id')` is ambiguous when two tables have `id`. Use `'user.id'`.
- **`distinctOn` is PostgreSQL only** — it is not available on MySQL or SQLite.
