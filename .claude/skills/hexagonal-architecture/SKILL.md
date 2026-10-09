---
name: hexagonal-architecture
description: Hexagonal (Ports & Adapters) architecture for NestJS - bounded contexts as modules, a pure domain, abstract-class ports, use cases (direct ports or @nestjs/cqrs), thin adapters, explicit transactions and machine-checked dependency rules. Use when designing, building or reviewing a NestJS service with real business rules, adding a use case or bounded context, or when the user mentions hexagonal, ports and adapters, clean architecture, DDD or bounded contexts.
metadata:
  version: "1.0.0"
  domain: architecture
  triggers: hexagonal architecture, ports and adapters, clean architecture, bounded context, use case, DDD, anti-corruption layer
  role: architect
  scope: design-and-implementation
  output-format: code
---

# Hexagonal Architecture Skill

How to build NestJS services with Ports & Adapters. This skill sets the **structure**; the `nestjs` skill sets the stack (NestJS 12, ESM `.js` imports, Vitest, TypeORM 1.x) and `typeorm-patterns` covers persistence pitfalls.

Naming follows *buckpal* ("Get Your Hands Dirty on Clean Architecture"). Examples use two contexts: `account` (direct use-case ports) and `notification` (`@nestjs/cqrs` buses).

## When to Use

- The service has business rules worth protecting (invariants, calculations, workflows)
- The same use cases are driven by more than one entry point (HTTP, events, jobs, other contexts)
- The codebase holds several bounded contexts, or will be long-lived
- The user asks for hexagonal / ports & adapters / clean architecture

**Don't use it** for a thin CRUD or proxy service: a plain Nest module (controller → service → repository, as in the `nestjs` skill) is fine. Write that decision down (README or module JSDoc).

---

## The Rules in One Page

1. **One bounded context = one Nest module = one top-level folder** under `src/`.
2. **Dependencies point inwards:** `adapter → application → domain`. The module file is the only file that sees every layer.
3. **The domain is plain TypeScript:** no decorators, no `@nestjs/*`, no ORM, no `class-validator`, no Node APIs, no I/O.
4. **Ports are abstract classes**, used directly as DI tokens. Implementations use `implements`, never `extends`.
5. **Application services orchestrate; they don't compute.** Business rules live in domain objects. From npm, services may import only `@nestjs/common` (`@Injectable`) and `@nestjs/cqrs` (Node built-ins such as `node:crypto` are allowed here, never in the domain).
6. **One orchestration style per context:** direct use-case ports (A) or `@nestjs/cqrs` buses (B). Never mix them inside a context.
7. **Adapters are thin translators** between the outside world (HTTP, SQL, events) and ports. Adapters never call each other.
8. **ORM entities, HTTP DTOs and integration events are not domain objects.** Map them explicitly at the boundary.
9. **The use case owns the transaction** (`TransactionPort.run`). Integration events are published **after** the commit.
10. **Contexts meet only through public contracts:** the exported `api/` facade and integration events. The consumer's outgoing adapter is the anti-corruption layer.
11. **The core throws `DomainException`s; adapters translate them** (e.g. to HTTP status). The core never throws `HttpException`.
12. **Every rule a machine can check, a machine checks:** `npm run verify` runs typecheck, lint, dependency-cruiser, unit, integration and system tests.

---

## Mental Model

```
                 driving side                                   driven side
           (who calls the application)                  (what the application calls)

  HTTP ──► adapter/in/web ──┐                         ┌──► adapter/out/persistence ──► DB
  events ► adapter/in/event ┤   ┌───────────────────┐ ├──► adapter/out/event ───────► EventBus / broker
  other  ► adapter/in/api ──┼──►│ application       │─┤
  context                   │   │  port/in   (API)  │ └──► adapter/out/<x> ──────────► other context / HTTP
                            │   │  service   (use   │
                            │   │            cases) │
                            │   │  port/out  (SPI)  │
                            │   │  ┌─────────────┐  │
                            │   │  │   domain    │  │
                            │   │  └─────────────┘  │
                            │   └───────────────────┘
                            └── composition root: <context>.module.ts (binds ports to implementations)
```

