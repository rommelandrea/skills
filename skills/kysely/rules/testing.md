---
name: testing
description: Integration testing, transaction-based isolation, and mocking Kysely
metadata:
  tags: kysely, testing, integration, transaction, mock
---

# Testing

## Integration Tests with a Real Database

Integration tests against a real PostgreSQL database catch issues that mocks miss (types, constraints, query semantics):

```typescript
import { describe, it, beforeAll, afterAll, beforeEach, afterEach } from 'node:test'
import assert from 'node:assert/strict'
import { Kysely, PostgresDialect } from 'kysely'
import { Pool } from 'pg'
import type { Database } from '../src/db/types.js'

let db: Kysely<Database>

beforeAll(async () => {
  db = new Kysely<Database>({
    dialect: new PostgresDialect({
      pool: new Pool({
        connectionString: process.env.TEST_DATABASE_URL,
      }),
    }),
  })
})

afterAll(async () => {
  await db.destroy()
})

describe('UserRepository', () => {
  it('should create a user', async () => {
    const user = await db.insertInto('user')
      .values({ email: 'test@example.com', name: 'Test' })
      .returningAll()
      .executeTakeFirstOrThrow()

    assert.equal(user.email, 'test@example.com')
    assert.ok(user.id)
  })
})
```

## Transaction-Based Test Isolation

Wrap each test in a transaction and roll back after to keep the database clean:

```typescript
import { describe, it, beforeAll, afterAll, beforeEach, afterEach } from 'node:test'
import assert from 'node:assert/strict'
import { Kysely, PostgresDialect, Transaction } from 'kysely'
import { Pool } from 'pg'
import type { Database } from '../src/db/types.js'

let db: Kysely<Database>
let trx: Transaction<Database>

beforeAll(async () => {
  db = new Kysely<Database>({
    dialect: new PostgresDialect({
      pool: new Pool({ connectionString: process.env.TEST_DATABASE_URL }),
    }),
  })
})

afterAll(async () => {
  await db.destroy()
})

// Each test runs in a transaction that gets rolled back
beforeEach(async () => {
  // Start a transaction — pass the trx to code under test
  trx = await startTransaction(db)
})

afterEach(async () => {
  // Roll back all changes made during the test
  await rollbackTransaction(trx)
})

// Helper using internal Kysely APIs
async function startTransaction(db: Kysely<Database>): Promise<Transaction<Database>> {
  // Use db.connection() to get a dedicated connection, then BEGIN
  // Implementation depends on your test setup
}
```

A simpler approach — use a test database that gets truncated between suites:

```typescript
afterEach(async () => {
  await db.deleteFrom('comment').execute()
  await db.deleteFrom('post').execute()
  await db.deleteFrom('user').execute()
})
```

## Testing Query Construction

Verify query SQL without executing, using `compile()`:

```typescript
import { describe, it } from 'node:test'
import assert from 'node:assert/strict'

describe('query building', () => {
  it('should build correct SQL', () => {
    const query = db.selectFrom('user')
      .select(['id', 'email'])
      .where('active', '=', true)
      .compile()

    assert.equal(
      query.sql,
      'select "id", "email" from "user" where "active" = $1'
    )
    assert.deepEqual(query.parameters, [true])
  })
})
```

## Testing with Dependency Injection

Pass `Kysely<Database>` or `Transaction<Database>` into functions for testability:

```typescript
// src/repositories/user.ts
import type { Kysely, Transaction } from 'kysely'
import type { Database } from '../db/types.js'

type DB = Kysely<Database> | Transaction<Database>

export function createUserRepository(db: DB) {
  return {
    async findByEmail(email: string) {
      return db.selectFrom('user')
        .selectAll()
        .where('email', '=', email)
        .executeTakeFirst()
    },

    async create(data: { email: string; name: string }) {
      return db.insertInto('user')
        .values(data)
        .returningAll()
        .executeTakeFirstOrThrow()
    },
  }
}

// In tests — pass the transaction
it('should find user by email', async () => {
  const repo = createUserRepository(trx)

  await repo.create({ email: 'a@b.com', name: 'Alice' })
  const user = await repo.findByEmail('a@b.com')

  assert.equal(user?.name, 'Alice')
})
```

## Pitfalls

- **Use a dedicated test database** — never run tests against production or development databases.
- **Truncation order matters** — delete child tables before parent tables to avoid foreign key violations.
- **`compile()` is dialect-specific** — the SQL output varies between PostgreSQL, MySQL, and SQLite.
- **Transaction rollback is faster than truncation** — prefer transaction-based isolation when possible.
