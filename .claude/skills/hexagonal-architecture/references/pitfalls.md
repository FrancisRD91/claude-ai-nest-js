# JavaScript, TypeScript & Node.js Pitfalls

Behaviours of the language and runtime that quietly break hexagonal NestJS code - why each one happens and how to address it.

| # | Pitfall | Address it with |
|---|---|---|
| 1 | Interfaces vanish at runtime | Abstract classes as ports |
| 2 | `import type` vs decorator metadata | Value imports for injected classes; `import type` for pure types |
| 3 | No decorator, no metadata | `useFactory` for undecorated classes |
| 4 | Two `useClass` entries, two instances | `useExisting` aliases |
| 5 | `Scope.REQUEST` spreads | Singletons + `nestjs-cls` |
| 6 | Circular imports load as `undefined` | One concept per file, no barrels, `no-circular` |
| 7 | Floating promises | `await`, or `void` + `.catch()`; lint rule |
| 8 | Single-threaded is not race-free | Row locks / optimistic versioning |
| 9 | Transactions are not ambient | `txHost.tx` in every persistence adapter |
| 10 | Async context follows what you start | Publish after commit, outbox |
| 11 | Event handlers are fire-and-forget | Catch in handlers; poll in tests |
| 12 | Cleanup skipped on errors | `try/finally` |
| 13 | `number` loses precision; `bigint` breaks JSON | `bigint` in the core, decimal strings on the wire |
| 14 | `JSON.stringify` surprises | Explicit response classes |
| 15 | Structural typing | Branded ids |
| 16 | `===` compares references | `equals()`, primitive map keys |
| 17 | Mutable `Date`, time zones | Defensive copies, UTC |
| 18 | `readonly` is compile-time only | Copies at the boundary |
| 19 | Class fields reset after `super()` | Initialise in the owning class |
| 20 | Everything from outside is a string | Parse and validate at the boundary |
| 21 | Native modules are per OS | `npm ci` in every environment |

---

## Dependency Injection & Decorators

### 1. Interfaces vanish at runtime

TypeScript `interface`s are erased during compilation, so Nest has no runtime value to use as a DI token. `@Inject()` with an interface type silently resolves nothing.

```typescript
// ❌ Compiles to nothing - cannot be injected
export interface LoadAccountPort { loadAccount(id: AccountId): Promise<Account | null>; }

// ✅ An abstract class is a type AND a runtime token
export abstract class LoadAccountPort {
  abstract loadAccount(id: AccountId): Promise<Account | null>;
}

@Injectable()
export class AccountPersistenceAdapter implements LoadAccountPort { /* … */ } // implements, not extends
```

Use `Symbol` tokens with `@Inject(TOKEN)` only for non-class values (config objects, third-party clients).

### 2. `import type` and decorator metadata

With `emitDecoratorMetadata`, the compiler writes each constructor parameter's runtime type into `design:paramtypes`. An `import type` has no runtime value, so the metadata becomes `Object` and Nest fails with "can't resolve dependencies".

The reverse also applies: with `isolatedModules`, a **pure type** (interface, type alias such as `ConfigType<…>`) used in a decorated constructor **must** be imported with `import type`, or `tsc` fails with TS1272.

```typescript
// ❌ DI token erased
import type { LoadAccountPort } from '../port/out/load-account.port.js';

// ✅ Injected classes: value import. Pure types: import type.
import { LoadAccountPort } from '../port/out/load-account.port.js';
import type { ConfigType } from '@nestjs/config';

constructor(
  private readonly loadAccountPort: LoadAccountPort,
  @Inject(accountConfig.KEY) private readonly config: ConfigType<typeof accountConfig>,
) {}
```

Never enable an autofixing `consistent-type-imports` lint rule in a Nest code base.

### 3. No decorator, no metadata

TypeScript emits `design:paramtypes` only for classes that carry a decorator. Domain classes must stay decorator-free, so Nest cannot auto-wire their constructor dependencies.

```typescript
// ✅ Build framework-free objects explicitly in the module
{
  provide: TransferFeePolicy,                        // plain domain service, no @Injectable()
  inject: [accountConfig.KEY],
  useFactory: (config: ConfigType<typeof accountConfig>) => new TransferFeePolicy(Money.of(config.feePerTransfer)),
},
```

### 4. Two `useClass` entries create two instances

Each `useClass` provider is instantiated separately. An adapter that serves two ports would exist twice, and any state, cache or connection inside it splits.

```typescript
// ❌ Two AccountPersistenceAdapter instances
{ provide: LoadAccountPort, useClass: AccountPersistenceAdapter },
{ provide: UpdateAccountStatePort, useClass: AccountPersistenceAdapter },

// ✅ One instance, two tokens
AccountPersistenceAdapter,
{ provide: LoadAccountPort, useExisting: AccountPersistenceAdapter },
{ provide: UpdateAccountStatePort, useExisting: AccountPersistenceAdapter },
```

