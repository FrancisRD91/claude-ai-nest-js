# Data Access - TypeORM 1.x

## Module Configuration

```typescript
// app.module.ts
TypeOrmModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService): TypeOrmModuleOptions => ({
    type: 'postgres',
    url: config.getOrThrow<string>('DATABASE_URL'),
    autoLoadEntities: true,            // entities registered via forFeature()
    synchronize: false,                // never in production - use migrations
    maxQueryExecutionTime: 200,        // log queries slower than 200 ms
    logging: config.get('DB_LOGGING') === 'true' ? ['query', 'error'] : ['error'],
    extra: { max: 10 },                // pg pool size per instance
  }),
}),
```

Requires `@nestjs/typeorm` >= 11.0.1 for TypeORM 1.x.

## Entity Pattern

```typescript
// src/common/audited.entity.ts
import { CreateDateColumn, UpdateDateColumn, VersionColumn } from 'typeorm';

export abstract class AuditedEntity {
  @CreateDateColumn({ type: 'timestamptz' })
  createdAt!: Date;

  @UpdateDateColumn({ type: 'timestamptz' })
  updatedAt!: Date;

  @VersionColumn()
  version!: number;
}

// src/orders/order.entity.ts
import { Column, Entity, Index, OneToMany, PrimaryGeneratedColumn, type Relation } from 'typeorm';
import { AuditedEntity } from '../common/audited.entity.js';
import { OrderLine } from './order-line.entity.js';

export enum OrderStatus {
  PENDING = 'PENDING',
  PAID = 'PAID',
  CANCELLED = 'CANCELLED',
}

@Entity('orders')
export class Order extends AuditedEntity {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Index()
  @Column({ length: 320 })
  customerEmail!: string;

  @Column({ type: 'enum', enum: OrderStatus, default: OrderStatus.PENDING })
  status!: OrderStatus;

  @Column('integer')
  totalCents!: number;

  @OneToMany(() => OrderLine, (line) => line.order, { cascade: ['insert'] })
  lines!: Relation<OrderLine[]>;
}

// src/orders/order-line.entity.ts
import { Column, Entity, ManyToOne, PrimaryGeneratedColumn, type Relation } from 'typeorm';
import { Order } from './order.entity.js';

@Entity('order_lines')
export class OrderLine {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @ManyToOne(() => Order, (order) => order.lines, { nullable: false, onDelete: 'CASCADE' })
  order!: Relation<Order>;

  @Column('uuid')
  productId!: string;

  @Column('integer')
  quantity!: number;
}
```

**Entity rules:**
- `Relation<T>` on relation properties - prevents ESM circular-import errors (`Cannot access 'Order' before initialization`)
- Arrow functions in relation decorators (`() => Order`) - never reference the class directly
- `timestamptz` for timestamps, integer minor units for money
- `bigint` and `numeric` columns come back as **strings** from `pg` - add a `transformer` if you need numbers
- Relations are not loaded unless requested; load them explicitly with `relations` or joins
- Index every foreign key and every column used in `WHERE`/`ORDER BY`

## Repository Usage

```typescript
@Injectable()
export class OrdersService {
  constructor(
    @InjectRepository(Order) private readonly orders: Repository<Order>,
    private readonly dataSource: DataSource,
  ) {}

  findWithLines(id: string): Promise<Order | null> {
    return this.orders.findOne({ where: { id }, relations: { lines: true } });
  }

  findByIds(ids: string[]): Promise<Order[]> {
    return this.orders.findBy({ id: In(ids) });
  }

  existsForCustomer(email: string): Promise<boolean> {
    return this.orders.exists({ where: { customerEmail: email } });
  }
}
```

### Custom Repository

`@EntityRepository` and `getCustomRepository` are gone in TypeORM 1.x - wrap the repository in a provider:

```typescript
@Injectable()
export class OrdersRepository {
  constructor(@InjectRepository(Order) private readonly repo: Repository<Order>) {}

  findPaidSince(since: Date): Promise<Order[]> {
    return this.repo.find({
      where: { status: OrderStatus.PAID, createdAt: MoreThanOrEqual(since) },
      order: { createdAt: 'DESC' },
      take: 500,
    });
  }
}
```

## Dynamic Filters (Specification-style)

```typescript
import { type FindOptionsWhere, ILike, In } from 'typeorm';

export interface OrderFilter { email?: string; statuses?: OrderStatus[] }

function toWhere(filter: OrderFilter): FindOptionsWhere<Order> {
  return {
    ...(filter.email && { customerEmail: ILike(`%${filter.email}%`) }),
    ...(filter.statuses?.length && { status: In(filter.statuses) }),
  };
}

const [items, total] = await this.orders.findAndCount({
  where: toWhere(filter),
  order: { createdAt: 'DESC', id: 'DESC' }, // deterministic order for paging
  skip: page * size,
  take: size,
});
```

Never pass `undefined`/`null` as a where value: TypeORM 1.x throws by default (older versions silently matched every row). Use `IsNull()` for `IS NULL`.

## QueryBuilder

