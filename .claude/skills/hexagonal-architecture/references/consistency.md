# Transactions, Events & Consistency

## The Transaction Boundary Is the Use Case

Node has no AOP proxies, so the boundary is **explicit** through an output port:

```typescript
// account/application/service/send-money.service.ts
async sendMoney(command: SendMoneyCommand): Promise<boolean> {
  this.checkThreshold(command);
  const transferred = await this.transaction.run(() => this.transfer(command)); // BEGIN … COMMIT
  if (transferred) {
    await this.publishAccountEventPort.publish(new MoneyTransferred(/* … */));  // after COMMIT
  }
  return transferred;
}
```

`ClsTransactionAdapter` (see `shared-kernel.md`) implements `TransactionPort` with `@nestjs-cls/transactional`; persistence adapters reading `txHost.tx` join the transaction automatically.

**Why a port instead of the `@Transactional()` decorator:**
- **Unit tests stay framework-free** - the decorator resolves a global `TransactionHost` at call time, so a service built with plain `new` would need the module mocked. With the port, tests pass a trivial fake (`FakeTransaction`).
- **The boundary is visible** - you can see exactly what runs inside the transaction and what runs after commit.

`@Transactional()` is an acceptable alternative if a team prefers it: use it consistently and remember it wraps the method itself, so it also applies to `this.method()` self-calls.

**Rules**
- One use case = one unit of work; domain services are pure and never open transactions
- No HTTP calls, message publishing or file I/O inside `TransactionPort.run` (holds locks and a pool connection)
- Never use `@InjectRepository()` repositories in persistence adapters - they run outside the transaction

## Publish After Commit

Event subscribers run detached from the publisher but **inherit its async context**. A subscriber started inside the transaction could read uncommitted data or use an `EntityManager` whose query runner was already released.

- Publish integration events **after** `TransactionPort.run` resolves (as above)
- In production, use a **transactional outbox**: "commit, then publish" loses the event if the process dies between the two steps

```typescript
// Outbox: write the event in the same transaction…
await this.transaction.run(async () => {
  await this.updateAccountStatePort.updateBalance(source);
  await this.updateAccountStatePort.updateBalance(target);
  await this.appendOutboxPort.append(new MoneyTransferred(/* … */)); // same transaction as the state change
});
// …and let an adapter (scheduler or broker connector) relay pending rows and mark them sent.
// Consumers must be idempotent: the relay delivers at least once.
```

## Concurrency in Node

"Single-threaded" does not mean race-free: requests interleave at every `await`, so two concurrent withdrawals can both read the old balance.

| Approach | How | Use when |
|---|---|---|
| Pessimistic lock | `LockAccountPort` → `findOne({ where, lock: { mode: 'pessimistic_write' } })` inside the transaction | Hot rows, money movements |
| Optimistic lock | `@VersionColumn` + conditional `UPDATE … WHERE version = :expected`, fail with a `conflict` `DomainException` | Low contention, user edits |
| Atomic update | `UPDATE … SET balance = balance - :x WHERE balance >= :x` in the adapter | Simple counters |

- Lock rows in a **stable order** (e.g. by id) to avoid deadlocks between opposite operations
- Row locks are released by COMMIT/ROLLBACK, so the port needs no release method. A **distributed lock** (Redis, etcd) is different: model the port as `lockAccount()` / `releaseAccount()` and release in nested `try/finally` blocks so a failed write never leaves an account locked
- An in-process mutex is not enough once you run more than one instance
- A no-op lock adapter is acceptable only in prototypes - write it down as a shortcut

## Idempotency

Retried HTTP requests and redelivered messages must not apply a use case twice:
- Accept an idempotency key in the request (header) and pass it in the command
- Store processed keys through an output port inside the same transaction as the state change
- Event listeners deduplicate by event id before executing the command
