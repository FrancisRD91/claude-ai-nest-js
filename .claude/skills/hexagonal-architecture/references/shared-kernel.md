# Shared Kernel - Errors, Validation, Transactions & Wiring

`src/shared` holds only generic building blocks used by every context. It never imports a context.

## Errors

```
Error
└── DomainException (shared/domain)          kind: 'invalid-input' | 'not-found' | 'rule-violation' | 'conflict'
    ├── InvalidInputException                → 400
    ├── AccountNotFoundException             → 404
    └── ThresholdExceededException           → 422
```

```typescript
// src/shared/domain/domain.exception.ts - pure, no framework
export type DomainExceptionKind = 'invalid-input' | 'not-found' | 'rule-violation' | 'conflict';

export abstract class DomainException extends Error {
  // kind and code are set through the constructor of the base class: subclass fields
  // without initialisers would be reset after super() (see pitfalls.md, class fields)
  protected constructor(
    readonly kind: DomainExceptionKind,
    /** Stable, machine-readable code, e.g. ACCOUNT_NOT_FOUND */
    readonly code: string,
    message: string,
  ) {
    super(message);
    this.name = new.target.name;
  }
}

/** Generic "the caller gave us something invalid" - used by self-validating commands and value objects. */
export class InvalidInputException extends DomainException {
  constructor(message: string, code = 'INVALID_INPUT') {
    super('invalid-input', code, message);
  }
}

// src/account/application/port/in/account-not-found.exception.ts
export class AccountNotFoundException extends DomainException {
  constructor(readonly accountId: AccountId) {
    super('not-found', 'ACCOUNT_NOT_FOUND', `Account ${accountId.value} not found`);
  }
}

// src/account/application/port/in/threshold-exceeded.exception.ts
export class ThresholdExceededException extends DomainException {
  constructor(threshold: Money, actual: Money) {
    super(
      'rule-violation',
      'TRANSFER_THRESHOLD_EXCEEDED',
      `Maximum threshold for transferring money exceeded: tried ${actual.toString()} but threshold is ${threshold.toString()}`,
    );
  }
}
```

- The core throws `DomainException` subclasses with a semantic **`kind`** and a stable **`code`** - never `HttpException`, because the same use case may be driven by a message consumer
- `DomainExceptionFilter` is the **only** place where `kind` becomes an HTTP status
- Anything that is not a `DomainException` is a bug and becomes a 500 via Nest's default filter - don't catch and rethrow just to log
- Event listeners and schedulers have no filter: they catch, log and decide (retry / dead-letter) themselves

```typescript
// src/shared/adapter/in/web/domain-exception.filter.ts
import { type ArgumentsHost, Catch, type ExceptionFilter, HttpStatus } from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';
import { DomainException, type DomainExceptionKind } from '../../../domain/domain.exception.js';

const STATUS_BY_KIND: Record<DomainExceptionKind, HttpStatus> = {
  'invalid-input': HttpStatus.BAD_REQUEST,
  'not-found': HttpStatus.NOT_FOUND,
  'rule-violation': HttpStatus.UNPROCESSABLE_ENTITY,
  conflict: HttpStatus.CONFLICT,
};

@Catch(DomainException)
export class DomainExceptionFilter implements ExceptionFilter<DomainException> {
  constructor(private readonly adapterHost: HttpAdapterHost) {}

  catch(exception: DomainException, host: ArgumentsHost): void {
    const statusCode = STATUS_BY_KIND[exception.kind];
    // same shape as Nest's HttpException bodies; works with Express and Fastify
    this.adapterHost.httpAdapter.reply(
      host.switchToHttp().getResponse(),
      { statusCode, error: exception.code, message: exception.message },
      statusCode,
    );
  }
}
```

## Two Levels of Validation (plus business rules)

| Level | Where | Tool | Example |
|---|---|---|---|
| **Syntactic** - shape and format of raw input | Adapter (`SendMoneyRequest`) | `class-validator` + global `ValidationPipe` (or Standard Schema) | `amount` is a string of digits |
| **Semantic** - rules of the use-case input | Input model constructor (`SendMoneyCommand`) | Plain TypeScript throwing `InvalidInputException` | amount > 0, source ≠ target |
| **Business** - needs state | Domain / application service | Domain methods, `DomainException`s | balance covers amount, threshold |

With `strict: true`, null checks on inputs are the compiler's job - no runtime `@NotNull` equivalent needed in the core.

