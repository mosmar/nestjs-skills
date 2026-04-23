---
name: nestjs-interceptors
description: >-
  NestJS interceptor patterns. Use when adding cross-cutting concerns like
  request/response logging, response transformation, caching, timeout handling,
  mapping exceptions, or any "wrap around a route handler" behaviour. Triggers:
  "interceptor", "NestInterceptor", "logging interceptor", "transform response",
  "response mapping", "timeout", "cache", "before/after handler", "tap", "map",
  "CallHandler", "ExecutionContext".
---

# NestJS Interceptors

> Ref: https://docs.nestjs.com/interceptors

## Overview

Interceptors implement `NestInterceptor` and wrap route handler execution.
They can run logic **before and after** the handler, transform responses,
remap exceptions, or short-circuit the request entirely (e.g. cache hit).

Inspired by AOP — use for cross-cutting concerns that shouldn't live in controllers.

Execution order: Middleware → Guards → **Interceptors** → Pipes → Handler → **Interceptors** (response)

---

## Structure

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class MyInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    // --- BEFORE handler ---
    return next.handle().pipe(
      // --- AFTER handler (operates on the response stream) ---
    );
  }
}
```

`next.handle()` returns an `Observable` of the handler's response. Use RxJS operators to transform it.

---

## Logging Interceptor (Timing)

```typescript
// common/interceptors/logging.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, Logger } from '@nestjs/common';
import { Observable, tap } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const req = context.switchToHttp().getRequest();
    const { method, url } = req;
    const now = Date.now();

    return next.handle().pipe(
      tap(() => this.logger.log(`${method} ${url} — ${Date.now() - now}ms`)),
    );
  }
}
```

---

## Response Transform Interceptor

Wraps every response in a `{ data: ... }` envelope:

```typescript
// common/interceptors/transform.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, map } from 'rxjs';

export interface Response<T> { data: T; }

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

---

## Timeout Interceptor

```typescript
// common/interceptors/timeout.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { timeout, catchError } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  }
}
```

---

## Cache Interceptor (Short-circuit Pattern)

```typescript
// common/interceptors/cache.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  private readonly cache = new Map<string, unknown>();

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const key = context.switchToHttp().getRequest().url;
    if (this.cache.has(key)) {
      return of(this.cache.get(key)); // short-circuit — never calls handler
    }
    return next.handle(); // call handler and cache result externally as needed
  }
}
```

---

## Exception Mapping Interceptor

Re-map exceptions thrown by the handler:

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, BadGatewayException } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      catchError(err => throwError(() => new BadGatewayException(err.message))),
    );
  }
}
```

---

## Binding Interceptors

```typescript
// Method-level
@UseInterceptors(LoggingInterceptor)
@Get()
findAll() { ... }

// Controller-level
@UseInterceptors(LoggingInterceptor)
@Controller('cats')
export class CatsController { ... }

// Global — supports DI (preferred)
import { APP_INTERCEPTOR } from '@nestjs/core';
providers: [{ provide: APP_INTERCEPTOR, useClass: LoggingInterceptor }]

// Global — no DI support
app.useGlobalInterceptors(new LoggingInterceptor());
```

> Prefer `APP_INTERCEPTOR` in `AppModule` over `useGlobalInterceptors` in `main.ts` to support DI.

---

## Checklist

- [ ] Interceptor class is `@Injectable()` and implements `NestInterceptor`
- [ ] `next.handle()` always called and its `Observable` returned — never omitted
- [ ] RxJS `tap` used for side effects (logging); `map` for response transformation
- [ ] `timeout` + `catchError` used in `TimeoutInterceptor` to emit `RequestTimeoutException`
- [ ] `APP_INTERCEPTOR` used for global registration (not `useGlobalInterceptors`) to enable DI
- [ ] Response envelope interceptor (`TransformInterceptor`) applied consistently — don't mix enveloped and raw responses
- [ ] Interceptors do not contain business logic — only cross-cutting concerns
