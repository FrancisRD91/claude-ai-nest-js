# Use Cases - Two Orchestration Styles

In NestJS, module boundaries and abstract-class tokens already decouple controllers from implementations, so **a mediator is optional**. Two styles are supported. **Choose one per bounded context and state it in the module's JSDoc.**

## Style A - Direct Use-Case Ports

```typescript
// application/port/in/send-money.use-case.ts
export abstract class SendMoneyUseCase {
  /** @throws AccountNotFoundException, ThresholdExceededException */
  abstract sendMoney(command: SendMoneyCommand): Promise<boolean>;
}

// application/service/send-money.service.ts
@Injectable()
export class SendMoneyService implements SendMoneyUseCase { /* … */ }

// adapter/in/web/send-money.controller.ts
@Controller({ path: 'accounts', version: '1' })
export class SendMoneyController {
  constructor(private readonly sendMoneyUseCase: SendMoneyUseCase) {}
}

// account.module.ts
{ provide: SendMoneyUseCase, useClass: SendMoneyService },
```

- ✅ Type-checked end to end: a missing binding fails at boot with a precise error; "go to implementation" works
- ✅ Zero extra concepts, least indirection
- ❌ Per-use-case cross-cutting behaviour (audit, metrics) needs interceptors (HTTP only) or decorators

### Queries in Style A

```typescript
// application/port/in/get-account-balance.use-case.ts
export class GetAccountBalanceQuery {
  constructor(readonly accountId: AccountId) {}
}

export abstract class GetAccountBalanceUseCase {
  /** @throws AccountNotFoundException */
  abstract getAccountBalance(query: GetAccountBalanceQuery): Promise<Money>;
}

// application/service/get-account-balance.service.ts
@Injectable()
export class GetAccountBalanceService implements GetAccountBalanceUseCase {
  constructor(private readonly loadAccountPort: LoadAccountPort) {}

  async getAccountBalance({ accountId }: GetAccountBalanceQuery): Promise<Money> {
    const account = await this.loadAccountPort.loadAccount(accountId);
    if (!account) throw new AccountNotFoundException(accountId);
    return account.getBalance();
  }
}
```

## Style B - `@nestjs/cqrs` Buses

`@nestjs/cqrs` is Nest's first-party mediator.

| Concept | `@nestjs/cqrs` |
|---|---|
| Command with result type `R` | `class X extends Command<R>` |
| Query with result type `R` | `class X extends Query<R>` |
| Use case | `@CommandHandler(X) class XHandler implements ICommandHandler<X>` |
| Dispatch | `commandBus.execute(command)` → `Promise<R>` (typed) |
| No result | `Command<void>` |
| Notifications | `EventBus` + `@EventsHandler` |
| Pipeline middleware | None - use interceptors (HTTP), a custom publisher, or decorators |

```typescript
// application/port/in/send-transfer-notifications.command.ts  <- the input port
import { Command } from '@nestjs/cqrs';

export class SendTransferNotificationsCommand extends Command<void> {
  constructor(
    readonly sourceAccountId: number,
    readonly targetAccountId: number,
    readonly amount: bigint,
  ) {
    super();
  }
}

// application/service/send-transfer-notifications.handler.ts  <- the use case
import { CommandHandler, type ICommandHandler } from '@nestjs/cqrs';

@CommandHandler(SendTransferNotificationsCommand)
export class SendTransferNotificationsHandler implements ICommandHandler<SendTransferNotificationsCommand> {
  constructor(
    private readonly loadAccountBalancePort: LoadAccountBalancePort,
    private readonly saveNotificationsPort: SaveNotificationsPort,
    private readonly deliverNotificationPort: DeliverNotificationPort,
  ) {}

  async execute(command: SendTransferNotificationsCommand): Promise<void> {
    const balance = await this.loadAccountBalancePort.loadBalance(command.targetAccountId);
    if (balance === null) return; // account no longer exists - nothing to notify

    const notification = Notification.credit(command.targetAccountId, command.amount, balance);
    await this.saveNotificationsPort.save(notification);
    await this.deliverNotificationPort.deliver(notification);
  }
}

// application/port/in/list-notifications.query.ts
import { Query } from '@nestjs/cqrs';

export class ListNotificationsQuery extends Query<Notification[]> {
  constructor(readonly accountId: number) {
    super();
  }
}

// application/service/list-notifications.handler.ts
@QueryHandler(ListNotificationsQuery)
export class ListNotificationsHandler implements IQueryHandler<ListNotificationsQuery> {
  constructor(private readonly loadNotificationsPort: LoadNotificationsPort) {}

  execute({ accountId }: ListNotificationsQuery): Promise<Notification[]> {
    return this.loadNotificationsPort.loadByAccount(accountId);
  }
}

// adapter/in/web/list-notifications.controller.ts  <- driving adapter
@Controller({ path: 'accounts', version: '1' })
export class ListNotificationsController {
  constructor(private readonly queryBus: QueryBus) {}

  @Get(':accountId/notifications')
  async list(@Param('accountId', ParseIntPipe) accountId: number): Promise<NotificationResponse[]> {
    const notifications = await this.queryBus.execute(new ListNotificationsQuery(accountId)); // Notification[]
    return notifications.map(NotificationResponse.from);
  }
}
```

**Rules for style B**
- Register `CqrsModule.forRoot()` **once** in `AppModule`. In feature modules, handlers and listeners are plain `providers`.
- A handler is still a use case: it depends on **output ports only**; unit-test it by calling `execute()` directly (no bus).
- Commands and queries are matched by class: exactly one handler per command across the application.
- A missing handler fails only when executed - cover every command/query with at least one test that goes through the bus.

- ✅ Uniform shape for every use case; natural fit for event-driven flows (events → commands) and sagas
- ✅ Controllers depend on one bus instead of N use cases
- ❌ Indirection: "who handles this?" is answered at runtime

## Which One?

| Situation | Style |
|---|---|
| Request/response core domain with a handful of use cases | **A** |
| Context driven mainly by events (reacting to other contexts, workflows, sagas) | **B** |
| Many use cases needing the same cross-cutting policy (audit trail, idempotency keys) | **B**, with a decorator around the bus |
| Public API consumed by other contexts | **A**, exposing a facade - even if the context uses B internally |
| Unsure | **A** - moving from A to B later is mechanical |

**Never mix both styles inside one context.** A reader must know where to look for "what can this context do": `port/in/*.use-case.ts` (A) or `port/in/*.command.ts | *.query.ts` (B).

## What an Application Service (or Handler) Does - and Doesn't

| Does | Doesn't |
|---|---|
| Check business preconditions that need data (existence, thresholds) | Compute business results (that's the domain) |
| Load aggregates through output ports | Know about HTTP, SQL or JSON |
| Call domain behaviour | Catch exceptions only to log them |
| Persist through output ports | Call another use case's service |
| Define the transaction boundary | Read `process.env` or `ConfigService` |
| Publish events after commit | Import adapters |

When two use cases share logic, extract a **domain service** or a shared **output port**.
