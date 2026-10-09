# Web Layer - Controllers & REST APIs

## REST Controller Pattern

```typescript
import {
  Body, Controller, Delete, Get, HttpCode, HttpStatus, Param, ParseUUIDPipe, Patch, Post, Query,
} from '@nestjs/common';
import { CreateOrderDto } from './dto/create-order.dto.js';
import { ListOrdersQuery } from './dto/list-orders.query.js';
import { OrderResponse } from './dto/order.response.js';
import { UpdateOrderDto } from './dto/update-order.dto.js';
import { OrdersService } from './orders.service.js';
import type { Page } from '../common/page.js';

@Controller({ path: 'orders', version: '1' })
export class OrdersController {
  constructor(private readonly orders: OrdersService) {}

  @Get()
  list(@Query() query: ListOrdersQuery): Promise<Page<OrderResponse>> {
    return this.orders.list(query);
  }

  @Get(':id')
  get(@Param('id', ParseUUIDPipe) id: string): Promise<OrderResponse> {
    return this.orders.get(id);
  }

  @Post()
  create(@Body() dto: CreateOrderDto): Promise<OrderResponse> {
    return this.orders.create(dto);
  }

  @Patch(':id')
  update(@Param('id', ParseUUIDPipe) id: string, @Body() dto: UpdateOrderDto): Promise<OrderResponse> {
    return this.orders.update(id, dto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseUUIDPipe) id: string): Promise<void> {
    return this.orders.remove(id);
  }
}
```

**Rules:**
- Controllers are thin: bind input, call one service method, return a DTO
- Use built-in pipes (`ParseIntPipe`, `ParseUUIDPipe`, `ParseEnumPipe`) for path params
- Use the platform-agnostic API; inject `@Res()` only for streaming (then use `passthrough: true`)
- Express 5 route syntax: named wildcards (`'files/*path'`), optional segments (`'/:id{/:slug}'`)

## Request DTOs with Validation

```typescript
import { Type } from 'class-transformer';
import {
  ArrayMinSize, IsEmail, IsInt, IsOptional, IsString, IsUUID, Max, Min, ValidateNested,
} from 'class-validator';
import { PartialType } from '@nestjs/swagger';

export class OrderLineDto {
  @IsUUID()
  productId!: string;

  @IsInt()
  @Min(1)
  quantity!: number;
}

export class CreateOrderDto {
  @IsEmail()
  customerEmail!: string;

  @ValidateNested({ each: true })
  @Type(() => OrderLineDto) // required for nested validation
  @ArrayMinSize(1)
  lines!: OrderLineDto[];

  @IsOptional()
  @IsString()
  note?: string;
}

// All fields optional, validation rules inherited (also keeps OpenAPI metadata)
export class UpdateOrderDto extends PartialType(CreateOrderDto) {}

export class ListOrdersQuery {
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  page = 0;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  size = 20;
}
```

### Standard Schema Alternative (NestJS 12)

```typescript
import { StandardSchemaValidationPipe } from '@nestjs/common';
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z.email(),
  name: z.string().min(1).max(100),
});
export type CreateUserDto = z.infer<typeof createUserSchema>;

// main/app.setup.ts
app.useGlobalPipes(new StandardSchemaValidationPipe());

// controller
@Post()
create(@Body({ schema: createUserSchema }) dto: CreateUserDto) { /* ... */ }
```

Pick ONE validation style per project. `class-validator` integrates best with the Swagger CLI plugin; Zod gives a single source for types and validation.

## Response DTOs

```typescript
import type { Order } from '../order.entity.js';

export class OrderResponse {
  constructor(
    readonly id: string,
    readonly status: string,
    readonly totalCents: number,
    readonly createdAt: string,
  ) {}

  static from(order: Order): OrderResponse {
    return new OrderResponse(order.id, order.status, order.totalCents, order.createdAt.toISOString());
  }
}

// src/common/page.ts
export interface Page<T> {
  items: T[];
  page: number;
  size: number;
  total: number;
}
```

Explicit mapping beats `ClassSerializerInterceptor` + `@Exclude()`: new entity columns never leak by accident.

## Exception Handling

### Built-in HTTP Exceptions

| Exception | Status |
|-----------|--------|
| `BadRequestException` | 400 |
| `UnauthorizedException` | 401 |
| `ForbiddenException` | 403 |
| `NotFoundException` | 404 |
| `ConflictException` | 409 |
| `UnprocessableEntityException` | 422 |
| `InternalServerErrorException` | 500 |

### Domain Exceptions + Filter

