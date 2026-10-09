# Testing Strategy

| Test type | Tooling | Example file |
|---|---|---|
| Domain unit | Vitest, plain `new`, no Nest | `account.spec.ts` |
| Use case unit | Typed mocks of ports (`vi.fn<Port['method']>()`), spies on real entities, hand-written fakes (`FakeTransaction`) | `send-money.service.spec.ts` |
| CQRS handler unit | Call `execute()` directly; in-memory fakes for ports | `send-transfer-notifications.handler.spec.ts` |
| Web adapter slice | `Test.createTestingModule({ imports: [WebAdapterModule], controllers, providers: [{ provide: Port, useValue }] })` + supertest | `send-money.controller.spec.ts` |
| Persistence adapter | Testing module with the real `DatabaseModule` + PostgreSQL via Testcontainers | `account-persistence.adapter.spec.ts` |
| System test | `Test.createTestingModule({ imports: [AppModule] })` + supertest | `test/send-money.e2e-spec.ts` |
| Provider override | `.overrideProvider(X).useValue(…)` | system test (threshold) |
| Architecture | dependency-cruiser + lint restrictions | `npm run lint:arch` |
| Test data | Builders in `test/fixtures/` | `defaultAccount().withBalance(500n).build()` |

## Guidelines

- **Most tests are plain unit tests** of domain objects and services - no Nest container, milliseconds each
- **Mock ports, never adapters or libraries**; type the mocks so signatures stay in sync
- **Spy on real domain objects** (`vi.spyOn(account, 'withdraw')`) instead of mocking entities wholesale
- **Prefer fakes for stateful ports** (in-memory repositories, a transaction that records begin/commit/rollback) - they test behaviour, not call sequences
- **Every adapter has an integration test against the real technology**: real HTTP pipeline for controllers, real database for persistence
- **Asynchronous effects** (event handlers) are asserted by polling with `vi.waitFor()`, never with sleeps
- **Each system test gets a fresh application and database state** to avoid hidden coupling

## Domain Unit Test

```typescript
// src/account/domain/model/account.spec.ts
import { describe, expect, it } from 'vitest';
import { AccountId } from './account-id.js';
import { Account } from './account.js';
import { Money } from './money.js';

describe('Account', () => {
  it('withdraws when the balance covers the amount', () => {
    const account = Account.withId(new AccountId(1), Money.of(100n));

    expect(account.withdraw(Money.of(60n))).toBe(true);
    expect(account.getBalance().equals(Money.of(40n))).toBe(true);
  });

  it('refuses to overdraw and keeps the balance', () => {
    const account = Account.withId(new AccountId(1), Money.of(100n));

    expect(account.withdraw(Money.of(101n))).toBe(false);
    expect(account.getBalance().equals(Money.of(100n))).toBe(true);
  });
});
```

## Use Case Unit Test (no Nest container)

```typescript
// test/support/fake-transaction.ts
import type { TransactionPort } from '../../src/shared/application/port/out/transaction.port.js';

export class FakeTransaction implements TransactionPort {
  readonly log: string[] = [];

  async run<T>(work: () => Promise<T>): Promise<T> {
    this.log.push('begin');
    try {
      const result = await work();
      this.log.push('commit');
      return result;
    } catch (err) {
      this.log.push('rollback');
      throw err;
    }
  }
}

// src/account/application/service/send-money.service.spec.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { FakeTransaction } from '../../../../test/support/fake-transaction.js';
// … domain, port and service imports

describe('SendMoneyService', () => {
  const sourceId = new AccountId(1);
  const targetId = new AccountId(2);
  const loadAccountPort = { loadAccount: vi.fn<LoadAccountPort['loadAccount']>() };
  const lockAccountPort = { lockAccount: vi.fn<LockAccountPort['lockAccount']>() };
  const updateAccountStatePort = { updateBalance: vi.fn<UpdateAccountStatePort['updateBalance']>() };
  const publishAccountEventPort = { publish: vi.fn<PublishAccountEventPort['publish']>() };
  let transaction: FakeTransaction;
  let service: SendMoneyService;

  beforeEach(() => {
    vi.resetAllMocks();
    transaction = new FakeTransaction();
    service = new SendMoneyService(
      loadAccountPort,
      lockAccountPort,
      updateAccountStatePort,
      publishAccountEventPort,
      transaction,
      new MoneyTransferProperties(Money.of(1_000n)),
    );
  });

  function givenAccounts(sourceBalance: bigint, targetBalance: bigint) {
    const source = Account.withId(sourceId, Money.of(sourceBalance));
    const target = Account.withId(targetId, Money.of(targetBalance));
    loadAccountPort.loadAccount.mockImplementation(async (id) => (id.equals(sourceId) ? source : target));
    return { source, target };
  }

  it('transfers inside the transaction and publishes after commit', async () => {
    const { source, target } = givenAccounts(500n, 0n);
    publishAccountEventPort.publish.mockImplementation(async () => void transaction.log.push('publish'));

    await expect(service.sendMoney(new SendMoneyCommand(sourceId, targetId, Money.of(300n)))).resolves.toBe(true);

    expect(source.getBalance().equals(Money.of(200n))).toBe(true);
    expect(target.getBalance().equals(Money.of(300n))).toBe(true);
    expect(transaction.log).toEqual(['begin', 'commit', 'publish']);
  });

  it('returns false and publishes nothing when funds are insufficient', async () => {
    givenAccounts(100n, 0n);

    await expect(service.sendMoney(new SendMoneyCommand(sourceId, targetId, Money.of(300n)))).resolves.toBe(false);

    expect(updateAccountStatePort.updateBalance).not.toHaveBeenCalled();
    expect(publishAccountEventPort.publish).not.toHaveBeenCalled();
  });

  it('rejects transfers above the threshold before opening a transaction', async () => {
    await expect(service.sendMoney(new SendMoneyCommand(sourceId, targetId, Money.of(5_000n))))
      .rejects.toBeInstanceOf(ThresholdExceededException);
    expect(loadAccountPort.loadAccount).not.toHaveBeenCalled();
    expect(transaction.log).toEqual([]);
  });

  it('rolls back when an account does not exist', async () => {
    loadAccountPort.loadAccount.mockResolvedValue(null);

    await expect(service.sendMoney(new SendMoneyCommand(sourceId, targetId, Money.of(10n))))
      .rejects.toBeInstanceOf(AccountNotFoundException);
    expect(transaction.log).toEqual(['begin', 'rollback']);
  });
});
```

