# Cloud Native - Configuration, Health, Microservices & Deployment

## Configuration

```typescript
// src/config/env.schema.ts
import { z } from 'zod';

export const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.url(),
  JWT_SECRET: z.string().min(32),
  REDIS_HOST: z.string().optional(),
  REDIS_PORT: z.coerce.number().int().default(6379),
});

// src/config/payment.config.ts - namespaced, typed config
import { registerAs } from '@nestjs/config';

export default registerAs('payment', () => ({
  baseUrl: process.env.PAYMENT_API_URL ?? 'http://localhost:8081',
  timeoutMs: Number(process.env.PAYMENT_TIMEOUT_MS ?? 3000),
}));

// app.module.ts
ConfigModule.forRoot({ isGlobal: true, cache: true, validationSchema: envSchema, load: [paymentConfig] });

// consumer
constructor(@Inject(paymentConfig.KEY) private readonly payment: ConfigType<typeof paymentConfig>) {}
```

**Rules:**
- The app must fail fast at boot when config is invalid
- Only `src/config/*` reads `process.env`; everything else injects config
- 12-factor: config from environment; `.env` files only for local development (`envFilePath`), never in images

## Health Checks - Terminus

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, TypeOrmHealthIndicator } from '@nestjs/terminus';
import { Public } from '../auth/public.decorator.js';

@Public()
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
  ) {}

  @Get('live') // process is up - no external dependencies here
  @HealthCheck()
  live() {
    return this.health.check([]);
  }

  @Get('ready') // can serve traffic
  @HealthCheck()
  ready() {
    return this.health.check([() => this.db.pingCheck('database')]);
  }
}

// src/health/health.module.ts
@Module({ imports: [TerminusModule.forRoot({ gracefulShutdownTimeoutMs: 5_000 })], controllers: [HealthController] })
export class HealthModule {}
```

### Custom Health Indicator (Terminus 11+ API)

```typescript
import { Inject, Injectable } from '@nestjs/common';
import type { ConfigType } from '@nestjs/config';
import { HealthIndicatorService } from '@nestjs/terminus';
import paymentConfig from '../config/payment.config.js';

@Injectable()
export class PaymentApiHealthIndicator {
  constructor(
    private readonly indicator: HealthIndicatorService,
    @Inject(paymentConfig.KEY) private readonly payment: ConfigType<typeof paymentConfig>,
  ) {}

  async check(key: string) {
    const check = this.indicator.check(key);
    try {
      const res = await fetch(`${this.payment.baseUrl}/health`, { signal: AbortSignal.timeout(1_000) });
      return res.ok ? check.up() : check.down({ status: res.status });
    } catch (err) {
      return check.down({ error: (err as Error).message });
    }
  }
}
```

The legacy `HealthIndicator` base class and `HealthCheckError` were removed in NestJS 12.

## Graceful Shutdown

```typescript
// main/app.setup.ts
app.enableShutdownHooks(); // SIGTERM -> onModuleDestroy -> beforeApplicationShutdown -> onApplicationShutdown

@Injectable()
export class OutboxPublisher implements OnApplicationShutdown {
  async onApplicationShutdown(signal?: string): Promise<void> {
    await this.flush(); // finish in-flight work, close clients
  }
}
```

NestJS 12's Express adapter drains in-flight requests on shutdown. Pair it with a Kubernetes `preStop` delay so the pod leaves the load balancer first.

## Microservices - @nestjs/microservices

```typescript
// main.ts - hybrid app: HTTP + message broker
const app = await NestFactory.create(AppModule);
const config = app.get(ConfigService);
app.connectMicroservice<MicroserviceOptions>({
  transport: Transport.KAFKA,
  options: {
    client: { brokers: config.getOrThrow<string>('KAFKA_BROKERS').split(',') },
    consumer: { groupId: 'orders-service' },
  },
});
await app.startAllMicroservices();
await app.listen(3000);

// consumer
@Controller()
export class PaymentEventsController {
  constructor(private readonly orders: OrdersService) {}

  @EventPattern('payment.completed') // fire-and-forget event
  async onPaymentCompleted(@Payload() event: PaymentCompletedEvent): Promise<void> {
    await this.orders.markPaid(event.orderId); // must be idempotent - brokers redeliver
  }

  @MessagePattern('orders.get') // request-response
  get(@Payload() id: string): Promise<OrderResponse> {
    return this.orders.get(id);
  }
}

// producer
@Module({
  imports: [ClientsModule.register([{ name: 'PAYMENTS', transport: Transport.KAFKA, options: { /* ... */ } }])],
})
export class OrdersModule {}

@Injectable()
export class PaymentsGateway implements OnModuleInit {
  constructor(@Inject('PAYMENTS') private readonly client: ClientKafka) {}

  async onModuleInit(): Promise<void> {
    this.client.subscribeToResponseOf('payment.status'); // required for send() with Kafka
    await this.client.connect();
  }

  requestPayment(orderId: string, amountCents: number): void {
    this.client.emit('payment.requested', { orderId, amountCents });
  }

