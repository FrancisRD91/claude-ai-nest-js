---
name: nestjs
description: NestJS 12 development with TypeScript - REST APIs, TypeORM, validation, security, testing (Vitest, supertest), configuration and cloud-native patterns. Use for building Node.js backend applications with NestJS.
metadata:
  version: "1.0.0"
  domain: backend
  triggers: NestJS, Nest, TypeScript REST API, Node.js backend, TypeORM, Guards, Interceptors, Pipes, Microservices Node
  role: specialist
  scope: implementation
  output-format: code
---

# NestJS Skill

NestJS 12 development with focus on modular architecture and production-ready code.

## Core Workflow

1. **Analyze** - Understand requirements, identify modules, APIs, data models
2. **Design** - Plan module boundaries and DTOs, confirm design before coding
3. **Implement** - Feature modules, constructor injection, DTO validation
4. **Secure** - Global auth guard, roles, helmet, rate limiting; verify tests pass
5. **Test** - Unit + e2e tests; run `npm test` and `npm run test:e2e` and confirm all pass
6. **Deploy** - Terminus health checks; validate `GET /health/ready` returns `{"status":"ok", ...}`

## Baseline Stack

| Concern | Choice |
|---------|--------|
| Runtime | Node.js Active LTS (24+) |
| Framework | NestJS 12, Express 5 platform (Fastify for high throughput) |
| Language | TypeScript 6, `strict: true` |
| Modules | ESM (`"type": "module"`, `"module": "nodenext"`) |
| Data | TypeORM 1.x + `@nestjs/typeorm` (>= 11.0.1), PostgreSQL |
| Validation | `class-validator` + `class-transformer` (or Standard Schema, e.g. Zod) |
| Config | `@nestjs/config` with a Zod env schema |
| Tests | Vitest + `unplugin-swc`, supertest, Testcontainers |
| Docs | `@nestjs/swagger` |
| Health | `@nestjs/terminus` |

## ESM Rules (NestJS 12 default)

- Relative imports end with `.js`, even in `.ts` files: `import { AppModule } from './app.module.js';`
- Use `import.meta.dirname` instead of `__dirname`
- Keep `experimentalDecorators` and `emitDecoratorMetadata` enabled in tsconfig
- Never use `import type` for classes used as constructor parameter types or DTOs - DI and validation need the runtime value
- Type entity relation properties as `Relation<T>` (from `typeorm`) to avoid circular-import errors between entities

## Quick Start Templates

### Entity
```typescript
// src/products/product.entity.ts
import { Column, CreateDateColumn, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity('products')
export class Product {
  @PrimaryGeneratedColumn()
  id!: number;

  @Column({ length: 200 })
  name!: string;

  @Column('integer')
  priceCents!: number; // money as integer minor units: no float rounding, no numeric-as-string

  @CreateDateColumn({ type: 'timestamptz' })
  createdAt!: Date;
}
```

### DTOs
```typescript
// src/products/dto/create-product.dto.ts
import { IsInt, IsNotEmpty, IsString, MaxLength, Min } from 'class-validator';

export class CreateProductDto {
  @IsString()
  @IsNotEmpty()
  @MaxLength(200)
  name!: string;

  @IsInt()
  @Min(0)
  priceCents!: number;
}

// src/products/dto/product.response.ts
import type { Product } from '../product.entity.js';

export class ProductResponse {
  constructor(
    readonly id: number,
    readonly name: string,
    readonly priceCents: number,
  ) {}

  static from(product: Product): ProductResponse {
    return new ProductResponse(product.id, product.name, product.priceCents);
  }
}
```

### Service
```typescript
// src/products/products.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { ILike, Repository } from 'typeorm';
import { CreateProductDto } from './dto/create-product.dto.js';
import { Product } from './product.entity.js';

@Injectable()
export class ProductsService {
  constructor(@InjectRepository(Product) private readonly products: Repository<Product>) {}

  search(name = ''): Promise<Product[]> {
    return this.products.find({
      where: { name: ILike(`%${name}%`) },
      order: { id: 'ASC' },
      take: 100,
    });
  }

  async findOne(id: number): Promise<Product> {
    const product = await this.products.findOneBy({ id });
    if (!product) throw new NotFoundException(`Product ${id} not found`);
    return product;
  }

  create(dto: CreateProductDto): Promise<Product> {
    return this.products.save(this.products.create(dto));
  }
}
```