## Input Model Unit Test

```typescript
// src/account/application/port/in/send-money.command.spec.ts
describe('SendMoneyCommand', () => {
  it('accepts valid input', () => {
    expect(() => new SendMoneyCommand(new AccountId(42), new AccountId(43), Money.of(10n))).not.toThrow();
  });

  it('rejects non-positive money', () => {
    expect(() => new SendMoneyCommand(new AccountId(42), new AccountId(43), Money.of(-10n))).toThrow(InvalidInputException);
  });

  it('rejects transfers to the same account', () => {
    expect(() => new SendMoneyCommand(new AccountId(42), new AccountId(42), Money.of(10n))).toThrow(InvalidInputException);
  });
  // no "null id" test: with strict: true the compiler already rejects null here
});
```

## CQRS Handler Unit Test

```typescript
it('stores and delivers a credit notification', async () => {
  const notifications = new InMemoryNotificationsAdapter(); // fake implementing Save/LoadNotificationsPort
  const deliver = { deliver: vi.fn<DeliverNotificationPort['deliver']>() };
  const handler = new SendTransferNotificationsHandler(
    { loadBalance: vi.fn<LoadAccountBalancePort['loadBalance']>().mockResolvedValue(800n) },
    notifications,
    deliver,
  );

  await handler.execute(new SendTransferNotificationsCommand(1, 2, 300n));

  expect(await notifications.loadByAccount(2)).toHaveLength(1);
  expect(deliver.deliver).toHaveBeenCalledOnce();
});
```

## Event Listener (Driving Adapter) Unit Test

```typescript
// src/notification/adapter/in/event/money-transferred.listener.spec.ts
import { Logger } from '@nestjs/common';
import type { CommandBus } from '@nestjs/cqrs';
import { beforeEach, describe, expect, it, vi } from 'vitest';
// … event, command and listener imports

describe('MoneyTransferredListener', () => {
  const event = new MoneyTransferredEvent(1, 2, '500', '2026-01-01T00:00:00.000Z');
  const execute = vi.fn<CommandBus['execute']>();
  let listener: MoneyTransferredListener;

  beforeEach(() => {
    vi.resetAllMocks();
    listener = new MoneyTransferredListener({ execute } as unknown as CommandBus);
  });

  it('translates the integration event into a command of this context', async () => {
    await listener.handle(event);

    expect(execute).toHaveBeenCalledWith(new SendTransferNotificationsCommand(1, 2, 500n));
  });

  it('never lets a failure escape to the publisher', async () => {
    execute.mockRejectedValue(new Error('smtp down'));
    const logError = vi.spyOn(Logger.prototype, 'error').mockImplementation(() => undefined);

    await expect(listener.handle(event)).resolves.toBeUndefined();
    expect(logError).toHaveBeenCalled();
  });
});
```

## Web Adapter Slice Test