| Layer | Contains | May depend on |
|---|---|---|
| **domain** | Entities, value objects, domain events, domain services (pure) | Its own domain, `src/shared/domain` |
| **application/port/in** | Input ports (use-case abstract classes *or* command/query classes), input models, the exceptions they throw | Domain |
| **application/port/out** | Output ports (abstract classes) | Domain |
| **application/service** | Use-case implementations (services / CQRS handlers) | Ports, domain, `@nestjs/common`, `@nestjs/cqrs`, Node built-ins |
| **adapter/in/\*** | Controllers, event listeners, schedulers, the public-API facade | Application ports, domain, any library |
| **adapter/out/\*** | Persistence, messaging, HTTP clients, clients of other contexts | Application ports, domain, any library |
| **api** | The context's published contract (facade + integration events) | Nothing (self-contained) |
| **`<context>.module.ts`** | DI wiring | Everything in the context, other contexts' modules |

`domain/` is a sibling of `application/` (not nested in it) so the dependency rule is visible in paths and checkable with one regex.

---

## Project Structure

```
src/
├── main.ts                         # bootstrap only
├── app.module.ts                   # root composition: infrastructure + contexts
├── config/                         # env schema, process-wide config namespaces
├── shared/                         # shared kernel + cross-cutting infrastructure
│   ├── domain/                     # DomainException hierarchy (pure)
│   ├── application/port/out/       # TransactionPort
│   └── adapter/
│       ├── in/web/                 # WebAdapterModule: ValidationPipe + DomainExceptionFilter
│       └── out/persistence/        # DatabaseModule (TypeORM + nestjs-cls), ClsTransactionAdapter, bigint transformer
├── account/                        # bounded context - style A: direct use-case ports
│   ├── account.module.ts           # composition root of the context
│   ├── account.config.ts           # typed config namespace (registerAs)
│   ├── api/                        # PUBLIC: AccountApi facade, MoneyTransferredEvent, index.ts
│   ├── domain/
│   │   ├── model/                  # Account, Money, AccountId
│   │   └── event/                  # MoneyTransferred (internal domain event)
│   ├── application/
│   │   ├── port/in/                # SendMoneyUseCase, SendMoneyCommand, exceptions
│   │   ├── port/out/               # LoadAccountPort, UpdateAccountStatePort, LockAccountPort, PublishAccountEventPort
│   │   └── service/                # SendMoneyService, MoneyTransferProperties
│   └── adapter/
│       ├── in/web/                 # SendMoneyController, request/response classes
│       ├── in/api/                 # AccountApiAdapter (implements the public facade)
│       ├── out/persistence/        # AccountPersistenceAdapter, *.orm-entity.ts, mapper
│       └── out/event/              # AccountEventPublisherAdapter
└── notification/                   # bounded context - style B: @nestjs/cqrs buses
    ├── notification.module.ts
    ├── domain/model/
    ├── application/{port/in,port/out,service}/
    └── adapter/{in/event,in/web,out/account,out/persistence}/
test/
├── fixtures/                       # test data builders and seeds
└── *.e2e-spec.ts                   # system tests
```

- **One adapter folder per technology and direction** (`adapter/in/web`, `adapter/out/persistence`). Sibling adapter folders never import each other.
- **Unit tests sit next to the file they test** (`*.spec.ts`); system tests live in `test/`.
- **No barrel files (`index.ts`) inside a context** - they create import cycles ("can't resolve dependencies (?)" at boot). The only exception is `api/index.ts`.
- **Relative imports only** (with `.js` extensions under ESM) - no path aliases, so every tool and dependency-cruiser sees real paths.

---

## Naming Conventions

Files are kebab-case with a role suffix.

