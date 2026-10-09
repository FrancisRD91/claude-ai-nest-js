# Adapters & Bounded Contexts

Adapters translate between the outside world and ports. They hold no business logic and never call each other.

## Web Adapters (driving)

```typescript
// adapter/in/web/get-account-balance.response.ts
export class AccountBalanceResponse {
  constructor(
    readonly accountId: number,
    readonly balance: string, // decimal string - bigint is not JSON-serialisable
  ) {}
}

// adapter/in/web/get-account-balance.controller.ts
@Controller({ path: 'accounts', version: '1' })
export class GetAccountBalanceController {
  constructor(private readonly getAccountBalanceUseCase: GetAccountBalanceUseCase) {}

  @Get(':accountId/balance')
  async getBalance(@Param('accountId', ParseIntPipe) accountId: number): Promise<AccountBalanceResponse> {
    const balance = await this.getAccountBalanceUseCase.getAccountBalance(
      new GetAccountBalanceQuery(new AccountId(accountId)),
    );
    return new AccountBalanceResponse(accountId, balance.toString());
  }
}
```

- **One controller per use case** keeps controllers tiny and their tests focused; several controllers can share a route prefix
- Request DTOs (`*.request.ts`) carry **syntactic** validation (`class-validator` or Standard Schema) and never reach the application layer
- **Never return domain objects** - Nest would serialise internals and `JSON.stringify` throws on `bigint`
- Controllers may throw `HttpException`s for transport concerns (e.g. 422 on a `false` result); domain failures arrive as `DomainException`s and are translated centrally by `DomainExceptionFilter`
- Prefer resourceful URLs with a JSON body (`POST /accounts/1/transfers`) over values in the path - amounts in URLs end up in access logs

## Persistence Adapters (driven)

```typescript
// shared/adapter/out/persistence/bigint.transformer.ts
import type { ValueTransformer } from 'typeorm';

// pg returns bigint columns as string, sqlite as number - normalise both to bigint
export const bigintTransformer: ValueTransformer = {
  to: (value: bigint | null | undefined): string | null | undefined =>
    value === null || value === undefined ? value : value.toString(),
  from: (value: string | number | bigint | null | undefined): bigint | null | undefined =>
    value === null || value === undefined ? value : BigInt(value),
};

// account/adapter/out/persistence/account.orm-entity.ts
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';
import { bigintTransformer } from '../../../../shared/adapter/out/persistence/bigint.transformer.js';

@Entity('accounts')
export class AccountOrmEntity {
  @PrimaryGeneratedColumn()
  id!: number;

  @Column('bigint', { transformer: bigintTransformer })
  balance!: bigint;
}
```

- **ORM entities live only in the adapter** (`*.orm-entity.ts`) and are mapped explicitly (`*.mapper.ts`); stateless mappers are **plain functions**, not injectables
- **Get the `EntityManager` from the transaction host** (`this.txHost.tx`), never from `@InjectRepository()` - injected repositories are bound to the default connection and **silently run outside** the use case's transaction
- Each context registers its entities with `TypeOrmModule.forFeature([...])` in its own module; the root uses `autoLoadEntities: true`, so `AppModule` never imports adapter internals
- Normalise driver differences in the adapter (e.g. `bigintTransformer`)
- `synchronize: true` only for throwaway test databases - real databases use **migrations**
- Read models for list/report screens may bypass the aggregate with a query port returning flat DTOs (a documented shortcut)

See `typeorm-patterns` for N+1, locking and bulk-write pitfalls.

## Event Adapters

### Outgoing - domain event → integration event

```typescript
// account/adapter/out/event/account-event-publisher.adapter.ts
import { Injectable } from '@nestjs/common';
import { EventBus } from '@nestjs/cqrs';
import { MoneyTransferredEvent } from '../../../api/index.js';
import type { MoneyTransferred } from '../../../domain/event/money-transferred.js';
import { PublishAccountEventPort } from '../../../application/port/out/publish-account-event.port.js';

@Injectable()
export class AccountEventPublisherAdapter implements PublishAccountEventPort {
  constructor(private readonly eventBus: EventBus) {}

  async publish(event: MoneyTransferred): Promise<void> {
    this.eventBus.publish(
      new MoneyTransferredEvent(
        event.sourceAccountId.value,
        event.targetAccountId.value,
        event.money.toString(),          // JSON-safe decimal string
        event.occurredAt.toISOString(),  // JSON-safe ISO date
      ),
    );
  }
}
```

The transport (Nest `EventBus` today, Kafka or an outbox relay tomorrow) is an adapter detail - only this class changes.

### Incoming - integration event → own command

```typescript
// notification/adapter/in/event/money-transferred.listener.ts
import { Logger } from '@nestjs/common';
import { CommandBus, EventsHandler, type IEventHandler } from '@nestjs/cqrs';
import { MoneyTransferredEvent } from '../../../../account/api/index.js';
import { SendTransferNotificationsCommand } from '../../../application/port/in/send-transfer-notifications.command.js';

@EventsHandler(MoneyTransferredEvent)
export class MoneyTransferredListener implements IEventHandler<MoneyTransferredEvent> {
  private readonly logger = new Logger(MoneyTransferredListener.name);

  constructor(private readonly commandBus: CommandBus) {}

  async handle(event: MoneyTransferredEvent): Promise<void> {
    try {
      await this.commandBus.execute(
        new SendTransferNotificationsCommand(event.sourceAccountId, event.targetAccountId, BigInt(event.amount)),
      );
    } catch (err) {
      // a listener has no exception filter: log, then retry or dead-letter - never let it escape
      this.logger.error('Failed to handle MoneyTransferredEvent', err instanceof Error ? err.stack : String(err));
    }
  }
}
```

