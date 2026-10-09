---
name: design-patterns
description: Common design patterns with TypeScript and NestJS examples (Factory, Builder, Strategy, Observer, Decorator, Adapter, etc.). Use when user asks "implement pattern", "use factory", "strategy pattern", or when designing extensible components.
---

# Design Patterns Skill

Quick reference for common design patterns in TypeScript, using NestJS dependency injection where it fits.

## When to Use
- User asks to implement a specific pattern
- Designing extensible/flexible components
- Refactoring rigid code

## Quick Reference: When to Use What

| Problem | Pattern | Use When |
|---------|---------|----------|
| Complex object construction | **Builder** | Many parameters, some optional, invariants to check |
| Create objects without specifying class | **Factory** | Type determined at runtime |
| Multiple algorithms, swap at runtime | **Strategy** | Behavior varies by context |
| Add behavior without changing class | **Decorator** | Dynamic composition needed |
| Notify multiple objects of changes | **Observer** | One-to-many dependency |
| Convert incompatible interfaces | **Adapter** | Integrate legacy/3rd party code |

---

## Creational Patterns

### Builder
**Problem:** Telescoping constructors, many optional parameters

In TypeScript, an options object usually replaces a Builder:

```typescript
// ✅ Options object - the idiomatic default
interface CreateUserOptions {
  name: string;   // required
  email: string;  // required
  age?: number;   // optional
}

export class User {
  readonly name: string;
  readonly email: string;
  readonly age: number;

  constructor({ name, email, age = 0 }: CreateUserOptions) {
    this.name = name;
    this.email = email;
    this.age = age;
  }
}

const user = new User({ name: 'John', email: 'john@example.com', age: 30 });
```

Use a real Builder when construction is stepwise or must validate the combination:

```typescript
// ✅ Builder pattern
export class ReportBuilder {
  private readonly sections: string[] = [];
  private title?: string;

  withTitle(title: string): this {
    this.title = title;
    return this;
  }

  addSection(section: string): this {
    this.sections.push(section);
    return this;
  }

  build(): Report {
    if (!this.title) throw new Error('Report title is required');
    if (this.sections.length === 0) throw new Error('Report needs at least one section');
    return new Report(this.title, [...this.sections]);
  }
}

const report = new ReportBuilder().withTitle('Q1').addSection('Revenue').build();
```

### Factory
**Problem:** Create objects without knowing exact class upfront

```typescript
// ✅ Plain factory
export interface Notification {
  send(message: string): Promise<void>;
}

export function createNotification(type: 'email' | 'sms' | 'push'): Notification {
  switch (type) {
    case 'email': return new EmailNotification();
    case 'sms': return new SmsNotification();
    case 'push': return new PushNotification();
  }
}

// ✅ NestJS version - preferred: implementations are injectable providers
export interface NotificationSender {
  readonly channel: 'email' | 'sms' | 'push';
  send(to: string, message: string): Promise<void>;
}

export const NOTIFICATION_SENDERS = Symbol('NOTIFICATION_SENDERS');

@Injectable()
export class NotificationFactory {
  private readonly senders: Map<string, NotificationSender>;

  constructor(@Inject(NOTIFICATION_SENDERS) senders: NotificationSender[]) {
    this.senders = new Map(senders.map((sender) => [sender.channel, sender]));
  }

  get(channel: string): NotificationSender {
    const sender = this.senders.get(channel);
    if (!sender) throw new BadRequestException(`Unsupported channel: ${channel}`);
    return sender;
  }
}

@Module({
  providers: [
    EmailSender,
    SmsSender,
    PushSender,
    {
      provide: NOTIFICATION_SENDERS,
      useFactory: (...senders: NotificationSender[]) => senders,
      inject: [EmailSender, SmsSender, PushSender],
    },
    NotificationFactory,
  ],
  exports: [NotificationFactory],
})
export class NotificationsModule {}
```

---

## Behavioral Patterns

### Strategy
**Problem:** Multiple algorithms for same operation, choose at runtime

```typescript
// ✅ Strategy pattern
export interface PaymentStrategy {
  pay(amountCents: number): Promise<string>;
}

export class CreditCardPayment implements PaymentStrategy {
  constructor(private readonly cardToken: string) {}

  async pay(amountCents: number): Promise<string> {
    // charge the card...
    return `card-${amountCents}`;
  }
}

export class Checkout {
  constructor(private readonly strategy: PaymentStrategy) {}

  complete(totalCents: number): Promise<string> {
    return this.strategy.pay(totalCents);
  }
}

// Usage
await new Checkout(new CreditCardPayment('tok_123')).complete(9999);

// Functional variant - a strategy is often just a function
type PricingStrategy = (baseCents: number) => number;

const regular: PricingStrategy = (base) => base;
const blackFriday: PricingStrategy = (base) => Math.round(base * 0.7);
```