| Element | Class name | File | Example |
|---|---|---|---|
| Input port (style A) | `<Action><Noun>UseCase` (abstract) | `*.use-case.ts` | `SendMoneyUseCase` |
| Input model (command) | `<Action><Noun>Command` | `*.command.ts` | `SendMoneyCommand` |
| Input model (query) | `<Action><Noun>Query` | use-case file or `*.query.ts` | `GetAccountBalanceQuery` |
| Use case (style A) | `<Action><Noun>Service` | `*.service.ts` | `SendMoneyService` |
| Input port (style B) | `…Command` / `…Query` extending `Command<R>` / `Query<R>` | `*.command.ts` / `*.query.ts` | `SendTransferNotificationsCommand` |
| Use case (style B) | `<Action><Noun>Handler` | `*.handler.ts` | `SendTransferNotificationsHandler` |
| Output port | `<Verb><Noun>Port` (abstract) | `*.port.ts` | `LoadAccountPort` |
| Persistence adapter | `<Aggregate>PersistenceAdapter` | `*.adapter.ts` | `AccountPersistenceAdapter` |
| Other driven adapter | `<Purpose>Adapter` | `*.adapter.ts` | `AccountEventPublisherAdapter` |
| Web adapter | `<Action><Noun>Controller` | `*.controller.ts` | `SendMoneyController` |
| HTTP DTOs | `…Request` / `…Response` | `*.request.ts` / `*.response.ts` | `SendMoneyRequest` |
| Event listener (driving) | `<Event>Listener` | `*.listener.ts` | `MoneyTransferredListener` |
| ORM entity | `<Name>OrmEntity` | `*.orm-entity.ts` | `AccountOrmEntity` |
| Domain exception | `<Problem>Exception` | `*.exception.ts` | `AccountNotFoundException` |
| Integration event | `<PastTense>Event` | `api/*.event.ts` | `MoneyTransferredEvent` |
| Domain event | `<PastTense>` | `domain/event/*.ts` | `MoneyTransferred` |
| Public facade | `<Context>Api` | `api/<context>.api.ts` | `AccountApi` |

---

## Domain Layer

**Rules**
- Plain classes and functions: no decorators, no `@Injectable()`, no `reflect-metadata`
- No imports from `@nestjs/*`, `typeorm`, `class-validator`, `rxjs`, `node:*` or any adapter
- No I/O: no logging, no `process.env`, no HTTP, no database
- Entities protect invariants: private constructor, factory methods (`withId` / `withoutId`), behaviour methods (`withdraw`, `deposit`) instead of setters
- Value objects are immutable (`readonly`) and compared with `equals()`, never `===`

| Concern | Do | Why |
|---|---|---|
| Money / big integers | `bigint` wrapped in a `Money` value object | `number` is exact only up to 2^53 |
| Ids | Small classes with a brand (`private readonly brand = 'AccountId'`) | Structural typing would make `AccountId` and `ActivityId` interchangeable |
| Optional values | `T \| null` with `strict: true` | The compiler forces callers to handle absence |
| Dates | Defensive copies (`new Date(d.getTime())`), UTC instants | `Date` is mutable |
| Immutability | `readonly` fields, `readonly T[]` returns, copies of internal arrays | `readonly` is compile-time only |

```typescript
// src/account/domain/model/money.ts
export class Money {
  static readonly ZERO = new Money(0n);

  private constructor(readonly amount: bigint) {}

  static of(amount: bigint): Money {
    return new Money(amount);
  }

  plus(other: Money): Money { return new Money(this.amount + other.amount); }
  minus(other: Money): Money { return new Money(this.amount - other.amount); }
  isPositive(): boolean { return this.amount > 0n; }
  isNegative(): boolean { return this.amount < 0n; }
  isGreaterThan(other: Money): boolean { return this.amount > other.amount; }
  equals(other: Money): boolean { return this.amount === other.amount; }
  toString(): string { return this.amount.toString(); }
}

// src/account/domain/model/account-id.ts
export class AccountId {
  private readonly brand = 'AccountId';

  constructor(readonly value: number) {}

  equals(other: AccountId): boolean { return this.value === other.value; }
}

// src/account/domain/model/account.ts
import { AccountId } from './account-id.js';
import { Money } from './money.js';

export class Account {
  private constructor(
    readonly id: AccountId | null,
    private balance: Money,
  ) {}

  static withId(id: AccountId, balance: Money): Account { return new Account(id, balance); }
  static withoutId(balance: Money): Account { return new Account(null, balance); }

  getBalance(): Money { return this.balance; }

  /** @returns false when the balance does not cover the amount */
  withdraw(amount: Money): boolean {
    const remaining = this.balance.minus(amount);
    if (remaining.isNegative()) return false;
    this.balance = remaining;
    return true;
  }

  deposit(amount: Money): void {
    this.balance = this.balance.plus(amount);
  }
}
```

