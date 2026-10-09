---
name: typeorm-patterns
description: TypeORM patterns and common pitfalls in NestJS (N+1, eager/lazy relations, transactions, locking, pagination, bulk writes). Use when user has TypeORM performance issues, too many queries, transaction bugs, or asks about entity relationships and loading strategies.
---

# TypeORM Patterns Skill

Best practices and common pitfalls for TypeORM 1.x in NestJS applications.

## When to Use
- User mentions "N+1 problem" / "too many queries" / slow endpoints
- Relations unexpectedly `undefined` or loaded everywhere
- Writes not rolled back, partial data after errors
- Lost updates under concurrency
- Pagination returning wrong counts or duplicates
- Entity relationship design and query optimization

---

## Quick Reference: Common Problems

| Problem | Symptom | Solution |
|---------|---------|----------|
| N+1 queries | Query per row in a loop | `relations`, `leftJoinAndSelect`, batch `In()` |
| Relation is `undefined` | Not loaded by default | Load explicitly with `relations` / joins |
| `eager: true` everywhere | Huge queries on every `find` | Remove `eager`, load per use case |
| Write escapes transaction | Partial data after rollback | Use the transaction's `manager` for every call |
| Lost updates | Concurrent writes overwrite each other | Conditional update on `@VersionColumn` |
| Cartesian explosion | Slow join of several one-to-many relations | `relationLoadStrategy: 'query'` or separate queries |
| Slow bulk writes | `save()` in loops | `insert()`, `upsert()`, `update()` |
| Unbounded results | Memory spikes | Always `take`, paginate |

---

## N+1 Problem

> The #1 TypeORM performance killer

### The Problem

```typescript
// ❌ BAD: N+1 queries
const authors = await this.authors.find();                          // 1 query
for (const author of authors) {
  author.books = await this.books.findBy({ author: { id: author.id } }); // N queries!
}
// 100 authors = 101 queries
```

### Solution 1: Load the Relation in the Same Query

```typescript
// ✅ GOOD: single query with LEFT JOIN
const authors = await this.authors.find({ relations: { books: true } });

// ✅ GOOD: QueryBuilder when you need filters on the relation
const authors = await this.authors
  .createQueryBuilder('a')
  .leftJoinAndSelect('a.books', 'b', 'b.publishedAt IS NOT NULL')
  .where('a.country = :country', { country })
  .getMany();
```

### Solution 2: Batch with `In()`

```typescript
// ✅ GOOD: 2 queries regardless of N
const authors = await this.authors.find({ take: 50 });
const books = await this.books.find({
  where: { author: { id: In(authors.map((a) => a.id)) } },
  relations: { author: true },
});
const byAuthor = new Map<number, Book[]>();
for (const book of books) {
  byAuthor.set(book.author.id, [...(byAuthor.get(book.author.id) ?? []), book]);
}
```

For GraphQL resolvers, use a per-request DataLoader to apply the same batching automatically.

### Detecting N+1

```typescript
TypeOrmModule.forRoot({
  // ...
  logging: ['query'],          // every SQL statement (development only)
  maxQueryExecutionTime: 200,  // logs slow queries in all environments
});
```

In e2e tests, count queries with a custom TypeORM `Logger` and assert the number per request.

---

## Relation Loading

### Eager vs. Explicit

```typescript
@Entity()
export class Order {
  // ⚠️ eager: loaded on EVERY find() of Order, recursively - avoid
  @ManyToOne(() => Customer, { eager: true })
  customer!: Relation<Customer>;

  // ✅ default: not loaded until you ask for it
  @OneToMany(() => OrderLine, (line) => line.order)
  lines!: Relation<OrderLine[]>;
}
```

### Best Practice: Load Per Use Case

```typescript
// ✅ GOOD: each query loads exactly what the use case needs
findForInvoice(id: string) {
  return this.orders.findOne({ where: { id }, relations: { customer: true, lines: true } });
}

findForList() {
  return this.orders.find({ select: { id: true, status: true, totalCents: true }, take: 50 });
}
```

### Avoid Lazy Relations (`Promise<T>` properties)

```typescript
// ❌ BAD: lazy relation - every access is a hidden query, easy N+1, awkward serialization
@OneToMany(() => OrderLine, (line) => line.order, { lazy: true })
lines!: Promise<OrderLine[]>;
```

### Cartesian Explosion

Joining two one-to-many relations multiplies rows (`orders x lines x payments`).

```typescript
// ✅ GOOD: one query per relation instead of one giant join
this.orders.find({
  where: { id },
  relations: { lines: true, payments: true },
  relationLoadStrategy: 'query',
});
```

---

## Transactions

### Repository Calls Are Separate Transactions

```typescript
// ❌ BAD: two independent auto-commits - a failure leaves an order without lines
async create(dto: CreateOrderDto) {
  const order = await this.orders.save(this.orders.create(dto));
  await this.lines.save(dto.lines.map((l) => this.lines.create({ ...l, order })));
}
```

### Use the Transactional EntityManager

