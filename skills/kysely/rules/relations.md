---
name: relations
description: JSON aggregation helpers, avoiding N+1 queries, and nested data loading
metadata:
  tags: kysely, relations, json, aggregation, n-plus-one, jsonArrayFrom
---

# Relations and Nested Data

## The N+1 Problem

Loading related data in a loop causes N+1 queries:

```typescript
// Bad — 1 query for users + N queries for posts
const users = await db.selectFrom('user').selectAll().execute()
for (const user of users) {
  user.posts = await db.selectFrom('post')
    .selectAll()
    .where('user_id', '=', user.id)
    .execute()
}
```

## JSON Aggregation with Kysely Helpers

Use `jsonArrayFrom` and `jsonObjectFrom` from `kysely/helpers/postgres` to load nested data in a single query:

```typescript
import { jsonArrayFrom, jsonObjectFrom } from 'kysely/helpers/postgres'

// Load users with their posts in one query
const usersWithPosts = await db.selectFrom('user')
  .select([
    'user.id',
    'user.name',
    (eb) =>
      jsonArrayFrom(
        eb.selectFrom('post')
          .select(['post.id', 'post.title', 'post.created_at'])
          .whereRef('post.user_id', '=', 'user.id')
          .orderBy('post.created_at', 'desc')
      ).as('posts'),
  ])
  .execute()

// Result type: { id: number; name: string; posts: { id: number; title: string; created_at: Date }[] }[]
```

## jsonObjectFrom for One-to-One

```typescript
// Load post with its author
const postWithAuthor = await db.selectFrom('post')
  .select([
    'post.id',
    'post.title',
    (eb) =>
      jsonObjectFrom(
        eb.selectFrom('user')
          .select(['user.id', 'user.name', 'user.email'])
          .whereRef('user.id', '=', 'post.user_id')
      ).as('author'),
  ])
  .executeTakeFirstOrThrow()

// Result type: { id: number; title: string; author: { id: number; name: string; email: string } | null }
```

Note: `jsonObjectFrom` returns `null` if no matching row is found.

## Nested Relations

Nest `jsonArrayFrom` calls for multi-level relations:

```typescript
const usersWithPostsAndComments = await db.selectFrom('user')
  .select([
    'user.id',
    'user.name',
    (eb) =>
      jsonArrayFrom(
        eb.selectFrom('post')
          .select([
            'post.id',
            'post.title',
            (eb2) =>
              jsonArrayFrom(
                eb2.selectFrom('comment')
                  .select(['comment.id', 'comment.body'])
                  .whereRef('comment.post_id', '=', 'post.id')
              ).as('comments'),
          ])
          .whereRef('post.user_id', '=', 'user.id')
      ).as('posts'),
  ])
  .execute()
```

## When to Use Separate Queries

JSON aggregation is not always the best approach. Use separate queries when:

- The nested data is large (many rows per parent) — JSON aggregation duplicates parent columns
- You need to paginate the nested data independently
- The nested query is complex with its own filtering/sorting

```typescript
// Better as two queries when posts are numerous
const users = await db.selectFrom('user')
  .select(['id', 'name'])
  .execute()

const userIds = users.map((u) => u.id)

const posts = await db.selectFrom('post')
  .select(['id', 'title', 'user_id'])
  .where('user_id', 'in', userIds)
  .execute()

// Group in application code
const postsByUser = Map.groupBy(posts, (p) => p.user_id)
```

## Filtering on Nested Data

Use `exists` to filter parents based on child data:

```typescript
// Users who have at least one published post
const activeAuthors = await db.selectFrom('user')
  .select(['id', 'name'])
  .where((eb) =>
    eb.exists(
      eb.selectFrom('post')
        .select(sql.lit(1).as('one'))
        .whereRef('post.user_id', '=', 'user.id')
        .where('post.published', '=', true)
    )
  )
  .execute()
```

## Pitfalls

- **Import from `kysely/helpers/postgres`** — not from the main `kysely` module. Other dialects have their own helpers (e.g., `kysely/helpers/mysql`).
- **`jsonObjectFrom` returns `null`** if no row matches — handle the null case.
- **JSON aggregation returns `[]` for empty sets** — jsonArrayFrom never returns null.
- **Large nested datasets bloat the query result** — profile and consider separate queries if response size grows.