**Domain services** (`domain/service/`) hold stateless business logic spanning several entities. They stay framework-free; if they need configuration, give them constructor parameters and register them with `useFactory`. They are **not** the transaction boundary.

**Domain events** (`domain/event/`) use domain types and never leave the context as-is - an outgoing adapter maps them to integration events.

---

## Ports

```typescript
// src/account/application/port/out/load-account.port.ts
import type { AccountId } from '../../../domain/model/account-id.js';
import type { Account } from '../../../domain/model/account.js';

export abstract class LoadAccountPort {
  abstract loadAccount(accountId: AccountId): Promise<Account | null>;
}

// src/account/application/port/out/update-account-state.port.ts
export abstract class UpdateAccountStatePort {
  abstract updateBalance(account: Account): Promise<void>;
}

// src/account/application/port/out/lock-account.port.ts
export abstract class LockAccountPort {
  /** Must run inside TransactionPort.run - locks the row until commit. */
  abstract lockAccount(accountId: AccountId): Promise<void>;
}
```

- An **abstract class is a type and a DI token at once** (interfaces vanish at runtime). Consumers inject it without `@Inject()`, and renames are refactor-safe.
- Implement ports with **`implements`**; `extends` forces `super()` and invites shared behaviour.
- `Symbol` tokens + `@Inject(TOKEN)` only for non-class values (config objects, third-party clients).
- Never `import type` an injected **class** (DI metadata becomes `Object`). Pure **types** in a decorated constructor (e.g. `ConfigType<…>`) are the opposite: they must use `import type` under `isolatedModules`.
- **Small, purpose-specific ports** (`LoadAccountPort`, `UpdateAccountStatePort`) instead of one `AccountRepository`. One adapter may implement several; register it once and alias with `useExisting` - two `useClass` entries create **two instances**.
- **Anything that may do I/O returns a `Promise`**, even if today's adapter is in-memory.
- **Absence is `null`, not an exception.** The application service decides whether absence is an error.
- **Exceptions are part of the contract:** document them with `@throws` on the input port and keep the classes next to it (`application/port/in/*.exception.ts`).

---

## Use Case (Style A - direct ports)

```typescript
// src/account/application/port/in/send-money.command.ts - self-validating input model
import { InvalidInputException } from '../../../../shared/domain/domain.exception.js';
import type { AccountId } from '../../../domain/model/account-id.js';
import type { Money } from '../../../domain/model/money.js';

export class SendMoneyCommand {
  constructor(
    readonly sourceAccountId: AccountId,
    readonly targetAccountId: AccountId,
    readonly money: Money,
  ) {
    if (!money.isPositive()) throw new InvalidInputException('Amount must be positive');
    if (sourceAccountId.equals(targetAccountId)) throw new InvalidInputException('Source and target accounts must differ');
  }
}

// src/account/application/port/in/send-money.use-case.ts
export abstract class SendMoneyUseCase {
  /** @throws AccountNotFoundException, ThresholdExceededException */
  abstract sendMoney(command: SendMoneyCommand): Promise<boolean>;
}

// src/account/application/port/in/account-not-found.exception.ts
export class AccountNotFoundException extends DomainException {
  constructor(readonly accountId: AccountId) {
    super('not-found', 'ACCOUNT_NOT_FOUND', `Account ${accountId.value} not found`);
  }
}
```

