---
name: code-quality
description: Comprehensive code review for TypeScript and NestJS - clean code principles, API contracts, type and null safety, async/error handling, and performance. Use when user says "review code", "refactor", "check API", or before merging changes.
---

# Code Quality Review Skill

Systematic code review combining clean code principles, API design, and TypeScript/NestJS best practices.

## When to Use
- "review this code" / "code review" / "check this PR"
- "refactor" / "clean this code" / "improve readability"
- "review API" / "check endpoints" / "REST review"
- Before merging PR or releasing API changes

## Review Strategy

1. **Quick scan** - Understand intent, identify scope
2. **Automated checks** - `npm run lint`, `npx tsc --noEmit`, `npm test`
3. **Checklist pass** - Apply relevant categories below
4. **Summary** - List findings by severity (Critical → Minor → Good)

---

## Clean Code Principles

### DRY - Don't Repeat Yourself

**Violation:**
```typescript
// ❌ Duplicated validation logic
createUser(req: UserRequest) {
  if (!req.email || !req.email.includes('@')) throw new BadRequestException('Invalid email');
}

updateUser(req: UserRequest) {
  if (!req.email || !req.email.includes('@')) throw new BadRequestException('Invalid email');
}
```

**Fix:**
```typescript
// ✅ Single source of truth - declarative DTO validation
export class UserRequest {
  @IsEmail()
  email!: string;
}
```

### KISS - Keep It Simple

**Violation:**
```typescript
// ❌ Over-engineered
interface UserFactory { createUser(): User }
class ConcreteUserFactory implements UserFactory {
  createUser(): User { return new User(); }
}
```

**Fix:**
```typescript
// ✅ Simple
const user = new User();
```

### YAGNI - You Aren't Gonna Need It

**Violation:**
```typescript
// ❌ Premature abstraction
export class ConfigurableUserServiceFactoryProvider {}
```

**Fix:**
```typescript
// ✅ Implement when actually needed
@Injectable()
export class UsersService {}
```

---

## API Contract Review

### HTTP Verb Semantics

| Verb | Use For | Idempotent | Safe |
|------|---------|------------|------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create new resource | No | No |
| PUT | Replace entire resource | Yes | No |
| PATCH | Partial update | No* | No |
| DELETE | Remove resource | Yes | No |

**Common Mistakes:**
```typescript
// ❌ POST for retrieval
@Post('users/search')
search(@Body() criteria: SearchCriteria) {}

// ✅ GET with a validated query DTO
@Get('users')
search(@Query() query: UserSearchQuery) {}

// ❌ GET for state change
@Get('users/:id/activate')
activate(@Param('id') id: string) {}

// ✅ POST for state change
@Post('users/:id/activate')
@HttpCode(HttpStatus.NO_CONTENT)
activate(@Param('id', ParseUUIDPipe) id: string) {}
```

### API Versioning

```typescript
// ✅ URI versioning (app.enableVersioning({ type: VersioningType.URI }))
@Controller({ path: 'users', version: '1' })
export class UsersController {}

// ❌ No versioning
@Controller('users') // Breaking changes affect all clients
```

### Response Status Codes

| Code | Use Case | Nest |
|------|----------|------|
| 200 OK | Successful GET/PUT/PATCH | default for non-POST |
| 201 Created | Successful POST | default for `@Post()` |
| 204 No Content | Successful DELETE | `@HttpCode(HttpStatus.NO_CONTENT)` |
| 400 Bad Request | Validation failure | `ValidationPipe`, `BadRequestException` |
| 404 Not Found | Resource doesn't exist | `NotFoundException` |
| 409 Conflict | State conflict | `ConflictException` |
| 500 Server Error | Unexpected error | unhandled errors |

### DTO vs Entity Exposure

```typescript
// ❌ Exposing TypeORM entity - leaks columns (passwordHash!), couples API to schema
@Get(':id')
get(@Param('id') id: string): Promise<User | null> {
  return this.users.findOneBy({ id });
}

// ✅ Map to a response DTO
@Get(':id')
async get(@Param('id', ParseUUIDPipe) id: string): Promise<UserResponse> {
  return UserResponse.from(await this.usersService.get(id));
}
```