### REST Controller
```typescript
// src/products/products.controller.ts
import { Body, Controller, Get, Param, ParseIntPipe, Post, Query } from '@nestjs/common';
import { CreateProductDto } from './dto/create-product.dto.js';
import { ProductResponse } from './dto/product.response.js';
import { ProductsService } from './products.service.js';

@Controller({ path: 'products', version: '1' })
export class ProductsController {
  constructor(private readonly service: ProductsService) {}

  @Get()
  async search(@Query('name') name?: string): Promise<ProductResponse[]> {
    return (await this.service.search(name)).map(ProductResponse.from);
  }

  @Get(':id')
  async findOne(@Param('id', ParseIntPipe) id: number): Promise<ProductResponse> {
    return ProductResponse.from(await this.service.findOne(id));
  }

  @Post() // 201 Created by default
  async create(@Body() dto: CreateProductDto): Promise<ProductResponse> {
    return ProductResponse.from(await this.service.create(dto));
  }
}
```

### Feature Module
```typescript
// src/products/products.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Product } from './product.entity.js';
import { ProductsController } from './products.controller.js';
import { ProductsService } from './products.service.js';

@Module({
  imports: [TypeOrmModule.forFeature([Product])],
  controllers: [ProductsController],
  providers: [ProductsService],
})
export class ProductsModule {}
```

### Root Module + Validated Config
```typescript
// src/config/env.schema.ts
import { z } from 'zod';

export const envSchema = z.object({
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.url(),
});

// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { envSchema } from './config/env.schema.js';
import { ProductsModule } from './products/products.module.js';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true, validationSchema: envSchema }),
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        url: config.getOrThrow<string>('DATABASE_URL'),
        autoLoadEntities: true,
        synchronize: false, // schema changes go through migrations
      }),
    }),
    ProductsModule,
  ],
})
export class AppModule {}
```

### Bootstrap (shared setup for app and e2e tests)
```typescript
// src/app.setup.ts
import { type INestApplication, ValidationPipe, VersioningType } from '@nestjs/common';
import helmet from 'helmet';

export function configureApp(app: INestApplication): INestApplication {
  app.use(helmet());
  app.setGlobalPrefix('api', { exclude: ['health/live', 'health/ready'] }); // probes stay at /health/*
  app.enableVersioning({ type: VersioningType.URI }); // -> /api/v1/products
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }));
  app.enableShutdownHooks();
  return app;
}

// src/main.ts
import { ConfigService } from '@nestjs/config';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module.js';
import { configureApp } from './app.setup.js';

const app = configureApp(await NestFactory.create(AppModule));
await app.listen(app.get(ConfigService).getOrThrow<number>('PORT'));
```

### Exception Filter (map persistence errors)
```typescript
// src/common/filters/database-exception.filter.ts
import { type ArgumentsHost, Catch, ConflictException, NotFoundException } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';
import { EntityNotFoundError, QueryFailedError } from 'typeorm';

const PG_UNIQUE_VIOLATION = '23505';

@Catch(QueryFailedError, EntityNotFoundError)
export class DatabaseExceptionFilter extends BaseExceptionFilter {
  override catch(exception: QueryFailedError | EntityNotFoundError, host: ArgumentsHost): void {
    if (exception instanceof EntityNotFoundError) {
      return super.catch(new NotFoundException('Resource not found'), host);
    }
    if ((exception.driverError as { code?: string }).code === PG_UNIQUE_VIOLATION) {
      return super.catch(new ConflictException('Resource already exists'), host);
    }
    super.catch(exception, host); // unknown DB errors -> 500 without leaking SQL
  }
}

// register in AppModule providers: { provide: APP_FILTER, useClass: DatabaseExceptionFilter }
```

### Unit Test (Vitest)
```typescript
// src/products/products.service.spec.ts
import { NotFoundException } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { Product } from './product.entity.js';
import { ProductsService } from './products.service.js';

describe('ProductsService', () => {
  const repo = { find: vi.fn(), findOneBy: vi.fn(), create: vi.fn(), save: vi.fn() };
  let service: ProductsService;

  beforeEach(async () => {
    vi.resetAllMocks();
    const moduleRef = await Test.createTestingModule({
      providers: [ProductsService, { provide: getRepositoryToken(Product), useValue: repo }],
    }).compile();
    service = moduleRef.get(ProductsService);
  });

  it('returns the product when it exists', async () => {
    repo.findOneBy.mockResolvedValue({ id: 1, name: 'Widget', priceCents: 1000 });
    await expect(service.findOne(1)).resolves.toMatchObject({ name: 'Widget' });
  });

  it('throws NotFoundException when the product is missing', async () => {
    repo.findOneBy.mockResolvedValue(null);
    await expect(service.findOne(99)).rejects.toBeInstanceOf(NotFoundException);
  });
});
```