```typescript
// src/account/application/service/send-money.service.ts
import { Injectable } from '@nestjs/common';
import { TransactionPort } from '../../../shared/application/port/out/transaction.port.js';
import { MoneyTransferred } from '../../domain/event/money-transferred.js';
import type { Account } from '../../domain/model/account.js';
import type { AccountId } from '../../domain/model/account-id.js';
import { AccountNotFoundException } from '../port/in/account-not-found.exception.js';
import type { SendMoneyCommand } from '../port/in/send-money.command.js';
import { SendMoneyUseCase } from '../port/in/send-money.use-case.js';
import { ThresholdExceededException } from '../port/in/threshold-exceeded.exception.js';
import { LoadAccountPort } from '../port/out/load-account.port.js';
import { LockAccountPort } from '../port/out/lock-account.port.js';
import { PublishAccountEventPort } from '../port/out/publish-account-event.port.js';
import { UpdateAccountStatePort } from '../port/out/update-account-state.port.js';
import { MoneyTransferProperties } from './money-transfer.properties.js';

@Injectable()
export class SendMoneyService implements SendMoneyUseCase {
  constructor(
    private readonly loadAccountPort: LoadAccountPort,
    private readonly lockAccountPort: LockAccountPort,
    private readonly updateAccountStatePort: UpdateAccountStatePort,
    private readonly publishAccountEventPort: PublishAccountEventPort,
    private readonly transaction: TransactionPort,
    private readonly properties: MoneyTransferProperties,
  ) {}

  async sendMoney(command: SendMoneyCommand): Promise<boolean> {
    const threshold = this.properties.maximumTransferThreshold;
    if (command.money.isGreaterThan(threshold)) throw new ThresholdExceededException(threshold, command.money);

    const transferred = await this.transaction.run(() => this.transfer(command)); // BEGIN … COMMIT
    if (transferred) {
      await this.publishAccountEventPort.publish(                                    // after COMMIT
        new MoneyTransferred(command.sourceAccountId, command.targetAccountId, command.money, new Date()),
      );
    }
    return transferred;
  }

  private async transfer({ sourceAccountId, targetAccountId, money }: SendMoneyCommand): Promise<boolean> {
    // lock rows in a stable order to avoid deadlocks between opposite transfers
    for (const id of [sourceAccountId, targetAccountId].sort((a, b) => a.value - b.value)) {
      await this.lockAccountPort.lockAccount(id);
    }
    const source = await this.load(sourceAccountId);
    const target = await this.load(targetAccountId);

    if (!source.withdraw(money)) return false;   // business rule lives in the domain
    target.deposit(money);

    await this.updateAccountStatePort.updateBalance(source);
    await this.updateAccountStatePort.updateBalance(target);
    return true;
  }

  private async load(id: AccountId): Promise<Account> {
    const account = await this.loadAccountPort.loadAccount(id);
    if (!account) throw new AccountNotFoundException(id);
    return account;
  }
}

// src/account/application/service/money-transfer.properties.ts - config as a plain object
export class MoneyTransferProperties {
  constructor(readonly maximumTransferThreshold: Money) {}
}
```

**An application service does:** check preconditions that need data, load aggregates through output ports, call domain behaviour, persist through output ports, define the transaction boundary, publish events after commit.
**It doesn't:** compute business results, know about HTTP/SQL/JSON, catch exceptions only to log them, or call another use case's service (extract a domain service or shared output port instead).

Style B (`@nestjs/cqrs`) and how to choose: `references/orchestration.md`.

---

## Adapters (summary)

```typescript
// src/account/adapter/in/web/send-money.request.ts - syntactic validation only
import { IsInt, IsPositive, IsString, Matches } from 'class-validator';

export class SendMoneyRequest {
  @IsInt()
  @IsPositive()
  targetAccountId!: number;

  @IsString()
  @Matches(/^\d{1,30}$/, { message: 'amount must be a non-negative integer encoded as a string' })
  amount!: string; // money travels as a decimal string - JSON numbers are doubles, JSON.stringify throws on bigint
}

// src/account/adapter/in/web/send-money.controller.ts
@Controller({ path: 'accounts', version: '1' })
export class SendMoneyController {
  constructor(private readonly sendMoneyUseCase: SendMoneyUseCase) {}

  @Post(':sourceAccountId/transfers')
  @HttpCode(HttpStatus.NO_CONTENT)
  async sendMoney(
    @Param('sourceAccountId', ParseIntPipe) sourceAccountId: number,
    @Body() request: SendMoneyRequest,
  ): Promise<void> {
    const command = new SendMoneyCommand(                     // semantic validation (core)
      new AccountId(sourceAccountId),
      new AccountId(request.targetAccountId),
      Money.of(BigInt(request.amount)),
    );
    if (!(await this.sendMoneyUseCase.sendMoney(command))) {
      throw new UnprocessableEntityException({ statusCode: 422, error: 'INSUFFICIENT_FUNDS', message: 'Insufficient funds' });
    }
  }
}
```

