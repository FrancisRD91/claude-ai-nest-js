# Security - Guards, JWT & Hardening

Install: `npm i @nestjs/jwt @nestjs/throttler helmet argon2`

## Security Configuration

```typescript
// src/app.module.ts (excerpt)
import { APP_GUARD } from '@nestjs/core';
import { JwtModule } from '@nestjs/jwt';
import { ThrottlerGuard, ThrottlerModule } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot([{ name: 'default', ttl: 60_000, limit: 100 }]),
    JwtModule.registerAsync({
      global: true,
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.getOrThrow<string>('JWT_SECRET'), // >= 32 random bytes, from env/secret manager
        signOptions: { expiresIn: '15m', issuer: 'orders-api', audience: 'orders-api', algorithm: 'HS256' },
        verifyOptions: { issuer: 'orders-api', audience: 'orders-api', algorithms: ['HS256'] },
      }),
    }),
  ],
  providers: [
    // Guards run in registration order: rate limit -> authenticate -> authorize
    { provide: APP_GUARD, useClass: ThrottlerGuard },
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RolesGuard },
  ],
})
export class AppModule {}

// src/app.setup.ts (excerpt)
app.use(helmet());
app.enableCors({ origin: allowedOrigins, credentials: true });
```

Secure by default: every route requires a valid token unless marked `@Public()`.

## Public Decorator

```typescript
// src/auth/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC, true);
```

## JWT Authentication Guard

```typescript
// src/auth/jwt-auth.guard.ts
import { type CanActivate, type ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { JwtService } from '@nestjs/jwt';
import type { Request } from 'express';
import { IS_PUBLIC } from './public.decorator.js';

export type Role = 'user' | 'admin';
export interface AuthUser { id: string; roles: Role[] }
interface JwtPayload { sub: string; roles: Role[] }

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly jwt: JwtService,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC, [context.getHandler(), context.getClass()]);
    if (isPublic) return true;

    const request = context.switchToHttp().getRequest<Request & { user?: AuthUser }>();
    const [scheme, token] = request.headers.authorization?.split(' ') ?? [];
    if (scheme !== 'Bearer' || !token) throw new UnauthorizedException();

    try {
      const payload = await this.jwt.verifyAsync<JwtPayload>(token); // uses verifyOptions from JwtModule
      request.user = { id: payload.sub, roles: payload.roles };
      return true;
    } catch {
      throw new UnauthorizedException();
    }
  }
}
```

## Role-Based Authorization

```typescript
// src/auth/roles.decorator.ts
import { Reflector } from '@nestjs/core';
import type { Role } from './jwt-auth.guard.js';

export const Roles = Reflector.createDecorator<Role[]>();

// src/auth/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride(Roles, [context.getHandler(), context.getClass()]);
    if (!required?.length) return true;
    const { user } = context.switchToHttp().getRequest<{ user?: AuthUser }>();
    return required.some((role) => user?.roles.includes(role) ?? false);
  }
}

// usage
@Roles(['admin'])
@Delete(':id')
remove(@Param('id', ParseUUIDPipe) id: string) { /* ... */ }
```

### Resource Ownership

Roles are not enough - check ownership in the service:

```typescript
async getForUser(id: string, user: AuthUser): Promise<Order> {
  const order = await this.orders.findOneBy({ id });
  // 404 (not 403) avoids revealing that the resource exists
  if (!order || (order.customerId !== user.id && !user.roles.includes('admin'))) {
    throw new NotFoundException(`Order ${id} not found`);
  }
  return order;
}
```

## Password Hashing

```typescript
// src/auth/password.service.ts
import { Injectable } from '@nestjs/common';
import argon2 from 'argon2';

@Injectable()
export class PasswordService {
  hash(password: string): Promise<string> {
    return argon2.hash(password); // argon2id by default
  }

  verify(hash: string, password: string): Promise<boolean> {
    return argon2.verify(hash, password);
  }
}
```

## Authentication Service

```typescript
// src/auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service.js';
import { PasswordService } from './password.service.js';

@Injectable()
export class AuthService {
  private readonly dummyHash: Promise<string>;

  constructor(
    private readonly users: UsersService,
    private readonly passwords: PasswordService,
    private readonly jwt: JwtService,
  ) {
    this.dummyHash = passwords.hash('timing-equalizer');
  }

  async login(email: string, password: string): Promise<{ accessToken: string }> {
    const user = await this.users.findByEmail(email);
    // always run a hash verification so response time does not reveal unknown emails
    const valid = await this.passwords.verify(user?.passwordHash ?? (await this.dummyHash), password);
    if (!user || !valid) throw new UnauthorizedException('Invalid credentials');
    return { accessToken: await this.jwt.signAsync({ sub: user.id, roles: user.roles }) };
  }
}
```