```typescript
// ✅ GOOD: atomic
async create(dto: CreateOrderDto) {
  return this.dataSource.transaction(async (manager) => {
    const order = await manager.save(manager.create(Order, dto));
    await manager.save(dto.lines.map((l) => manager.create(OrderLine, { ...l, order })));
    return order;
  });
}
```

### The Classic Bug: Injected Repository Inside a Transaction

```typescript
// ❌ BAD: this.orders is NOT bound to the transaction
await this.dataSource.transaction(async (manager) => {
  await manager.save(payment);
  await this.orders.update(id, { status: OrderStatus.PAID }); // commits independently!
});

// ✅ GOOD: derive repositories from the manager
await this.dataSource.transaction(async (manager) => {
  await manager.save(payment);
  await manager.getRepository(Order).update(id, { status: OrderStatus.PAID });
});
```

### Keep Transactions Short

- No HTTP calls, message publishing, or file I/O inside a transaction (holds locks and a pool connection)
- Publish events after commit, or write them to an outbox table inside the transaction
- Pass the `EntityManager` down to helpers that must join the transaction

---

## Concurrency

### Optimistic Locking

`@VersionColumn` increments on `save()`, but `save()` does **not** reject stale versions. Enforce it with a conditional update:

```typescript
async rename(id: string, name: string, expectedVersion: number): Promise<void> {
  const result = await this.products
    .createQueryBuilder()
    .update(Product)
    .set({ name, version: () => 'version + 1' })
    .where('id = :id AND version = :expectedVersion', { id, expectedVersion })
    .execute();

  if (result.affected === 0) {
    throw new ConflictException('Product was modified by someone else, reload and retry');
  }
}
```

Expose `version` in the response DTO (or an `ETag`) and require it on updates (`If-Match`).

### Pessimistic Locking

```typescript
await this.dataSource.transaction(async (manager) => {
  const account = await manager
    .getRepository(Account)
    .createQueryBuilder('a')
    .setLock('pessimistic_write')   // SELECT ... FOR UPDATE
    .where('a.id = :id', { id })
    .getOneOrFail();
  account.balanceCents -= amountCents;
  await manager.save(account);
});
```

### Atomic Updates Beat Read-Modify-Write

```typescript
// ✅ GOOD: no race, no lock held in application code
await this.products.decrement({ id }, 'stock', quantity);
```

---

## Pagination

```typescript
// ✅ Offset pagination with deterministic ordering
const [items, total] = await this.orders.findAndCount({
  where,
  order: { createdAt: 'DESC', id: 'DESC' }, // tie-breaker avoids duplicates/skips
  skip: page * size,
  take: size,
});
```

With joins in QueryBuilder, use `.skip()`/`.take()` (entity-aware), not `.offset()`/`.limit()` (row-based, cut through joined rows).

```typescript
// ✅ Keyset pagination for large tables / infinite scroll
const items = await this.orders
  .createQueryBuilder('o')
  .where('(o.createdAt, o.id) < (:createdAt, :id)', { createdAt: cursor.createdAt, id: cursor.id })
  .orderBy('o.createdAt', 'DESC')
  .addOrderBy('o.id', 'DESC')
  .take(size)
  .getMany();
```

---

## Bulk Writes

```typescript
// ❌ BAD: save() does a SELECT + INSERT/UPDATE per entity
for (const row of rows) await this.products.save(row);

// ✅ GOOD
await this.products.insert(rows);                                // one INSERT
await this.products.upsert(rows, { conflictPaths: ['sku'] });    // INSERT ... ON CONFLICT DO UPDATE
await this.products.update({ categoryId }, { active: false });   // one UPDATE
```

For very large imports, chunk the input (e.g. 1 000 rows per statement) to stay under parameter limits.

---

## Other Pitfalls

| Pitfall | Fix |
|---------|-----|
| `where: { field: undefined }` | TypeORM 1.x throws (0.3 matched ALL rows). Build `where` conditionally; use `IsNull()` |
| `bigint`/`numeric` returned as string | Column `transformer`, or integer minor units for money |
| `timestamp` without zone | Use `timestamptz` |
| `synchronize: true` in production | Migrations only |
| Raw SQL with string interpolation | Parameters: `:name` in QueryBuilder, `$1` in `query()` |
| Missing FK indexes | `@Index()` on relation columns used in joins/filters |
| Entities returned from controllers | Map to response DTOs |
| ESM circular imports between entities | `Relation<T>` property types + arrow-function relation targets |

---

## Quick Reference

| Need | Use |
|------|-----|
| Load relation | `relations: { lines: true }` |
| Filtered join | `leftJoinAndSelect('a.books', 'b', 'condition')` |
| Many relations | `relationLoadStrategy: 'query'` |
| Atomic writes | `dataSource.transaction(async (manager) => ...)` |
| Optimistic lock | `@VersionColumn` + conditional `UPDATE ... WHERE version = :v` |
| Pessimistic lock | `.setLock('pessimistic_write')` inside a transaction |
| Counters | `increment()` / `decrement()` |
| Bulk | `insert()`, `upsert()`, `update()` |
| Slow query log | `maxQueryExecutionTime` |