## Other Adapters

| Need | Adapter | Notes |
|---|---|---|
| Scheduled jobs | `adapter/in/scheduler` (`@nestjs/schedule`) | Calls an input port; holds no logic; runs on every replica unless locked |
| Message consumers | `adapter/in/messaging` (`@nestjs/microservices`, broker SDK) | Idempotent; maps messages to commands |
| Outbound HTTP | `adapter/out/http` (`fetch` or `@nestjs/axios`) | Maps remote DTOs to the port's types; timeouts and retries live here |
| Another context | `adapter/out/<context>` | Anti-corruption layer over the other context's `api/` |

```typescript
// account/adapter/in/scheduler/close-dormant-accounts.scheduler.ts
@Injectable()
export class CloseDormantAccountsScheduler {
  constructor(private readonly closeDormantAccountsUseCase: CloseDormantAccountsUseCase) {}

  @Cron(CronExpression.EVERY_DAY_AT_2AM)
  async run(): Promise<void> {
    await this.closeDormantAccountsUseCase.closeDormantAccounts(new CloseDormantAccountsCommand(new Date()));
  }
}
```

---

## Multiple Bounded Contexts

```
account                                        notification
───────                                        ────────────
api/AccountApi  ◄──implements── adapter/in/api/AccountApiAdapter
     ▲                                          adapter/out/account/AccountBalanceAdapter ──► LoadAccountBalancePort
     └──────────────────── sync call ───────────┘                       (notification's own port)

api/MoneyTransferredEvent ◄── adapter/out/event/AccountEventPublisherAdapter
     │
     └──────────── async (EventBus) ──────────► adapter/in/event/MoneyTransferredListener ──► SendTransferNotificationsCommand
```

### The Public Contract (`api/`)

```typescript
// account/api/account.api.ts - self-contained: primitives and plain classes, no imports
export abstract class AccountApi {
  /** @returns the balance as a decimal string, or null when the account does not exist */
  abstract getBalance(accountId: number): Promise<string | null>;
}

// account/api/money-transferred.event.ts
export class MoneyTransferredEvent {
  constructor(
    readonly sourceAccountId: number,
    readonly targetAccountId: number,
    readonly amount: string,     // decimal string
    readonly occurredAt: string, // ISO-8601
  ) {}
}

// account/api/index.ts - the only barrel file allowed in a context
export { AccountApi } from './account.api.js';
export { MoneyTransferredEvent } from './money-transferred.event.js';
```

### Provider Side - Facade Implemented by an Incoming Adapter

```typescript
// account/adapter/in/api/account-api.adapter.ts
@Injectable()
export class AccountApiAdapter implements AccountApi {
  constructor(private readonly getAccountBalanceUseCase: GetAccountBalanceUseCase) {}

  async getBalance(accountId: number): Promise<string | null> {
    try {
      const balance = await this.getAccountBalanceUseCase.getAccountBalance(
        new GetAccountBalanceQuery(new AccountId(accountId)),
      );
      return balance.toString();
    } catch (err) {
      if (err instanceof AccountNotFoundException) return null; // translate internals into the contract
      throw err;
    }
  }
}
```

### Consumer Side - Own Port + Anti-Corruption Layer

```typescript
// notification/application/port/out/load-account-balance.port.ts - consumer's own vocabulary
export abstract class LoadAccountBalancePort {
  abstract loadBalance(accountId: number): Promise<bigint | null>;
}

// notification/adapter/out/account/account-balance.adapter.ts
import { Injectable } from '@nestjs/common';
import { AccountApi } from '../../../../account/api/index.js';
import { LoadAccountBalancePort } from '../../../application/port/out/load-account-balance.port.js';

@Injectable()
export class AccountBalanceAdapter implements LoadAccountBalancePort {
  constructor(private readonly accountApi: AccountApi) {}

  async loadBalance(accountId: number): Promise<bigint | null> {
    const balance = await this.accountApi.getBalance(accountId);
    return balance === null ? null : BigInt(balance);
  }
}

// notification/notification.module.ts
@Module({
  imports: [AccountModule], // only module files import other modules
  providers: [{ provide: LoadAccountBalancePort, useClass: AccountBalanceAdapter } /* … */],
})
export class NotificationModule {}
```

If `account` becomes a separate service, only `AccountBalanceAdapter` changes (to an HTTP or gRPC client).

### Rules

1. A context's public contract lives in `src/<context>/api/`: an abstract facade (the exported DI token) and integration events - **self-contained**, JSON-safe (decimal strings for money, ISO strings for dates), no imports from internals or libraries
2. The facade is implemented by an **incoming adapter** of the provider (`adapter/in/api`) on top of its input ports, translating internal exceptions into the contract
3. The consumer defines **its own output port** and implements it with an **outgoing adapter** that calls the facade - the anti-corruption layer
4. **Domain and application code never import another context**, not even its `api/`; only adapters do, and only module files import other modules
5. Each context has **its own model** of shared concepts (notification uses `number`/`bigint`, never `AccountId`/`Money`); share contracts, not classes. `src/shared` holds only generic building blocks (`DomainException`, `TransactionPort`)
6. Use **events** for "something happened, react if you care" and the **facade** for "I need an answer now"; avoid synchronous call chains across more than two contexts
