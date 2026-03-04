---
name: transactions
description: Transaction patterns, isolation levels, and error recovery
metadata:
  tags: kysely, transaction, isolation, rollback, atomic
---

# Transactions

## Basic Transaction

Use `db.transaction().execute()` to run multiple queries atomically. The transaction automatically commits on success and rolls back on error:

```typescript
const result = await db.transaction().execute(async (trx) => {
  const user = await trx.insertInto('user')
    .values({ email: 'alice@example.com', name: 'Alice' })
    .returningAll()
    .executeTakeFirstOrThrow()

  await trx.insertInto('profile')
    .values({ user_id: user.id, bio: '' })
    .execute()

  return user
})
```

## Automatic Rollback on Error

If any query inside the transaction throws, the entire transaction is rolled back:

```typescript
try {
  await db.transaction().execute(async (trx) => {
    await trx.updateTable('account')
      .set((eb) => ({ balance: eb('balance', '-', amount) }))
      .where('id', '=', fromId)
      .execute()

    await trx.updateTable('account')
      .set((eb) => ({ balance: eb('balance', '+', amount) }))
      .where('id', '=', toId)
      .execute()

    // If this throws, both updates are rolled back
    await validateTransfer(trx, fromId, toId, amount)
  })
} catch (error) {
  // Transaction already rolled back
  console.error('Transfer failed:', error)
}
```

## Isolation Levels

Set the isolation level on the transaction builder:

```typescript
await db.transaction()
  .setIsolationLevel('serializable')
  .execute(async (trx) => {
    // Serializable isolation — strongest consistency
    const balance = await trx.selectFrom('account')
      .select('balance')
      .where('id', '=', accountId)
      .executeTakeFirstOrThrow()

    await trx.updateTable('account')
      .set({ balance: balance.balance - amount })
      .where('id', '=', accountId)
      .execute()
  })
```

Available isolation levels:
- `'read uncommitted'`
- `'read committed'` (PostgreSQL default)
- `'repeatable read'`
- `'serializable'`

## Pass Transaction to Helper Functions

Always pass the `trx` parameter to functions that run inside a transaction. Never use the global `db` inside a transaction callback:

```typescript
// Good — accepts Kysely or Transaction
import type { Kysely, Transaction } from 'kysely'

async function createOrder(
  trx: Kysely<Database> | Transaction<Database>,
  data: { userId: number; items: Array<{ productId: number; quantity: number }> }
) {
  const order = await trx.insertInto('order')
    .values({ user_id: data.userId, status: 'pending' })
    .returningAll()
    .executeTakeFirstOrThrow()

  await trx.insertInto('order_item')
    .values(
      data.items.map((item) => ({
        order_id: order.id,
        product_id: item.productId,
        quantity: item.quantity,
      }))
    )
    .execute()

  return order
}

// Usage
await db.transaction().execute(async (trx) => {
  const order = await createOrder(trx, {
    userId: 1,
    items: [{ productId: 10, quantity: 2 }],
  })
  await sendConfirmation(order)
})
```

## Nested Transactions with Savepoints

Kysely does not natively support savepoints. Use raw SQL if you need nested rollback points:

```typescript
import { sql } from 'kysely'

await db.transaction().execute(async (trx) => {
  await trx.insertInto('audit_log')
    .values({ action: 'start' })
    .execute()

  try {
    await sql`SAVEPOINT sp1`.execute(trx)

    await trx.insertInto('risky_table')
      .values({ data: 'maybe fails' })
      .execute()
  } catch {
    await sql`ROLLBACK TO SAVEPOINT sp1`.execute(trx)
    // Continue with the rest of the transaction
  }

  await trx.insertInto('audit_log')
    .values({ action: 'end' })
    .execute()
})
```

## Pitfalls

- **Never use the global `db` inside `transaction().execute()`** — queries on `db` run outside the transaction, defeating atomicity.
- **Do not catch and swallow errors inside the callback** if you want the transaction to roll back — rethrow or let them propagate.
- **Long-running transactions hold locks** — keep transaction bodies short and fast.
- **Serializable transactions can fail with serialization errors** — retry on `40001` (serialization failure) error code.