### 5. `Scope.REQUEST` spreads

A request-scoped provider makes every provider that depends on it request-scoped too, and Nest rebuilds that whole chain for each request - slow, and it breaks singletons such as caches.

Keep the default singleton scope. Put per-request data (correlation id, current user) in `nestjs-cls`, which stores it in `AsyncLocalStorage`.

---

## Modules & Loading

### 6. Circular imports load as `undefined`

When two files import each other, one side runs before the other has finished loading. Under CommonJS it sees `undefined`; under ESM it throws `Cannot access 'X' before initialization`. Nest usually reports it as `Nest can't resolve dependencies of X (?)`.

- One concept per file; never declare a second class "for convenience" in the same file
- No barrel files (`index.ts`) inside a context - they are the most common source of cycles (the public `api/index.ts` is the only exception)
- Don't paper over cycles with `forwardRef()` - fix the design (an event, or a concept moved to the right place)
- Keep the dependency-cruiser `no-circular` rule on

---

## Asynchrony

### 7. Floating promises

A promise that nobody awaits runs detached: its rejection becomes an unhandled rejection (which can crash the process), and the caller continues before the work is done - for example, a response is sent before the write commits.

```typescript
// ❌ Detached work, errors lost
this.deliverNotificationPort.deliver(notification);
items.forEach(async (item) => await this.save(item)); // forEach never awaits

// ✅ Await it
await this.deliverNotificationPort.deliver(notification);
for (const item of items) await this.save(item);

// ✅ Intentional fire-and-forget: explicit and handled
void this.audit.record(entry).catch((err) => this.logger.error('Audit failed', err instanceof Error ? err.stack : String(err)));
```

Enable `@typescript-eslint/no-floating-promises` as an error.

### 8. Single-threaded is not race-free

Node runs one callback at a time, but two requests interleave at every `await`. "Load balance → check → write" lets two concurrent withdrawals both read the old balance.

- Lock rows inside the transaction (`lock: { mode: 'pessimistic_write' }`) in a stable order, or use optimistic versioning
- An in-process mutex stops working as soon as a second instance runs
- See `consistency.md`

### 9. Transactions are not ambient

Nothing joins a transaction automatically. Only code that reads the transactional `EntityManager` from the async context takes part; a repository injected with `@InjectRepository()` uses the default connection and commits on its own.

```typescript
// ❌ Runs outside the use case's transaction
constructor(@InjectRepository(AccountOrmEntity) private readonly accounts: Repository<AccountOrmEntity>) {}

// ✅ Joins whatever transaction TransactionPort.run() opened
constructor(private readonly txHost: TransactionHost<TransactionalAdapterTypeOrm>) {}
private get entityManager(): EntityManager { return this.txHost.tx; }
```

Cover it with a rollback test: write inside `TransactionPort.run()`, throw, assert nothing was persisted.

### 10. Async context follows whatever you start

`AsyncLocalStorage` propagates into every callback, timer and event handler started from the current context. An event published inside a transaction runs its subscribers "inside" it: they can read uncommitted data or use an `EntityManager` whose query runner has already been released.

```typescript
// ✅ Publish only after the transaction has resolved
const transferred = await this.transaction.run(() => this.transfer(command));
if (transferred) await this.publishAccountEventPort.publish(event);
```

For guaranteed delivery, write the event to an outbox table inside the transaction instead.

### 11. Event handlers are fire-and-forget

`EventBus.publish()` returns immediately: it doesn't wait for subscribers, and their errors never reach the publisher (`@nestjs/cqrs` only logs them and emits them on the `UnhandledExceptionBus`).

- Every event handler catches its own errors and decides: log with context, retry, or dead-letter
- Tests observe the effect by polling with `vi.waitFor()`, never by asserting right after the HTTP response or by sleeping

### 12. Cleanup is skipped when something throws

Code after an `await` that rejects never runs. A lock released, a query runner released, or a file closed "at the end" stays held forever after the first failure.

```typescript
// ❌ Lock leaks when updateActivities() throws
await this.lock.lockAccount(id);
await this.updateAccountStatePort.updateActivities(account);
await this.lock.releaseAccount(id);

// ✅ Always release
await this.lock.lockAccount(id);
try {
  await this.updateAccountStatePort.updateActivities(account);
} finally {
  await this.lock.releaseAccount(id);
}
```

Unit-test the failure path: make the port reject and assert the release still happened.

---

## Numbers, Values & Data

### 13. `number` loses precision; `bigint` breaks JSON