  status(orderId: string): Promise<PaymentStatus> {
    return firstValueFrom(this.client.send<PaymentStatus>('payment.status', orderId).pipe(timeout(3_000)));
  }
}
```

Transports: TCP, Redis, NATS (`@nats-io/transport-node` in v12), MQTT, RabbitMQ, Kafka, gRPC. For guaranteed delivery from a DB transaction, write events to an **outbox** table in the same transaction and publish them asynchronously.

## Background Jobs & Scheduling

```typescript
// Queues (Redis-backed): npm i @nestjs/bullmq bullmq
BullModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    connection: { host: config.getOrThrow<string>('REDIS_HOST'), port: config.getOrThrow<number>('REDIS_PORT') },
  }),
});
BullModule.registerQueue({ name: 'emails' });

@Processor('emails')
export class EmailProcessor extends WorkerHost {
  async process(job: Job<{ to: string; template: string }>): Promise<void> { /* send */ }
}

// Cron: npm i @nestjs/schedule
@Cron('0 */5 * * * *')
async purgeExpiredReservations(): Promise<void> { /* runs on EVERY replica - use a queue or lock for singletons */ }
```

## Resilience - Retry, Circuit Breaker, Timeout

```typescript
// npm i cockatiel
import {
  circuitBreaker, ConsecutiveBreaker, ExponentialBackoff, handleAll, retry, timeout, TimeoutStrategy, wrap,
} from 'cockatiel';

const policy = wrap(
  retry(handleAll, { maxAttempts: 3, backoff: new ExponentialBackoff() }),
  circuitBreaker(handleAll, { halfOpenAfter: 10_000, breaker: new ConsecutiveBreaker(5) }),
  timeout(2_000, TimeoutStrategy.Aggressive),
);

@Injectable()
export class InventoryClient {
  private readonly baseUrl: string;

  constructor(config: ConfigService) {
    this.baseUrl = config.getOrThrow<string>('INVENTORY_API_URL');
  }

  getStock(sku: string): Promise<number> {
    return policy.execute(async ({ signal }) => {
      const res = await fetch(`${this.baseUrl}/stock/${encodeURIComponent(sku)}`, { signal });
      if (!res.ok) throw new Error(`Inventory responded ${res.status}`);
      return ((await res.json()) as { available: number }).available;
    });
  }
}
```

Only retry idempotent operations (or send an idempotency key).

## Caching

```typescript
// npm i @nestjs/cache-manager cache-manager
CacheModule.register({ isGlobal: true, ttl: 30_000 });

@UseInterceptors(CacheInterceptor) // GET responses only
@Get(':id')
get(@Param('id') id: string) { /* ... */ }
```

Use a shared store (Redis via Keyv) when running multiple replicas; in-memory caches diverge per pod.

## Observability

- **Logs:** structured JSON to stdout (see the `logging-patterns` skill)
- **Traces/metrics:** `@nestjs/observe` (NestJS 12, opt-in via the `instrument` option of `NestFactory.create`) or the OpenTelemetry Node SDK loaded before the app (`node --import ./dist/otel.js dist/main.js`)
- **Prometheus:** expose `/metrics` with `prom-client` (default Node.js metrics include event-loop lag and heap)

## Docker Configuration

```dockerfile
# Dockerfile (multi-stage)
FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:24-alpine
ENV NODE_ENV=production
WORKDIR /app
# package.json is required at runtime: it carries "type": "module"
COPY --from=build --chown=node:node /app/package.json ./
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:3000/health/live || exit 1
CMD ["node", "dist/main.js"]
```

```gitignore
# .dockerignore
node_modules
dist
coverage
.env*
.git
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://app:app@postgres:5432/app
      JWT_SECRET: ${JWT_SECRET:?set JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:18-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      retries: 10
    volumes:
      - pgdata:/var/lib/postgresql

volumes:
  pgdata:
```

## Kubernetes Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 3
  selector:
    matchLabels: { app: orders-api }
  template:
    metadata:
      labels: { app: orders-api }
    spec:
      terminationGracePeriodSeconds: 30
      securityContext: { runAsNonRoot: true }
      containers:
        - name: orders-api
          image: registry.example.com/orders-api:1.0.0
          ports: [{ containerPort: 3000 }]
          env:
            - name: NODE_OPTIONS
              value: "--max-old-space-size=384" # ~75% of the memory limit
          envFrom:
            - configMapRef: { name: orders-api-config }
            - secretRef: { name: orders-api-secrets }
          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits: { memory: 512Mi }
          startupProbe:
            httpGet: { path: /health/live, port: 3000 }
            failureThreshold: 30
            periodSeconds: 2
          livenessProbe:
            httpGet: { path: /health/live, port: 3000 }
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health/ready, port: 3000 }
            periodSeconds: 5
          lifecycle:
            preStop:
              exec: { command: ["sleep", "5"] }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
```

## Quick Reference

| Concern | Tool |
|---------|------|
| Config + validation | `@nestjs/config` + Zod schema |
| Health checks | `@nestjs/terminus` (`/health/live`, `/health/ready`) |
| Graceful shutdown | `enableShutdownHooks()` + lifecycle hooks |
| Messaging | `@nestjs/microservices` (Kafka, RabbitMQ, NATS, Redis, gRPC) |
| Background jobs | `@nestjs/bullmq` |
| Cron | `@nestjs/schedule` |
| Resilience | `cockatiel` (retry, circuit breaker, timeout) |
| Caching | `@nestjs/cache-manager` |
| Tracing/metrics | `@nestjs/observe` or OpenTelemetry, `prom-client` |
| Container | Multi-stage `node:24-alpine`, non-root, `node dist/main.js` |