## WebAdapterModule

```typescript
// src/shared/adapter/in/web/web-adapter.module.ts
import { Module, ValidationPipe } from '@nestjs/common';
import { APP_FILTER, APP_PIPE } from '@nestjs/core';
import { DomainExceptionFilter } from './domain-exception.filter.js';

@Module({
  providers: [
    { provide: APP_PIPE, useValue: new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }) },
    { provide: APP_FILTER, useClass: DomainExceptionFilter },
  ],
})
export class WebAdapterModule {}
```

Registered as providers (not `app.useGlobal*()`), so controller slice tests that import `WebAdapterModule` get the production pipeline.

## TransactionPort

```typescript
// src/shared/application/port/out/transaction.port.ts
export abstract class TransactionPort {
  /** Runs `work` in one database transaction: commit on success, rollback on throw. */
  abstract run<T>(work: () => Promise<T>): Promise<T>;
}

// src/shared/adapter/out/persistence/cls-transaction.adapter.ts
import { Injectable } from '@nestjs/common';
import { TransactionHost } from '@nestjs-cls/transactional';
import type { TransactionalAdapterTypeOrm } from '@nestjs-cls/transactional-adapter-typeorm';
import { TransactionPort } from '../../../application/port/out/transaction.port.js';

@Injectable()
export class ClsTransactionAdapter implements TransactionPort {
  constructor(private readonly txHost: TransactionHost<TransactionalAdapterTypeOrm>) {}

  run<T>(work: () => Promise<T>): Promise<T> {
    return this.txHost.withTransaction(work);
  }
}
```

`@nestjs-cls/transactional` keeps the transactional `EntityManager` in **AsyncLocalStorage**. Every persistence adapter that reads `txHost.tx` joins the transaction automatically, however deep in the call stack it runs.

## DatabaseModule

```bash
npm i nestjs-cls @nestjs-cls/transactional @nestjs-cls/transactional-adapter-typeorm
```

Check that the installed versions list your NestJS and TypeORM majors in their peer dependencies.

```typescript
// src/shared/adapter/out/persistence/database.module.ts
import { Global, Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { getDataSourceToken, TypeOrmModule } from '@nestjs/typeorm';
import { ClsPluginTransactional } from '@nestjs-cls/transactional';
import { TransactionalAdapterTypeOrm } from '@nestjs-cls/transactional-adapter-typeorm';
import { ClsModule } from 'nestjs-cls';
import { TransactionPort } from '../../../application/port/out/transaction.port.js';
import { ClsTransactionAdapter } from './cls-transaction.adapter.js';

@Global()
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        url: config.getOrThrow<string>('DATABASE_URL'),
        autoLoadEntities: true, // contexts register their own entities with forFeature()
        synchronize: false,
      }),
    }),
    ClsModule.forRoot({
      global: true,
      middleware: { mount: true }, // one CLS context per HTTP request
      plugins: [
        new ClsPluginTransactional({
          imports: [TypeOrmModule],
          adapter: new TransactionalAdapterTypeOrm({ dataSourceToken: getDataSourceToken() }),
        }),
      ],
    }),
  ],
  providers: [{ provide: TransactionPort, useClass: ClsTransactionAdapter }],
  exports: [TransactionPort],
})
export class DatabaseModule {}
```

`TransactionPort.run()` also works outside an HTTP request (cron jobs, broker consumers, tests) - the reference implementation's rollback test calls it directly, without a request. Use `@UseCls()` from `nestjs-cls` only when a non-HTTP entry point needs other per-request CLS values (correlation id, user).

## AppModule

```typescript
// src/app.module.ts - root composition: infrastructure + contexts
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, validationSchema: envSchema }),
    CqrsModule.forRoot(),   // once, if any context uses style B or the EventBus
    DatabaseModule,
    WebAdapterModule,
    AccountModule,
    NotificationModule,
  ],
})
export class AppModule {}
```

## Context Configuration

```typescript
// src/account/account.config.ts
import { registerAs } from '@nestjs/config';

export const accountConfig = registerAs('account', () => ({
  transferThreshold: BigInt(process.env.ACCOUNT_TRANSFER_THRESHOLD ?? '1000000'),
}));
```

Validated at boot (env schema in `src/config`), injected with `@Inject(accountConfig.KEY)` (`import type { ConfigType }`), and converted in the module's `useFactory` into a plain object such as `MoneyTransferProperties` before it reaches the core.
