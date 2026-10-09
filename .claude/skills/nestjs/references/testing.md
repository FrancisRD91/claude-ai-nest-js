# Testing - Vitest, supertest & Testcontainers

NestJS 12 ESM projects use **Vitest**. Vitest transpiles with esbuild, which does not emit decorator metadata, so Nest DI needs the SWC plugin.

Install: `npm i -D vitest unplugin-swc @swc/core @vitest/coverage-v8 supertest @types/supertest @testcontainers/postgresql`

## Configuration

```typescript
// vitest.config.ts - unit tests (src/**/*.spec.ts)
import { resolve } from 'node:path';
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
    include: ['src/**/*.spec.ts'],
    mockReset: true,
    coverage: {
      provider: 'v8',
      include: ['src/**/*.ts'],
      exclude: ['src/main.ts', 'src/**/*.module.ts', 'src/database/**'],
      thresholds: { lines: 85, branches: 80 },
    },
  },
  plugins: [swc.vite({ module: { type: 'es6' } })], // builds tests with SWC (decorator metadata)
  resolve: { alias: { src: resolve(import.meta.dirname, './src') } },
});

// vitest.config.e2e.ts - e2e and integration tests (test/**)
import swc from 'unplugin-swc';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    root: './',
    include: ['test/**/*.e2e-spec.ts', 'test/**/*.int-spec.ts'],
    mockReset: true,
    hookTimeout: 120_000, // container start-up
  },
  plugins: [swc.vite({ module: { type: 'es6' } })],
});
```

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:cov": "vitest run --coverage",
    "test:debug": "vitest --inspect-brk --no-file-parallelism",
    "test:e2e": "vitest run --config ./vitest.config.e2e.ts"
  }
}
```

## Unit Testing Services

```typescript
// src/orders/orders.service.spec.ts
import { ConflictException } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { Order } from './order.entity.js';
import { OrdersService } from './orders.service.js';
import { PaymentClient } from '../payments/payment.client.js';

describe('OrdersService', () => {
  const orders = { findOneBy: vi.fn(), save: vi.fn() };
  const payments = { charge: vi.fn() };
  const manager = { createQueryBuilder: vi.fn(), save: vi.fn(), create: vi.fn() };
  const dataSource = { transaction: vi.fn((work: (m: typeof manager) => unknown) => work(manager)) };
  let service: OrdersService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
      providers: [
        OrdersService,
        { provide: getRepositoryToken(Order), useValue: orders },
        { provide: DataSource, useValue: dataSource },
        { provide: PaymentClient, useValue: payments },
      ],
    }).compile();
    service = moduleRef.get(OrdersService);
  });

  it('charges the customer for a pending order', async () => {
    orders.findOneBy.mockResolvedValue({ id: 'o1', status: 'PENDING', totalCents: 4200 });
    payments.charge.mockResolvedValue({ id: 'ch_1' });

    await service.pay('o1');

    expect(payments.charge).toHaveBeenCalledWith('o1', 4200);
  });

  it('refuses to pay an order twice', async () => {
    orders.findOneBy.mockResolvedValue({ id: 'o1', status: 'PAID', totalCents: 4200 });

    await expect(service.pay('o1')).rejects.toBeInstanceOf(ConflictException);
    expect(payments.charge).not.toHaveBeenCalled();
  });
});
```

## Overriding Providers in a Real Module Graph

```typescript
const moduleRef = await Test.createTestingModule({ imports: [OrdersModule] })
  .overrideProvider(PaymentClient)
  .useValue({ charge: vi.fn().mockResolvedValue({ id: 'ch_1' }) })
  .overrideGuard(JwtAuthGuard)
  .useValue({ canActivate: () => true })
  .compile();
```

## E2E Testing the HTTP Layer (supertest)

```typescript
// test/products.e2e-spec.ts
import { type INestApplication, NotFoundException } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import request from 'supertest'; // default import - supertest is CommonJS
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest';
import { configureApp } from '../src/app.setup.js';
import { ProductsController } from '../src/products/products.controller.js';
import { ProductsService } from '../src/products/products.service.js';