```typescript
// src/account/adapter/out/persistence/account-persistence.adapter.ts
@Injectable()
export class AccountPersistenceAdapter implements LoadAccountPort, LockAccountPort, UpdateAccountStatePort {
  constructor(private readonly txHost: TransactionHost<TransactionalAdapterTypeOrm>) {}

  // joins the transaction opened by the use case - never use @InjectRepository() here
  private get accounts(): Repository<AccountOrmEntity> {
    return this.txHost.tx.getRepository(AccountOrmEntity);
  }

  async loadAccount(accountId: AccountId): Promise<Account | null> {
    const row = await this.accounts.findOneBy({ id: accountId.value });
    return row ? toDomain(row) : null;
  }

  async lockAccount(accountId: AccountId): Promise<void> {
    await this.accounts.findOne({ where: { id: accountId.value }, lock: { mode: 'pessimistic_write' } });
  }

  async updateBalance(account: Account): Promise<void> {
    if (!account.id) throw new Error('Cannot update an account without id'); // programming error, not a domain failure
    await this.accounts.update({ id: account.id.value }, { balance: account.getBalance().amount });
  }
}

// src/account/adapter/out/persistence/account.mapper.ts - stateless mappers are plain functions
export function toDomain(row: AccountOrmEntity): Account {
  return Account.withId(new AccountId(row.id), Money.of(row.balance));
}
```

- **One controller per use case**; request DTOs never reach the application layer; controllers return response classes, never domain objects
- **ORM entities live only in the adapter** and are mapped explicitly
- **Incoming event listeners are driving adapters** - translate to a command, never let an exception escape

Details (event, scheduler, messaging and HTTP adapters, bounded-context integration): `references/adapters.md`.

---

## Wiring: Modules as Composition Roots

```typescript
// src/account/account.module.ts
/**
 * Account bounded context.
 * Orchestration style: A (direct use-case ports).
 */
@Module({
  imports: [ConfigModule.forFeature(accountConfig), TypeOrmModule.forFeature([AccountOrmEntity])],
  controllers: [SendMoneyController, GetAccountBalanceController],
  providers: [
    { provide: SendMoneyUseCase, useClass: SendMoneyService },               // input port -> service
    { provide: GetAccountBalanceUseCase, useClass: GetAccountBalanceService },
    AccountPersistenceAdapter,                                               // one instance…
    { provide: LoadAccountPort, useExisting: AccountPersistenceAdapter },    // …serving several ports
    { provide: LockAccountPort, useExisting: AccountPersistenceAdapter },
    { provide: UpdateAccountStatePort, useExisting: AccountPersistenceAdapter },
    { provide: PublishAccountEventPort, useClass: AccountEventPublisherAdapter },
    {
      provide: MoneyTransferProperties,                                      // config -> plain object
      inject: [accountConfig.KEY],
      useFactory: (config: ConfigType<typeof accountConfig>) =>
        new MoneyTransferProperties(Money.of(config.transferThreshold)),
    },
    { provide: AccountApi, useClass: AccountApiAdapter },                    // public facade
  ],
  exports: [AccountApi],                                                     // the ONLY public surface
})
export class AccountModule {}
```

| Provider form | Use it for |
|---|---|
| `{ provide: Port, useClass: Impl }` | Binding a port to its single implementation |
| `Impl` + `{ provide: Port, useExisting: Impl }` | One instance serving several ports |
| `{ provide: X, inject: [...], useFactory }` | Framework-free objects (config, parameterised domain services) |
| `{ provide: TOKEN, useValue }` | Constants and test doubles |