## Authentication Controller

```typescript
// src/auth/auth.controller.ts
import { Body, Controller, HttpCode, HttpStatus, Post } from '@nestjs/common';
import { Throttle } from '@nestjs/throttler';
import { IsEmail, IsString, MaxLength, MinLength } from 'class-validator';
import { AuthService } from './auth.service.js';
import { Public } from './public.decorator.js';

export class LoginDto {
  @IsEmail()
  email!: string;

  @IsString()
  @MinLength(12)
  @MaxLength(128)
  password!: string;
}

@Controller({ path: 'auth', version: '1' })
export class AuthController {
  constructor(private readonly auth: AuthService) {}

  @Public()
  @Throttle({ default: { limit: 5, ttl: 60_000 } }) // brute-force protection
  @Post('login')
  @HttpCode(HttpStatus.OK)
  login(@Body() dto: LoginDto): Promise<{ accessToken: string }> {
    return this.auth.login(dto.email, dto.password);
  }
}
```

**Refresh tokens:** issue long-lived opaque refresh tokens, store only their hash, rotate on every use, and revoke the whole family when a reused token is detected.

## OAuth2 / OIDC Resource Server

When an external identity provider issues tokens, verify them against its JWKS instead of a shared secret:

```typescript
// npm i jose
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { createRemoteJWKSet, type JWTPayload, jwtVerify } from 'jose';

@Injectable()
export class OidcTokenVerifier {
  private readonly jwks: ReturnType<typeof createRemoteJWKSet>;
  private readonly issuer: string;
  private readonly audience: string;

  constructor(config: ConfigService) {
    this.issuer = config.getOrThrow<string>('OIDC_ISSUER');
    this.audience = config.getOrThrow<string>('OIDC_AUDIENCE');
    this.jwks = createRemoteJWKSet(new URL(config.getOrThrow<string>('OIDC_JWKS_URI'))); // cached + rotated
  }

  async verify(token: string): Promise<JWTPayload> {
    const { payload } = await jwtVerify(token, this.jwks, {
      issuer: this.issuer,
      audience: this.audience,
      algorithms: ['RS256'],
    });
    return payload;
  }
}
```

Use it inside `JwtAuthGuard` in place of `JwtService.verifyAsync`, mapping provider claims (e.g. `realm_access.roles`) to `AuthUser.roles`.

## Testing Guards

```typescript
it('rejects requests without a token', async () => {
  await request(app.getHttpServer()).get('/api/v1/orders').expect(401);
});

it('allows admins to delete', async () => {
  const token = await jwt.signAsync({ sub: 'u1', roles: ['admin'] });
  await request(app.getHttpServer()).delete(`/api/v1/orders/${id}`).auth(token, { type: 'bearer' }).expect(204);
});
```

## Quick Reference

| Need | Use |
|------|-----|
| Authenticate all routes | `APP_GUARD` + `JwtAuthGuard` |
| Public endpoint | `@Public()` |
| Role check | `@Roles(['admin'])` + `RolesGuard` |
| Ownership check | Service-level check, 404 on mismatch |
| Rate limit | `ThrottlerModule` + `@Throttle()` per route |
| Security headers | `helmet()` |
| Password hashing | argon2id (`argon2` package) |
| External IdP tokens | `jose` + JWKS |

## Security Best Practices

- Validate and whitelist all input (`ValidationPipe` with `whitelist` + `forbidNonWhitelisted`)
- Pin JWT algorithms and check `iss`/`aud`; keep access tokens short-lived (5-15 min)
- Never log tokens, passwords, or full request bodies; redact `authorization` and `cookie` headers
- Use parameter binding in every query; never build SQL from strings
- Behind a proxy, set `app.set('trust proxy', 1)` (Express) so rate limiting sees real client IPs
- Use Redis storage for `@nestjs/throttler` when running more than one instance
- Return generic messages for auth failures ("Invalid credentials")
- Keep dependencies patched (`npm audit`, Renovate/Dependabot) and commit `package-lock.json`
- Load secrets from env or a secret manager; validate them at boot; never commit `.env`