## Reference Guide

Load detailed patterns based on context:

| Topic | Reference | When to Load |
|-------|-----------|-------------|
| Web/REST | `references/web.md` | Controllers, DTOs, validation, filters, interceptors, OpenAPI |
| Data Access | `references/data.md` | TypeORM setup, repositories, transactions, migrations |
| Security | `references/security.md` | JWT, guards, roles, helmet, throttling, CORS |
| Cloud/Config | `references/cloud.md` | Config, health, microservices, resilience, Docker, Kubernetes |
| Testing | `references/testing.md` | Vitest, e2e with supertest, Testcontainers |

## Constraints

### MUST DO
- Constructor injection with `private readonly` dependencies
- Global `ValidationPipe` with `whitelist` + `forbidNonWhitelisted` + `transform`
- DTO class for every request body and query object
- Return response DTOs, never TypeORM entities
- Multi-step writes in one transaction (`dataSource.transaction(...)`)
- Validate env vars at boot with a schema in `ConfigModule.forRoot`
- Throw Nest `HttpException` subclasses (or map domain errors in a filter)
- `app.enableShutdownHooks()` for graceful shutdown
- Await or explicitly `void` every Promise

### MUST NOT DO
- Use `any` or disable `strict`
- Skip input validation on endpoints
- Expose entities or internal error details in responses
- Use `synchronize: true` outside tests
- Read `process.env` directly in services (inject `ConfigService` or a `registerAs` config)
- Block the event loop (sync fs/crypto, CPU-heavy loops) in request handlers
- Use the service-locator pattern (`moduleRef.get`) instead of injection
- Create circular module dependencies (`forwardRef` is a design smell)
- Hardcode URLs, credentials, environment values

## Architecture Patterns

**Project Structure:**
```
src/
├── main.ts                 # Bootstrap only
├── app.setup.ts            # Global pipes, filters, versioning (shared with e2e tests)
├── app.module.ts           # Root module
├── config/                 # Env schema, registerAs() namespaces
├── common/                 # Filters, guards, interceptors, decorators
├── database/               # data-source.ts (CLI) and migrations/
├── health/                 # Terminus health checks
└── products/               # One folder per feature module
    ├── dto/
    ├── product.entity.ts
    ├── products.controller.ts
    ├── products.module.ts
    ├── products.service.ts
    └── products.service.spec.ts
test/
└── products.e2e-spec.ts
```

**Layering:**
- Controller -> Service -> Repository
- Controller handles HTTP, DTO validation and response mapping
- Service handles business logic and transactions
- Repository (TypeORM) handles persistence

**Request Lifecycle:**
Middleware -> Guards -> Interceptors (before) -> Pipes -> Handler -> Interceptors (after) -> Exception filters

## Common Decorators

| Decorator | Purpose |
|-----------|---------|
| `@Module()` | Group controllers/providers, declare imports/exports |
| `@Controller()` | HTTP route handler class |
| `@Injectable()` | Provider managed by the DI container |
| `@Get()`/`@Post()`/`@Put()`/`@Patch()`/`@Delete()` | Route handlers |
| `@Body()`/`@Param()`/`@Query()` | Bind request data (optionally with pipes) |
| `@HttpCode()` | Override default status code |
| `@UseGuards()`/`@UseInterceptors()`/`@UseFilters()` | Attach request lifecycle components |
| `@InjectRepository()` | Inject a TypeORM repository |
| `@Inject(TOKEN)` | Inject a custom provider by token |

## Knowledge Base

NestJS 12, TypeScript 6, Node.js 24 LTS, Express 5, Fastify, TypeORM 1.x, PostgreSQL, class-validator, Zod (Standard Schema), @nestjs/config, @nestjs/jwt, @nestjs/throttler, @nestjs/terminus, @nestjs/swagger, @nestjs/microservices, @nestjs/event-emitter, @nestjs/bullmq, Vitest, supertest, Testcontainers, npm