```typescript
// Keep services HTTP-agnostic when they are reused by queues/microservices
export class DomainError extends Error {
  constructor(message: string, readonly code: 'NOT_FOUND' | 'CONFLICT' | 'INVALID_STATE', options?: ErrorOptions) {
    super(message, options);
    this.name = new.target.name;
  }
}

import { type ArgumentsHost, Catch, type ExceptionFilter, HttpStatus } from '@nestjs/common';
import type { Request, Response } from 'express';

const STATUS: Record<DomainError['code'], HttpStatus> = {
  NOT_FOUND: HttpStatus.NOT_FOUND,
  CONFLICT: HttpStatus.CONFLICT,
  INVALID_STATE: HttpStatus.UNPROCESSABLE_ENTITY,
};

@Catch(DomainError)
export class DomainExceptionFilter implements ExceptionFilter {
  catch(error: DomainError, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const status = STATUS[error.code];
    ctx.getResponse<Response>().status(status).json({
      statusCode: status,
      error: error.code,
      message: error.message,
      path: ctx.getRequest<Request>().url,
      timestamp: new Date().toISOString(),
    });
  }
}

// app.module.ts providers: [{ provide: APP_FILTER, useClass: DomainExceptionFilter }]
```

**Error response shape** (keep consistent across the API):
```json
{ "statusCode": 404, "error": "NOT_FOUND", "message": "Order 42 not found", "path": "/api/v1/orders/42", "timestamp": "2026-01-29T10:15:30.000Z" }
```

## Custom Validation

```typescript
import { registerDecorator, type ValidationOptions } from 'class-validator';

export function IsFutureDate(options?: ValidationOptions): PropertyDecorator {
  return (target, propertyKey) =>
    registerDecorator({
      name: 'isFutureDate',
      target: target.constructor,
      propertyName: propertyKey as string,
      options: { message: '$property must be in the future', ...options },
      validator: {
        validate: (value: unknown) => value instanceof Date && value.getTime() > Date.now(),
      },
    });
}

export class CreateReservationDto {
  @Type(() => Date)
  @IsFutureDate()
  startsAt!: Date;
}
```

## Custom Param Decorator

```typescript
import { createParamDecorator, type ExecutionContext } from '@nestjs/common';
import type { Request } from 'express';

export interface AuthUser { id: string; roles: string[] }

export const CurrentUser = createParamDecorator(
  (_: unknown, ctx: ExecutionContext): AuthUser => ctx.switchToHttp().getRequest<Request & { user: AuthUser }>().user,
);

// usage
@Get('me')
me(@CurrentUser() user: AuthUser) { return user; }
```

## Interceptors

```typescript
import { type CallHandler, type ExecutionContext, Injectable, type NestInterceptor, RequestTimeoutException } from '@nestjs/common';
import { catchError, type Observable, throwError, timeout, TimeoutError } from 'rxjs';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(_: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      timeout(5_000),
      catchError((err: unknown) =>
        throwError(() => (err instanceof TimeoutError ? new RequestTimeoutException() : err)),
      ),
    );
  }
}
```

Use interceptors for cross-cutting concerns: timing, timeouts, caching, response envelopes.

## HTTP Clients for External APIs

```typescript
import { Injectable, BadGatewayException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class PaymentClient {
  private readonly baseUrl: string;

  constructor(config: ConfigService) {
    this.baseUrl = config.getOrThrow<string>('PAYMENT_API_URL');
  }

  async charge(orderId: string, amountCents: number): Promise<{ id: string }> {
    const res = await fetch(`${this.baseUrl}/charges`, {
      method: 'POST',
      headers: { 'content-type': 'application/json', 'idempotency-key': orderId },
      body: JSON.stringify({ orderId, amountCents }),
      signal: AbortSignal.timeout(3_000), // always bound external calls
    });
    if (!res.ok) throw new BadGatewayException(`Payment API responded ${res.status}`);
    return (await res.json()) as { id: string };
  }
}
```

Node's native `fetch` is enough for most cases; use `@nestjs/axios` (`HttpModule`) when you need interceptors or Axios-specific features. Add retries/circuit breaking from `references/cloud.md`.

## OpenAPI (Swagger)

```typescript
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

const config = new DocumentBuilder().setTitle('Orders API').setVersion('1.0').addBearerAuth().build();
SwaggerModule.setup('docs', app, () => SwaggerModule.createDocument(app, config));
```

Enable the CLI plugin in `nest-cli.json` so DTO properties are documented without extra decorators:
```json
{ "compilerOptions": { "plugins": ["@nestjs/swagger"] } }
```

## CORS Configuration

```typescript
app.enableCors({
  origin: config.getOrThrow<string>('CORS_ORIGINS').split(','), // explicit allow-list
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  credentials: true,
  maxAge: 3600,
});
```

## Quick Reference

| Need | Use |
|------|-----|
| Validate body/query | DTO class + global `ValidationPipe` |
| Parse path params | `ParseIntPipe`, `ParseUUIDPipe` |
| Partial update DTO | `PartialType()` from `@nestjs/swagger` |
| Nested objects | `@ValidateNested()` + `@Type(() => Child)` |
| Map errors to HTTP | `HttpException` subclasses or an exception filter |
| Cross-cutting logic | Interceptor |
| Current user | Custom param decorator |
| API docs | `@nestjs/swagger` + CLI plugin |
| Versioning | `app.enableVersioning({ type: VersioningType.URI })` |
