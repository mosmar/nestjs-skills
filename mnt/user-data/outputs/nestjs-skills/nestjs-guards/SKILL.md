---
name: nestjs-guards
description: >-
  NestJS guard patterns for authorization and access control. Use when
  implementing authentication guards, role-based access control (RBAC),
  reading route metadata with Reflector, using ExecutionContext to access
  request data, applying guards globally or per route, or creating custom
  decorators for roles/permissions. Triggers: "guard", "auth guard",
  "authorization", "roles", "permissions", "CanActivate", "Reflector",
  "protect route", "access control", "JWT guard", "API key".
---

# NestJS Guards

> Ref: https://docs.nestjs.com/guards

## Overview

Guards implement `CanActivate` and return `true` (allow) or `false` (deny → 403).
They execute **after middleware** but **before interceptors and pipes**.

Guards are designed for **authorization** — deciding if an authenticated user can
access a given route. Use middleware for authentication (token extraction/validation).

---

## Basic Auth Guard

```typescript
// common/guards/auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return this.validateRequest(request);
  }

  private validateRequest(request: Request): boolean {
    // Extract and validate token from request.headers.authorization
    // Return true to allow, false to deny (→ 403 ForbiddenException)
    return !!request.headers['authorization'];
  }
}
```

---

## Role-Based Access Control (RBAC)

### Step 1: Custom `@Roles()` decorator

```typescript
// common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

### Step 2: RolesGuard reads metadata via Reflector

```typescript
// common/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // Read roles from handler, fall back to controller metadata
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) return true; // no roles required — allow all

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user?.roles?.includes(role));
  }
}
```

### Step 3: Apply on routes

```typescript
@Controller('orders')
@UseGuards(AuthGuard, RolesGuard)   // AuthGuard first, then RolesGuard
export class OrdersController {
  @Post()
  @Roles('admin', 'manager')
  create(@Body() dto: CreateOrderDto) { ... }

  @Get()
  @Roles('admin', 'manager', 'viewer')
  findAll() { ... }
}
```

---

## Binding Guards

```typescript
// Method-level
@UseGuards(AuthGuard)
@Get('profile')
getProfile() { ... }

// Controller-level
@UseGuards(AuthGuard)
@Controller('cats')
export class CatsController { ... }

// Global — supports DI (preferred)
import { APP_GUARD } from '@nestjs/core';
providers: [{ provide: APP_GUARD, useClass: AuthGuard }]

// Global — no DI support
app.useGlobalGuards(new AuthGuard());
```

> Prefer `APP_GUARD` in `AppModule` over `useGlobalGuards` in `main.ts` to support dependency injection.

---

## ExecutionContext Helpers

```typescript
// HTTP
const request = context.switchToHttp().getRequest<Request>();
const response = context.switchToHttp().getResponse<Response>();

// Microservice (RPC)
const data = context.switchToRpc().getData();

// WebSocket
const client = context.switchToWs().getClient();

// Get handler/controller class (for metadata)
const handler = context.getHandler();   // the route method
const controller = context.getClass();  // the controller class
```

---

## Public Routes (Skip Global Guard)

```typescript
// common/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

```typescript
// In your global AuthGuard
canActivate(context: ExecutionContext): boolean {
  const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
    context.getHandler(),
    context.getClass(),
  ]);
  if (isPublic) return true;
  // ... normal auth logic
}
```

```typescript
// Usage on open endpoints
@Public()
@Get('health')
health() { return 'ok'; }
```

---

## Checklist

- [ ] Guards implement `CanActivate` and are decorated with `@Injectable()`
- [ ] `RolesGuard` uses `reflector.getAllAndOverride()` — checks handler first, then controller
- [ ] `APP_GUARD` used for global registration (not `useGlobalGuards`) so DI works
- [ ] Multiple guards applied in order: `@UseGuards(AuthGuard, RolesGuard)` — left to right
- [ ] `@Public()` decorator implemented if using a global auth guard, to opt-out specific routes
- [ ] Guard returns `false` → NestJS throws `ForbiddenException` (403), not 401
- [ ] Authentication (token extraction) done in middleware; authorization (can access?) done in guards