```typescript
const rows = await this.orders
  .createQueryBuilder('o')
  .select('o.status', 'status')
  .addSelect('COUNT(*)::int', 'count')
  .addSelect('SUM(o.totalCents)::int', 'totalCents')
  .where('o.createdAt >= :since', { since })   // always bind parameters
  .groupBy('o.status')
  .getRawMany<{ status: OrderStatus; count: number; totalCents: number }>();
```

## Transaction Management

```typescript
async placeOrder(dto: CreateOrderDto): Promise<Order> {
  return this.dataSource.transaction(async (manager) => {
    for (const line of dto.lines) {
      const result = await manager
        .createQueryBuilder()
        .update(Product)
        .set({ stock: () => 'stock - :qty' })
        .where('id = :id AND stock >= :qty', { id: line.productId, qty: line.quantity })
        .execute();
      if (result.affected !== 1) throw new ConflictException(`Insufficient stock for ${line.productId}`);
    }
    const totalCents = await this.priceLines(manager, dto.lines);
    // use `manager`, NOT the injected repository - otherwise the write escapes the transaction
    return manager.save(manager.create(Order, { customerEmail: dto.customerEmail, lines: dto.lines, totalCents }));
  });
}

// Isolation level
await this.dataSource.transaction('SERIALIZABLE', async (manager) => { /* ... */ });
```

### Manual Control with QueryRunner

```typescript
const qr = this.dataSource.createQueryRunner();
await qr.connect();
await qr.startTransaction();
try {
  await qr.manager.save(order);
  await qr.manager.save(payment);
  await qr.commitTransaction();
} catch (err) {
  await qr.rollbackTransaction();
  throw err;
} finally {
  await qr.release(); // always release, or the pool leaks connections
}
```

Prefer `dataSource.transaction()`; use QueryRunner only when you need savepoints or connection-level settings.

## Migrations

```typescript
// src/database/data-source.ts - used only by the TypeORM CLI (run from the project root)
import 'reflect-metadata';
import { DataSource } from 'typeorm';

export default new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  entities: ['dist/**/*.entity.js'],
  migrations: ['dist/database/migrations/*.js'],
});
```

```json
{
  "scripts": {
    "migration:generate": "npm run build && typeorm migration:generate -d dist/database/data-source.js",
    "migration:create": "typeorm migration:create",
    "migration:run": "npm run build && typeorm migration:run -d dist/database/data-source.js",
    "migration:revert": "typeorm migration:revert -d dist/database/data-source.js"
  }
}
```

```bash
npm run migration:generate -- src/database/migrations/AddOrders
npm run migration:run
```

**Rules:**
- Review every generated migration; generated SQL can drop and recreate columns on renames
- Never edit a migration that has already run in a shared environment - add a new one
- Run migrations as a separate deploy step (CI job, Kubernetes Job), not on every app start
- Expand/contract for zero-downtime: add nullable column -> backfill -> enforce constraint -> remove old column later

## Projections

```typescript
// Partial entity
const emails = await this.orders.find({ select: { id: true, customerEmail: true }, where: { status: OrderStatus.PAID } });

// Raw DTO-shaped rows
const summaries = await this.orders
  .createQueryBuilder('o')
  .select(['o.id AS id', 'o.status AS status', 'o.totalCents AS "totalCents"'])
  .getRawMany<{ id: string; status: OrderStatus; totalCents: number }>();
```

## Upserts and Bulk Writes

```typescript
// save() issues a SELECT per entity first - avoid it for bulk work
await this.products.insert(rows);                                         // bulk INSERT
await this.products.upsert(rows, { conflictPaths: ['sku'] });             // INSERT ... ON CONFLICT
await this.products.update({ status: 'DRAFT' }, { status: 'ARCHIVED' });  // single UPDATE
```

## TypeORM 1.x Migration Notes

| 0.3 API | 1.x replacement |
|---------|-----------------|
| `Connection`, `createConnection` | `DataSource`, `new DataSource().initialize()` |
| `getRepository()` / `getManager()` globals | Inject `Repository`/`DataSource` |
| `findByIds(ids)` | `findBy({ id: In(ids) })` |
| `findOneById(id)` | `findOneBy({ id })` |
| `exist()` | `exists()` |
| `@EntityRepository` | Provider wrapping `Repository<T>` |
| String `select`/`relations` arrays | Object syntax (`{ lines: true }`) |
| `sqlite3` driver | `better-sqlite3` |

Automated upgrade: `npx @typeorm/codemod v1 src/` (`--dry` to preview).

## Quick Reference

| Need | Use |
|------|-----|
| Inject repository | `@InjectRepository(Entity) repo: Repository<Entity>` |
| Load relations | `relations: { lines: true }` or `leftJoinAndSelect` |
| Atomic multi-step write | `dataSource.transaction(async (manager) => ...)` |
| Optimistic locking | `@VersionColumn` + conditional update (see `typeorm-patterns`) |
| Paging | `findAndCount({ skip, take, order })` or keyset with QueryBuilder |
| Bulk insert/upsert | `insert()`, `upsert()` |
| Schema changes | Migrations via the TypeORM CLI |