describe('Products API (e2e)', () => {
  let app: INestApplication;
  const service = { search: vi.fn(), findOne: vi.fn(), create: vi.fn() };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [ProductsController],
      providers: [{ provide: ProductsService, useValue: service }],
    }).compile();
    app = configureApp(moduleRef.createNestApplication()); // same pipes/prefix/versioning as production
    await app.init();
  });

  afterAll(() => app.close());

  it('POST /api/v1/products creates a product', async () => {
    service.create.mockResolvedValue({ id: 1, name: 'Widget', priceCents: 1000 });

    const res = await request(app.getHttpServer())
      .post('/api/v1/products')
      .send({ name: 'Widget', priceCents: 1000 })
      .expect(201);

    expect(res.body).toEqual({ id: 1, name: 'Widget', priceCents: 1000 });
  });

  it('POST /api/v1/products rejects an invalid payload', async () => {
    await request(app.getHttpServer()).post('/api/v1/products').send({ name: '', priceCents: -1 }).expect(400);
    expect(service.create).not.toHaveBeenCalled();
  });

  it('POST /api/v1/products rejects unknown properties', async () => {
    await request(app.getHttpServer())
      .post('/api/v1/products')
      .send({ name: 'Widget', priceCents: 1000, isAdmin: true })
      .expect(400);
  });

  it('GET /api/v1/products/:id returns 404 for a missing product', async () => {
    service.findOne.mockRejectedValue(new NotFoundException());
    await request(app.getHttpServer()).get('/api/v1/products/99').expect(404);
  });

  it('GET /api/v1/products/:id returns 400 for a non-numeric id', async () => {
    await request(app.getHttpServer()).get('/api/v1/products/abc').expect(400);
  });
});
```

Add the auth guards (`APP_GUARD` providers) to the testing module when the routes under test are protected.

## Integration Testing with Testcontainers

```typescript
// test/products.int-spec.ts
import { Test, type TestingModule } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { afterAll, beforeAll, describe, expect, it } from 'vitest';
import { ProductsModule } from '../src/products/products.module.js';
import { ProductsService } from '../src/products/products.service.js';

describe('ProductsService (integration)', () => {
  let container: StartedPostgreSqlContainer;
  let moduleRef: TestingModule;
  let service: ProductsService;

  beforeAll(async () => {
    container = await new PostgreSqlContainer('postgres:18-alpine').start();
    moduleRef = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'postgres',
          url: container.getConnectionUri(),
          autoLoadEntities: true,
          synchronize: true, // acceptable in throwaway test databases only
        }),
        ProductsModule,
      ],
    }).compile();
    service = moduleRef.get(ProductsService);
  });

  afterAll(async () => {
    await moduleRef?.close();
    await container?.stop();
  });

  it('persists and reads a product', async () => {
    const created = await service.create({ name: 'Widget', priceCents: 1000 });
    await expect(service.findOne(created.id)).resolves.toMatchObject({ name: 'Widget', priceCents: 1000 });
  });

  it('searches case-insensitively', async () => {
    await service.create({ name: 'Blue Lamp', priceCents: 2500 });
    await expect(service.search('blue')).resolves.toEqual(
      expect.arrayContaining([expect.objectContaining({ name: 'Blue Lamp' })]),
    );
  });
});
```

Testcontainers needs a Docker daemon: locally Docker Desktop/Podman, in CircleCI a `machine` executor.

## Time and Randomness

```typescript
beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(new Date('2026-01-29T10:00:00Z'));
});
afterEach(() => vi.useRealTimers());

it('expires reservations after 15 minutes', () => {
  const reservation = service.reserve('sku-1');
  vi.advanceTimersByTime(15 * 60_000);
  expect(service.isActive(reservation.id)).toBe(false);
});
```

## Quick Reference

| Need | Use |
|------|-----|
| Build a DI container | `Test.createTestingModule({...}).compile()` |
| Mock a repository | `{ provide: getRepositoryToken(Entity), useValue: mock }` |
| Replace a provider/guard | `.overrideProvider(X).useValue(...)`, `.overrideGuard(G).useValue(...)` |
| HTTP assertions | `supertest` against `app.getHttpServer()` |
| Real database | `@testcontainers/postgresql` |
| Control time | `vi.useFakeTimers()`, `vi.setSystemTime()` |
| Coverage gate | `coverage.thresholds` in `vitest.config.ts` |

## Testing Best Practices

- Every endpoint and service method gets at least one positive and one negative test
- Reuse `configureApp()` in e2e tests so they exercise the production pipes, prefix and versioning
- Name tests after behavior ("refuses to pay an order twice"), not methods
- Reset mocks between tests (`mockReset: true` or `vi.resetAllMocks()`)
- Mock collaborators you own at the boundary; use Testcontainers instead of mocking the database
- Always close the app/testing module and stop containers in `afterAll`
- Never sleep in tests; use fake timers or poll with a timeout
- Keep unit tests fast (no network, no Docker) so `npm test` runs on every save