**Rules**
- **`exports` is the public API of the context:** export the `api/` facade only - never use cases, ports or adapters
- **Modules are not `@Global()`**, except cross-cutting infrastructure (`DatabaseModule`, `ConfigModule`, `ClsModule`)
- **`forwardRef()` is a design smell:** a module cycle means two contexts are too entangled - introduce an event or move the concept
- **Default (singleton) scope only.** For request data (correlation id, user) use `nestjs-cls`, not `Scope.REQUEST`
- **Config is typed** (`registerAs` in `<context>.config.ts`), validated at boot, and turned into plain objects before it reaches the core. **The core never reads `process.env` or `ConfigService`.**
- **Register HTTP-pipeline concerns as `APP_PIPE`/`APP_FILTER` providers in `WebAdapterModule`**, not with `app.useGlobal*()`, so tests that import the module behave like production (in hexagonal projects this replaces the pipe/filter part of the `nestjs` skill's `configureApp()`)

---

## Conscious Shortcuts

Allowed **when written down** (ADR or module JSDoc) and cheap to undo:

- **No input-port abstract class** for a context with exactly one driving adapter and no test that mocks it - the controller depends on the service class. Re-introduce the port when a second adapter appears.
- **Read models that bypass the aggregate:** a query port returning a flat DTO straight from SQL for lists and reports.
- **One model shared by in/out ports** in a trivial CRUD context.
- **No hexagonal at all** for thin CRUD/proxy services.

**Never acceptable:** framework or ORM imports in `domain/`, adapters calling adapters, contexts importing each other's internals, `HttpException` in the core.

---

## Recipes

### Add a use case (style A - direct ports)
1. Model the behaviour in `domain/model` and unit-test it
2. Create `application/port/in/<action>.command.ts` (self-validating) and `<action>.use-case.ts` (abstract class with `@throws` JSDoc)
3. Add missing output ports to `application/port/out/` (small, one purpose each)
4. Implement `application/service/<action>.service.ts` (`@Injectable()`, `implements <Action>UseCase`); wrap writes in `TransactionPort.run`, publish events after it; unit-test with mocked ports
5. Implement or extend adapters: controller + request DTO + slice test; persistence + integration test
6. Bind everything in `<context>.module.ts`
7. `npm run verify`

### Add a use case (style B - CQRS)
1. Domain first, as above
2. `application/port/in/<action>.command.ts` (`extends Command<R>`) or `<action>.query.ts` (`extends Query<R>`)
3. `application/service/<action>.handler.ts` with `@CommandHandler(...)` / `@QueryHandler(...)`, depending on output ports only; unit-test by calling `execute()`
4. Driving adapter: controller calling `commandBus.execute()` / `queryBus.execute()`, or an `@EventsHandler` translating an event
5. Add the handler to the module `providers`; add or extend a system test that goes through the bus
6. `npm run verify`

### Add a bounded context
1. `src/<context>/` with `domain`, `application/port/{in,out}`, `application/service`, `adapter/...`, `<context>.module.ts`
2. Choose the orchestration style and state it in the module JSDoc
3. If other contexts need it: create `api/` (facade + events), implement the facade in `adapter/in/api`, export **only** the facade
4. Import the module in `AppModule` - the path-based dependency-cruiser rules apply automatically

## Pull Request Checklist

- [ ] No import from `@nestjs/*`, `typeorm`, `class-validator` or `node:*` in `domain/`
- [ ] No `HttpException` and no `process.env` outside adapters and config
- [ ] Ports are abstract classes; implementations use `implements`; multi-port adapters use `useExisting`
- [ ] Writes run inside `TransactionPort.run`; events are published after it
- [ ] Persistence adapters use `txHost.tx`, not `@InjectRepository()`
- [ ] Money is `bigint`/`Money` in the core and a decimal string in JSON
- [ ] Controllers return response classes, not domain objects
- [ ] Module `exports` contains only the `api/` facade
- [ ] New behaviour has unit tests, new adapters have integration tests, `npm run verify` is green

---

## Reference Guide

| Topic | Reference | When to Load |
|-------|-----------|-------------|
| Orchestration | `references/orchestration.md` | Choosing style A vs B, `@nestjs/cqrs` commands/queries/handlers |
| Adapters & contexts | `references/adapters.md` | Web, persistence, event, scheduler, HTTP adapters; `api/` facades; anti-corruption layer |
| Shared kernel | `references/shared-kernel.md` | `DomainException` + filter, validation levels, `WebAdapterModule`, `TransactionPort`, `DatabaseModule` |
| Consistency | `references/consistency.md` | Transaction boundary, publish after commit, outbox, locking and concurrency |
| Testing | `references/testing.md` | Domain, use case, handler, controller slice, persistence and system tests (Vitest) |
| Pitfalls | `references/pitfalls.md` | JavaScript/TypeScript/Node behaviours that break hexagonal code, and how to address each |
| Enforcement | `references/enforcement.md` | dependency-cruiser rules, ESLint restrictions, `npm run verify` |