```typescript
// src/account/adapter/in/web/send-money.controller.spec.ts
import type { INestApplication } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import request from 'supertest';
import { afterAll, beforeAll, beforeEach, describe, it, vi } from 'vitest';
import { WebAdapterModule } from '../../../../shared/adapter/in/web/web-adapter.module.js';
// … use case, exception and controller imports

describe('SendMoneyController', () => {
  const sendMoneyUseCase = { sendMoney: vi.fn<SendMoneyUseCase['sendMoney']>() };
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [WebAdapterModule], // production ValidationPipe + DomainExceptionFilter
      controllers: [SendMoneyController],
      providers: [{ provide: SendMoneyUseCase, useValue: sendMoneyUseCase }],
    }).compile();
    app = moduleRef.createNestApplication();
    await app.init();
  });

  afterAll(() => app.close());
  beforeEach(() => vi.resetAllMocks());

  const transfer = (body: object) => request(app.getHttpServer()).post('/accounts/1/transfers').send(body);

  it('maps the request to a SendMoneyCommand and answers 204', async () => {
    sendMoneyUseCase.sendMoney.mockResolvedValue(true);

    await transfer({ targetAccountId: 2, amount: '500' }).expect(204);

    expect(sendMoneyUseCase.sendMoney).toHaveBeenCalledWith(
      new SendMoneyCommand(new AccountId(1), new AccountId(2), Money.of(500n)),
    );
  });

  it('answers 422 when the source account cannot cover the amount', async () => {
    sendMoneyUseCase.sendMoney.mockResolvedValue(false);

    const response = await transfer({ targetAccountId: 2, amount: '500' }).expect(422);

    expect(response.body).toMatchObject({ error: 'INSUFFICIENT_FUNDS' });
  });

  it('rejects syntactically invalid input before reaching the use case (400)', async () => {
    await transfer({ targetAccountId: 2, amount: 500 }).expect(400); // number instead of string
    await transfer({ targetAccountId: 2, amount: '5.00' }).expect(400);

    expect(sendMoneyUseCase.sendMoney).not.toHaveBeenCalled();
  });

  it('rejects semantically invalid commands (400)', async () => {
    const response = await transfer({ targetAccountId: 1, amount: '500' }).expect(400); // same account

    expect(response.body).toMatchObject({ error: 'INVALID_INPUT' });
    expect(sendMoneyUseCase.sendMoney).not.toHaveBeenCalled();
  });

  it('translates domain exceptions into HTTP responses', async () => {
    sendMoneyUseCase.sendMoney.mockRejectedValue(new AccountNotFoundException(new AccountId(2)));

    const response = await transfer({ targetAccountId: 2, amount: '500' }).expect(404);

    expect(response.body).toEqual({ statusCode: 404, error: 'ACCOUNT_NOT_FOUND', message: 'Account 2 not found' });
  });
});
```

Money must survive the trip to JSON intact - test a value above `Number.MAX_SAFE_INTEGER`:

```typescript
// src/account/adapter/in/web/get-account-balance.controller.spec.ts
it('serialises the balance as a decimal string (bigint-safe)', async () => {
  getAccountBalanceUseCase.getAccountBalance.mockResolvedValue(Money.of(9_007_199_254_740_993n));

  const response = await request(app.getHttpServer()).get('/accounts/7/balance').expect(200);

  expect(response.body).toEqual({ accountId: 7, balance: '9007199254740993' });
  expect(getAccountBalanceUseCase.getAccountBalance).toHaveBeenCalledWith(new GetAccountBalanceQuery(new AccountId(7)));
});
```

## Persistence Adapter Integration Test

```typescript
// src/account/adapter/out/persistence/account-persistence.adapter.spec.ts
describe('AccountPersistenceAdapter', () => {
  let container: StartedPostgreSqlContainer;
  let moduleRef: TestingModule;
  let adapter: AccountPersistenceAdapter;

  beforeAll(async () => {
    container = await new PostgreSqlContainer('postgres:18-alpine').start();
    process.env.DATABASE_URL = container.getConnectionUri();
    moduleRef = await Test.createTestingModule({
      imports: [
        ConfigModule.forRoot({ isGlobal: true, ignoreEnvFile: true }),
        DatabaseModule,                                   // the real one
        TypeOrmModule.forFeature([AccountOrmEntity]),
      ],
      providers: [AccountPersistenceAdapter],
    }).compile();
    await moduleRef.get(DataSource).synchronize();        // throwaway schema for the test database
    adapter = moduleRef.get(AccountPersistenceAdapter);
  }, 120_000);

  afterAll(async () => {
    await moduleRef?.close();
    await container?.stop();
  });

  it('loads and maps an account', async () => {
    const { id } = await moduleRef.get(DataSource).getRepository(AccountOrmEntity).save({ balance: 500n });

    const account = await adapter.loadAccount(new AccountId(id));

    expect(account?.getBalance().equals(Money.of(500n))).toBe(true);
  });

  it('returns null for an unknown account', async () => {
    await expect(adapter.loadAccount(new AccountId(9_999))).resolves.toBeNull();
  });

  it('writes inside the transaction opened through TransactionPort', async () => {
    const { id } = await moduleRef.get(DataSource).getRepository(AccountOrmEntity).save({ balance: 500n });
    const transaction = moduleRef.get(TransactionPort);

    await expect(
      transaction.run(async () => {
        await adapter.updateBalance(Account.withId(new AccountId(id), Money.of(1n)));
        throw new Error('boom');
      }),
    ).rejects.toThrow('boom');

    expect((await adapter.loadAccount(new AccountId(id)))?.getBalance().equals(Money.of(500n))).toBe(true); // rolled back
  });
});
```

