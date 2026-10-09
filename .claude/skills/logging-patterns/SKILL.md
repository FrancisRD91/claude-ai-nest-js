---
name: logging-patterns
description: NestJS logging best practices - built-in Logger, structured JSON logs (ConsoleLogger JSON mode, nestjs-pino), and request correlation IDs. Includes AI-friendly log formats for Claude Code debugging. Use when user asks about logging, debugging application flow, or analyzing logs.
---

# Logging Patterns Skill

Effective logging for NestJS applications with focus on structured, AI-parsable formats.

## When to Use
- User says "add logging" / "improve logs" / "debug this"
- Analyzing application flow from logs
- Setting up structured logging (JSON)
- Request tracing with correlation IDs
- AI/Claude Code needs to analyze application behavior

---

## AI-Friendly Logging

> **Key insight:** JSON logs are better for AI analysis - faster parsing, fewer tokens, direct field access.

### Why JSON for AI/Claude Code?

```
# Text format - AI must "interpret" the string
[Nest] 4120  - 01/29/2026, 10:15:30 AM     LOG [OrdersService] Order 12345 created for user-789, total: 9999

# JSON format - AI extracts fields directly
{"level":"info","time":"2026-01-29T10:15:30.123Z","context":"OrdersService","orderId":12345,"userId":"user-789","totalCents":9999,"msg":"Order created"}
```

| Aspect | Text | JSON |
|--------|------|------|
| Parsing | Regex/interpretation | Direct field access |
| Token usage | Higher (repeated patterns) | Lower (structured) |
| Error extraction | Parse stack trace text | `err` field |
| Filtering | grep patterns | `jq` queries |

### Recommended Setup for AI-Assisted Development

- JSON to stdout by default (local, CI, production)
- `LOG_FORMAT=pretty` only when a human reads the console
- A request ID on every line, returned to clients as `x-request-id`

### Log Format Optimized for AI Analysis

```json
{
  "level": "info",
  "time": "2026-01-29T10:15:30.123Z",
  "context": "OrdersService",
  "msg": "Order created",
  "req": { "id": "req-abc123", "method": "POST", "url": "/api/v1/orders" },
  "orderId": 12345,
  "userId": "user-789",
  "durationMs": 45,
  "step": "payment_completed"
}
```

**Key fields for AI debugging:**
- `req.id` - group all logs from the same request
- `step` - track progress through a flow
- `durationMs` / `responseTime` - identify slow operations
- `level` - quick filter for errors

### Reading Logs with AI/Claude Code

```bash
# Get recent errors
cat app.log | jq 'select(.level == "error")' | tail -20

# Follow a specific request
cat app.log | jq 'select(.req.id == "req-abc123")'

# Find slow requests (pino-http adds responseTime in ms)
cat app.log | jq 'select(.responseTime > 1000)'

# Running in Docker Compose
docker compose logs app --no-log-prefix | jq 'select(.level == "error")'
```

AI can then:
1. Parse JSON directly (no guessing)
2. Follow request flow via `req.id`
3. Identify exactly where errors occurred
4. Measure timing between steps

---

## Option A: Built-in ConsoleLogger (zero dependencies)

```typescript
// src/main.ts
import { ConsoleLogger } from '@nestjs/common';

const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({
    json: process.env.LOG_FORMAT !== 'pretty',
    colors: process.env.LOG_FORMAT === 'pretty',
  }),
});
```

NestJS 12 adds structured params: plain objects passed after the message land in the same JSON entry (under `params`, or at the root with `flattenParams: true`):

```typescript
this.logger.log('Order created', { orderId: order.id, totalCents: order.totalCents });
```

Good for small services and workers. It does not add request IDs on its own - use Option B for HTTP APIs.

## Option B: nestjs-pino (recommended for HTTP APIs)

```bash
npm i nestjs-pino pino-http pino
npm i -D pino-pretty
```

Check that the `nestjs-pino` release you install lists your NestJS major in its peer dependencies.