---

## TypeScript Review Checklist

### Type Safety

```typescript
// ❌ any disables the compiler
function total(items: any) { return items.reduce((s: any, i: any) => s + i.price, 0); }

// ✅ Precise types
function total(items: readonly { priceCents: number }[]): number {
  return items.reduce((sum, item) => sum + item.priceCents, 0);
}

// ✅ unknown + narrowing for untrusted data
function parse(raw: unknown): Order {
  if (!isOrder(raw)) throw new BadRequestException('Invalid order');
  return raw;
}
```

**Flags:**
- `any`, `as any`, `@ts-ignore` (prefer `@ts-expect-error` with a reason)
- Type assertions (`as Foo`) on external data instead of validation
- Non-null assertions (`user!.name`) without a guarantee
- `strict` or `strictNullChecks` disabled

### Null Safety

```typescript
// ❌ Crash risk
const name = user.profile.name.toUpperCase();

// ✅ Optional chaining + nullish coalescing
const name = user.profile?.name?.toUpperCase() ?? '';

// ❌ || treats 0 and '' as missing
const limit = query.limit || 20;   // limit=0 becomes 20

// ✅ ?? only replaces null/undefined
const limit = query.limit ?? 20;
```

**Flags:**
- `findOneBy()` result used without a null check
- `||` used for defaults on numbers/strings
- Returning `null` where an empty array is expected

### Async Handling

```typescript
// ❌ Floating promise - errors become unhandled rejections, work may not finish
this.mailer.send(welcomeEmail);

// ✅ Await it, or make fire-and-forget explicit and handled
await this.mailer.send(welcomeEmail);
void this.mailer.send(welcomeEmail).catch((err) => this.logger.error('Welcome email failed', err));

// ❌ Sequential awaits for independent work
const user = await this.users.get(id);
const orders = await this.orders.forUser(id);

// ✅ Parallel
const [user, orders] = await Promise.all([this.users.get(id), this.orders.forUser(id)]);

// ❌ Unbounded parallelism (exhausts the DB pool / rate limits)
await Promise.all(ids.map((id) => this.sync(id)));

// ✅ Bounded batches
for (let i = 0; i < ids.length; i += 10) {
  await Promise.all(ids.slice(i, i + 10).map((id) => this.sync(id)));
}
```

**Flags:**
- Missing `await` (lint: `no-floating-promises`)
- `async` callbacks in `forEach` (never awaited)
- `return promise` inside `try` without `await` (catch is skipped)

### Error Handling

```typescript
// ❌ Swallowing errors
try { await process(); } catch {}

// ❌ Losing the cause
catch (err) { throw new Error('Processing failed'); }

// ✅ Log with context, keep the cause
catch (err) {
  this.logger.error(`Failed to process file ${filename}`, err instanceof Error ? err.stack : String(err));
  throw new ProcessingError('File processing failed', { cause: err });
}

// ❌ Throwing strings / plain objects
throw 'not found';

// ✅ Nest HTTP exceptions at the edge, domain errors inside
throw new NotFoundException(`Order ${id} not found`);
```

**Flags:**
- Empty `catch` blocks
- Errors re-thrown without `cause`
- Internal error messages / SQL returned to clients
- Generic `Error` thrown from controllers (becomes 500)

### Resource Management

```typescript
// ❌ Connection leak when the work throws
const qr = this.dataSource.createQueryRunner();
await qr.connect();
await work(qr);
await qr.release();

// ✅ try/finally
const qr = this.dataSource.createQueryRunner();
await qr.connect();
try {
  await work(qr);
} finally {
  await qr.release();
}

// ✅ Streams: pipeline() closes everything on error
await pipeline(createReadStream(src), createGzip(), createWriteStream(dest));
```

### Transaction Boundaries

```typescript
// ❌ Two independent writes
async createUser(dto: CreateUserDto) {
  const user = await this.users.save(this.users.create(dto));
  await this.roles.save(this.roles.create({ user, name: 'user' }));
}

// ✅ One atomic transaction
async createUser(dto: CreateUserDto) {
  return this.dataSource.transaction(async (manager) => {
    const user = await manager.save(manager.create(User, dto));
    await manager.save(manager.create(Role, { user, name: 'user' }));
    return user;
  });
}
```