## System Test

```typescript
// test/send-money.e2e-spec.ts
let container: StartedPostgreSqlContainer;
let app: INestApplication;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:18-alpine').start();
  process.env.DATABASE_URL = container.getConnectionUri();
}, 120_000);

afterAll(() => container?.stop());

beforeEach(async () => {
  const moduleRef = await Test.createTestingModule({ imports: [AppModule] })
    .overrideProvider(MoneyTransferProperties)
    .useValue(new MoneyTransferProperties(Money.of(1_000n)))
    .compile();
  app = configureApp(moduleRef.createNestApplication()); // same prefix/versioning as main.ts
  await app.init();
  const dataSource = moduleRef.get(DataSource);
  await dataSource.synchronize(true); // fresh schema per test
  await seedAccounts(dataSource, [defaultAccount().withId(1).withBalance(500n), defaultAccount().withId(2).withBalance(0n)]);
});

afterEach(() => app.close());

it('transfers money and notifies the target account', async () => {
  await request(app.getHttpServer()).post('/api/v1/accounts/1/transfers').send({ targetAccountId: 2, amount: '300' }).expect(204);

  const balance = await request(app.getHttpServer()).get('/api/v1/accounts/1/balance').expect(200);
  expect(balance.body.balance).toBe('200');

  // the notification is created asynchronously by the event listener -> poll, don't sleep
  await vi.waitFor(async () => {
    const res = await request(app.getHttpServer()).get('/api/v1/accounts/2/notifications').expect(200);
    expect(res.body).toHaveLength(1);
  });
});

it('refuses a transfer the source account cannot cover (422) and changes nothing', async () => {
  await request(app.getHttpServer()).post('/api/v1/accounts/1/transfers').send({ targetAccountId: 2, amount: '501' }).expect(422);

  const balance = await request(app.getHttpServer()).get('/api/v1/accounts/1/balance').expect(200);
  expect(balance.body.balance).toBe('500');
});

it('refuses transfers above the configured threshold (422)', async () => {
  const response = await request(app.getHttpServer())
    .post('/api/v1/accounts/1/transfers')
    .send({ targetAccountId: 2, amount: '5000' })
    .expect(422);

  expect(response.body).toMatchObject({ error: 'TRANSFER_THRESHOLD_EXCEEDED' });
});

it('answers 404 for unknown accounts and 400 for invalid input', async () => {
  await request(app.getHttpServer()).post('/api/v1/accounts/1/transfers').send({ targetAccountId: 404, amount: '1' }).expect(404);
  await request(app.getHttpServer()).post('/api/v1/accounts/1/transfers').send({ targetAccountId: 2, amount: '-5' }).expect(400);
  await request(app.getHttpServer()).get('/api/v1/accounts/abc/balance').expect(400);
});
```

One PostgreSQL container per test file; a fresh schema and seed per test.

## Test Data Builders

```typescript
// test/fixtures/account.fixture.ts
export class AccountBuilder {
  private id = 42;
  private balance = 999n;

  withId(id: number): this { this.id = id; return this; }
  withBalance(balance: bigint): this { this.balance = balance; return this; }

  build(): Account {
    return Account.withId(new AccountId(this.id), Money.of(this.balance));
  }
}

export const defaultAccount = (): AccountBuilder => new AccountBuilder();

// test/fixtures/account.seed.ts - seeding goes through the ORM entity of the adapter
export async function seedAccounts(dataSource: DataSource, builders: AccountBuilder[]): Promise<void> {
  const rows = builders
    .map((builder) => builder.build())
    .map((account) => ({ id: account.id!.value, balance: account.getBalance().amount }));
  await dataSource.getRepository(AccountOrmEntity).save(rows);
}
```