`number` is an IEEE-754 double, exact only up to 2^53 − 1 (`Number.MAX_SAFE_INTEGER`); beyond that, integers round silently. `bigint` is exact, but `JSON.stringify({ a: 1n })` throws `TypeError: Do not know how to serialize a BigInt`, and database drivers return 64-bit integers as `string` (pg) or `number` (sqlite).

- Money and large integers are `bigint` in the core (wrapped in a `Money` value object); decimals become integer minor units
- They travel as **decimal strings** in JSON, in both directions (`"amount": "500"`), validated with `@IsString()` + `@Matches(/^\d{1,30}$/)`
- Never parse money with `ParseIntPipe` or put it in the URL
- Normalise driver output with a `bigint` column transformer in the persistence adapter
- `BigInt(1.5)` throws `RangeError` - good: fractional input is rejected instead of rounded

### 14. `JSON.stringify` surprises

Serialisation is not a faithful copy of an object: `Date` becomes an ISO string, `Map` and `Set` become `{}`, `undefined` properties disappear, methods and private state are dropped or exposed as-is, and `bigint` throws.

Never return domain objects or ORM entities from controllers. Map them to explicit response classes with JSON-safe fields (strings for money and dates).

### 15. Structural typing

TypeScript compares types by shape. `class AccountId { constructor(readonly value: number) {} }` and an `ActivityId` with the same shape are interchangeable, so passing the wrong id compiles.

```typescript
export class AccountId {
  private readonly brand = 'AccountId' as const; // makes the shape unique

  constructor(readonly value: number) {
    if (!Number.isSafeInteger(value) || value <= 0) throw new InvalidInputException(`Account id must be a positive integer, got ${value}`);
  }

  equals(other: AccountId): boolean { return this.value === other.value; }
}
```

### 16. `===` compares references

Two value objects with the same content are different objects, so `===`, `includes()`, `indexOf()`, `Set` and `Map` keys treat them as different.

- Give value objects an `equals()` method and use it
- Key maps and sets by primitive values (`accountId.value`), not by value objects
- In tests, `toEqual()` compares structure; `toBe()` compares identity

### 17. Mutable `Date` and time zones

`Date` objects can be changed by anyone holding a reference (`setHours()` on a shared instance changes it everywhere), and they represent UTC instants that are formatted in the process's local time zone.

- Store a defensive copy in constructors: `this.timestamp = new Date(timestamp.getTime())`
- Treat every `Date` as a UTC instant; send ISO-8601 strings in JSON; use `timestamptz` columns
- Some databases (e.g. SQLite) store dates as text - compare like with like in queries

### 18. `readonly` is compile-time only

`readonly` fields and `readonly T[]` types disappear after compilation; a caller can still cast and mutate.

```typescript
export class ActivityWindow {
  private readonly activities: Activity[];

  constructor(activities: readonly Activity[] = []) {
    this.activities = [...activities];   // copy in
  }

  getActivities(): readonly Activity[] {
    return [...this.activities];         // copy out
  }
}
```

### 19. Class fields are reset after `super()`

With `target` ES2022 or later, class fields use define semantics: fields declared in a subclass are initialised **after** the base constructor returns. A field the subclass redeclares without an initialiser is defined as `undefined`, overwriting whatever the base constructor assigned to it.

Initialise fields in the class that owns them and pass shared values through the constructor:

```typescript
export abstract class DomainException extends Error {
  protected constructor(readonly kind: DomainExceptionKind, readonly code: string, message: string) {
    super(message);
    this.name = new.target.name;
  }
}

export class AccountNotFoundException extends DomainException {
  constructor(readonly accountId: AccountId) {
    super('not-found', 'ACCOUNT_NOT_FOUND', `Account ${accountId.value} not found`);
  }
}
```

---

## Runtime & Environment

### 20. Everything from outside is a string

`process.env.X` is `string | undefined`; route params and query strings are strings until a pipe converts them; JSON numbers are doubles.

- Parse and validate environment variables once at startup (env schema in `src/config`) and fail fast; the core never reads `process.env`
- Convert params with pipes (`ParseIntPipe`, `ParseUUIDPipe`) and validate bodies with DTOs (`whitelist`, `forbidNonWhitelisted`, `transform`)
- Re-check semantics in the input model constructor (`SendMoneyCommand`) - syntax checks in the adapter are not enough

### 21. Native modules are built per OS and architecture

Packages with native add-ons (`@swc/core`, `better-sqlite3`, `argon2`, `bcrypt`) ship binaries for one platform. A `node_modules` folder installed on Windows does not run in a Linux container or CI runner.

- Run `npm ci` inside every environment (local, CI, Docker build stage); never copy `node_modules` between platforms
- Exclude `node_modules` in `.dockerignore`
- Prefer pure-JavaScript or WebAssembly alternatives when a native dependency adds nothing essential