```typescript
// src/app.module.ts
import { randomUUID } from 'node:crypto';
import { ConfigService } from '@nestjs/config';
import { LoggerModule } from 'nestjs-pino';
import { stdTimeFunctions } from 'pino';

LoggerModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    pinoHttp: {
      level: config.get<string>('LOG_LEVEL', 'info'),
      timestamp: stdTimeFunctions.isoTime,
      formatters: { level: (label: string) => ({ level: label }) }, // "info" instead of 30
      genReqId: (req, res) => {
        const id = (req.headers['x-request-id'] as string | undefined) ?? randomUUID();
        res.setHeader('x-request-id', id);
        return id;
      },
      redact: ['req.headers.authorization', 'req.headers.cookie', 'res.headers["set-cookie"]'],
      autoLogging: { ignore: (req) => req.url?.startsWith('/health') ?? false },
      transport: config.get('LOG_FORMAT') === 'pretty' ? { target: 'pino-pretty' } : undefined,
    },
  }),
}),

// src/main.ts
import { Logger } from 'nestjs-pino';

const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(Logger));
```

Every log line written during a request - including from the standard `Logger` - now carries `req.id` (propagated with AsyncLocalStorage), and each request gets one access log with `responseTime`.

### Structured Fields with PinoLogger

```typescript
import { Injectable } from '@nestjs/common';
import { InjectPinoLogger, PinoLogger } from 'nestjs-pino';

@Injectable()
export class OrdersService {
  constructor(@InjectPinoLogger(OrdersService.name) private readonly logger: PinoLogger) {}

  async placeOrder(dto: CreateOrderDto, userId: string): Promise<Order> {
    const started = performance.now();
    const order = await this.create(dto);
    this.logger.info(
      { orderId: order.id, userId, totalCents: order.totalCents, step: 'order_created', durationMs: Math.round(performance.now() - started) },
      'Order created',
    );
    return order;
  }
}
```

### Add Context for the Rest of the Request

```typescript
// e.g. in JwtAuthGuard after authentication
this.pinoLogger.assign({ userId: payload.sub }); // appears on every later log line of this request
```

### Usage

```bash
# Default: JSON (for AI, CI/CD, production)
npm run start:dev

# Human-readable when needed
LOG_FORMAT=pretty npm run start:dev          # bash
$env:LOG_FORMAT="pretty"; npm run start:dev  # PowerShell
```

---

## Logger Basics

### Logger Declaration

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class OrdersService {
  private readonly logger = new Logger(OrdersService.name); // context = class name

  // use `this.logger` directly - never console.log
}
```

### Pass Data, Not Strings

```typescript
// ✅ GOOD: fields stay queryable
this.logger.log('Processing order', { orderId, userId });

// ❌ BAD: data buried in text, cost paid even when the level is disabled
this.logger.debug(`Processing order ${JSON.stringify(order)} for user ${userId}`);
```

### Logging Errors

```typescript
// Nest Logger: message + stack
this.logger.error(`Payment failed for order ${orderId}`, err instanceof Error ? err.stack : String(err));

// PinoLogger: the `err` key is serialized with type, message and stack
this.logger.error({ err, orderId }, 'Payment failed');
```

Unhandled exceptions are already logged by Nest's exception layer - do not log and re-throw the same error at every level.

### Log Levels

| Level (Nest / pino) | Use For |
|---------------------|---------|
| `fatal` | Process cannot continue |
| `error` | Request/job failed, needs attention |
| `warn` | Unexpected but handled (retry, fallback) |
| `log` / `info` | Business events (order created, payment completed) |
| `debug` | Diagnostic detail for development |
| `verbose` / `trace` | Very fine-grained flow |

Production default: `info`. Raise to `debug` temporarily via `LOG_LEVEL`, never by code change.

---

## What Not to Log

- Passwords, tokens, API keys, `authorization`/`cookie` headers (use `redact`)
- Full request/response bodies (PII, size)
- Personal data beyond IDs (emails, addresses) unless required and approved
- Health-check access logs (noise)

## Quick Reference

| Need | Use |
|------|-----|
| Logger in a provider | `new Logger(ClassName.name)` |
| JSON without dependencies | `new ConsoleLogger({ json: true })` |
| Request IDs + access logs | `nestjs-pino` with `genReqId` |
| Structured fields | `logger.log('msg', { ... })` / `pinoLogger.info({ ... }, 'msg')` |
| Per-request context | `PinoLogger.assign({ userId })` |
| Hide secrets | `pinoHttp.redact` |
| Pretty local output | `LOG_FORMAT=pretty` (+ `pino-pretty`) |
| Analyze logs | `jq 'select(.level == "error")'` |