In NestJS, register each strategy as a provider and select it with the Factory above (by key) or bind one implementation to an injection token per environment.

### Observer
**Problem:** Notify multiple objects when state changes

```typescript
// ✅ @nestjs/event-emitter (preferred for in-process events)
// app.module.ts: imports: [EventEmitterModule.forRoot()]

export class OrderPlacedEvent {
  constructor(readonly orderId: string, readonly totalCents: number) {}
}

@Injectable()
export class OrdersService {
  constructor(private readonly events: EventEmitter2) {}

  async placeOrder(dto: CreateOrderDto): Promise<Order> {
    const order = await this.save(dto);
    this.events.emit('order.placed', new OrderPlacedEvent(order.id, order.totalCents));
    return order;
  }
}

@Injectable()
export class InventoryListener {
  @OnEvent('order.placed')
  async handleOrderPlaced(event: OrderPlacedEvent): Promise<void> {
    // Reduce inventory
  }
}

@Injectable()
export class EmailListener {
  @OnEvent('order.placed', { async: true })
  async handleOrderPlaced(event: OrderPlacedEvent): Promise<void> {
    // Send email
  }
}
```

In-process events are lost on crash and listener errors are suppressed by default - log them. When delivery must be guaranteed, use a queue (`@nestjs/bullmq`) or a transactional outbox.

---

## Structural Patterns

### Decorator
**Problem:** Add behavior dynamically without modifying class

```typescript
// ✅ Decorator pattern
export interface Coffee {
  description(): string;
  costCents(): number;
}

export class SimpleCoffee implements Coffee {
  description(): string { return 'Coffee'; }
  costCents(): number { return 200; }
}

export class MilkDecorator implements Coffee {
  constructor(private readonly coffee: Coffee) {}

  description(): string { return `${this.coffee.description()}, Milk`; }
  costCents(): number { return this.coffee.costCents() + 50; }
}

// Usage
let coffee: Coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
```

**NestJS equivalents:** interceptors wrap handlers with extra behavior (caching, timing, retries), and `applyDecorators()` composes metadata decorators:

```typescript
export function AdminOnly() {
  return applyDecorators(Roles(['admin']), ApiBearerAuth(), ApiForbiddenResponse());
}
```

### Adapter
**Problem:** Make incompatible interfaces work together

```typescript
// ✅ Adapter pattern - the app depends on a port it owns
export interface MailPort {
  send(to: string, subject: string, html: string): Promise<void>;
}
export const MAIL_PORT = Symbol('MAIL_PORT');

// Adapter around a third-party SDK with a different API
@Injectable()
export class VendorMailAdapter implements MailPort {
  constructor(private readonly client: VendorMailClient) {}

  async send(to: string, subject: string, html: string): Promise<void> {
    await this.client.messages.create({ recipient: to, title: subject, body: { html } });
  }
}

// Wiring - swap the adapter without touching consumers
@Module({ providers: [{ provide: MAIL_PORT, useClass: VendorMailAdapter }], exports: [MAIL_PORT] })
export class MailModule {}

// Usage
constructor(@Inject(MAIL_PORT) private readonly mail: MailPort) {}
```

---

## Pattern Selection Guide

| Situation | Pattern |
|-----------|---------|
| Object creation is complex | Options object, Builder, Factory |
| Need to add features dynamically | Decorator, Nest interceptor |
| Multiple implementations of algorithm | Strategy (+ injection token) |
| React to state changes | Observer (`@nestjs/event-emitter`) |
| Integrate with legacy/3rd-party code | Adapter behind a port interface |

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| Module-level singletons (`export const db = new ...`) | Global state, hard to test | Providers + dependency injection |
| Service locator (`moduleRef.get()` everywhere) | Hidden dependencies | Constructor injection |
| Factory everywhere | Over-engineering | Simple `new` if type is known |
| Deep decorator/interceptor chains | Hard to debug | Composition, keep chains short |
| God module importing everything | Tight coupling, slow tests | Small feature modules with explicit exports |