### Dependency Injection

**Flags:**
- `new SomeService()` inside a provider instead of injection
- `moduleRef.get()` used as a service locator
- `forwardRef()` to break circular dependencies (extract a shared module instead)
- Request-scoped providers without a real need (they re-instantiate the whole chain per request)
- Reading `process.env` outside `src/config`

### Naming Conventions

**Good:**
```typescript
// ✅ Clear intent
findActiveUsersByRole(role: Role): Promise<User[]>
isEmailTaken(email: string): Promise<boolean>
activateUser(userId: string): Promise<void>
```

**Bad:**
```typescript
// ❌ Unclear
get(s: string)
check(str: string)
doStuff(id: string)
```

Files: `kebab-case` with role suffix (`orders.service.ts`, `create-order.dto.ts`); classes `PascalCase`; constants `UPPER_SNAKE_CASE`.

### Performance

```typescript
// ❌ N+1 query problem
const users = await this.users.find();
for (const user of users) {
  user.orders = await this.orders.findBy({ userId: user.id }); // N queries
}

// ✅ Load the relation
const users = await this.users.find({ relations: { orders: true }, take: 50 });

// ❌ Loading all data
const all = await this.users.find(); // could be millions

// ✅ Pagination
const [items, total] = await this.users.findAndCount({ skip: page * size, take: size, order: { id: 'ASC' } });

// ❌ Blocking the event loop in a request
const hash = crypto.pbkdf2Sync(password, salt, 600_000, 64, 'sha512');
const data = fs.readFileSync(path);

// ✅ Async APIs (or a worker thread / queue for CPU-heavy work)
const data = await readFile(path);
```

---

## Review Output Format

```markdown
## Code Review: [Component/Feature Name]

### Critical Issues
- **Null safety** (users.service.ts:42) - `findOneBy()` result dereferenced without a check. Throw `NotFoundException`.
- **Connection leak** (report.service.ts:15) - QueryRunner not released on error. Use try/finally.

### Important Improvements
- **API design** - POST used for idempotent update (users.controller.ts:28). Use PUT/PATCH.
- **Transaction missing** - Order and lines saved separately (orders.service.ts:56). Use `dataSource.transaction()`.
- **Floating promise** - `mailer.send()` not awaited (users.service.ts:71).
- **N+1 query** - Loop fetches orders individually (line 89). Use `relations`.

### Code Smells
- **Long method** - `buildReport()` is 80 lines. Extract steps.
- **Magic number** - Use a named constant instead of `86400` (line 123).
- **any** - `payload: any` in webhook handler (line 12). Validate with a DTO.

### Good Practices Observed
- ✅ Constructor injection throughout
- ✅ Response DTOs separate from entities
- ✅ Global ValidationPipe with whitelist
- ✅ Good test coverage (87%)
```

---

## Quick Reference Flags

| Category | Red Flags |
|----------|-----------|
| **Types** | `any`, `as` casts on input, `!` assertions, `@ts-ignore` |
| **Null Safety** | Unchecked `findOneBy()`, `||` defaults, deep property chains |
| **Async** | Floating promises, `forEach(async ...)`, unbounded `Promise.all` |
| **Errors** | Empty catch, lost `cause`, internals leaked to clients |
| **Resources** | QueryRunner without `finally`, unclosed streams |
| **API Design** | Wrong HTTP verb, no versioning, entity exposure |
| **Transactions** | Multi-step writes outside `dataSource.transaction()` |
| **DI** | `new` on providers, service locator, `forwardRef` cycles |
| **Performance** | N+1 queries, unbounded finds, sync I/O on requests |
| **Clean Code** | Duplication, magic numbers, unclear names |

---

## Severity Levels

- **Critical** - Security, data loss, crash risk → Must fix before merge
- **Important** - Performance, maintainability, correctness → Should fix
- **Code Smell** - Style, complexity, minor issues → Nice to have
- **Good** - Positive feedback to reinforce good practices
