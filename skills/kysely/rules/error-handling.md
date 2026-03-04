---
name: error-handling
description: Error types, constraint violations, and executeTakeFirstOrThrow
metadata:
  tags: kysely, error-handling, NoResultError, constraints, postgres-errors
---

# Error Handling

## NoResultError

`executeTakeFirstOrThrow()` throws `NoResultError` when no row matches. Catch it to return 404 or handle missing data:

```typescript
import { NoResultError } from 'kysely'

async function getUserById(id: number) {
  try {
    return await db.selectFrom('user')
      .selectAll()
      .where('id', '=', id)
      .executeTakeFirstOrThrow()
  } catch (error) {
    if (error instanceof NoResultError) {
      throw new NotFoundError(`User ${id} not found`)
    }
    throw error
  }
}
```

## executeTakeFirst vs executeTakeFirstOrThrow

Choose based on whether a missing result is normal or exceptional:

```typescript
// Use executeTakeFirst when no result is a valid outcome
const user = await db.selectFrom('user')
  .selectAll()
  .where('email', '=', email)
  .executeTakeFirst()

if (!user) {
  // Expected: user might not exist
  return null
}

// Use executeTakeFirstOrThrow when a result is required
const config = await db.selectFrom('app_config')
  .selectAll()
  .where('key', '=', 'site_name')
  .executeTakeFirstOrThrow()
// Throws if config is missing — this should never happen
```

## Database Constraint Errors

PostgreSQL signals constraint violations through error codes. Catch them from the underlying driver:

```typescript
async function createUser(data: NewUser) {
  try {
    return await db.insertInto('user')
      .values(data)
      .returningAll()
      .executeTakeFirstOrThrow()
  } catch (error: unknown) {
    if (isUniqueViolation(error)) {
      throw new ConflictError('Email already exists')
    }
    if (isForeignKeyViolation(error)) {
      throw new BadRequestError('Referenced record does not exist')
    }
    throw error
  }
}

// Helper to check PostgreSQL error codes
function isUniqueViolation(error: unknown): boolean {
  return (
    error instanceof Error &&
    'code' in error &&
    (error as any).code === '23505'
  )
}

function isForeignKeyViolation(error: unknown): boolean {
  return (
    error instanceof Error &&
    'code' in error &&
    (error as any).code === '23503'
  )
}
```

## Common PostgreSQL Error Codes

| Code    | Constraint          | Meaning                          |
|---------|---------------------|----------------------------------|
| `23505` | unique_violation    | Duplicate key value              |
| `23503` | foreign_key_violation | Referenced row does not exist  |
| `23502` | not_null_violation  | NULL in a NOT NULL column        |
| `23514` | check_violation     | CHECK constraint failed          |
| `23P01` | exclusion_violation | Exclusion constraint violated    |
| `40001` | serialization_failure | Serializable transaction conflict |
| `40P01` | deadlock_detected   | Deadlock between transactions    |

## Connection Errors

Handle connection failures separately from query errors:

```typescript
async function queryWithRetry<T>(
  fn: () => Promise<T>,
  retries = 3,
): Promise<T> {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fn()
    } catch (error: unknown) {
      const isConnectionError =
        error instanceof Error &&
        ('code' in error && (error as any).code === 'ECONNREFUSED' ||
         error.message.includes('Connection terminated'))

      if (isConnectionError && attempt < retries) {
        await new Promise((r) => setTimeout(r, 100 * attempt))
        continue
      }
      throw error
    }
  }
  throw new Error('Unreachable')
}
```

## Wrapping Errors with Context

Add meaningful context when re-throwing database errors:

```typescript
async function transferFunds(fromId: number, toId: number, amount: number) {
  try {
    return await db.transaction().execute(async (trx) => {
      // ... transfer logic
    })
  } catch (error) {
    throw new Error(
      `Failed to transfer ${amount} from account ${fromId} to ${toId}`,
      { cause: error }
    )
  }
}
```

## Pitfalls

- **Kysely itself does not define database error types** — constraint errors come from the underlying driver (`pg`).
- **Always rethrow unknown errors** — only catch errors you can handle meaningfully.
- **`executeTakeFirstOrThrow()` is cleaner than checking `undefined`** for queries that must return a result.
- **Connection errors are transient** — retry logic is appropriate; constraint violations are not — do not retry those.
