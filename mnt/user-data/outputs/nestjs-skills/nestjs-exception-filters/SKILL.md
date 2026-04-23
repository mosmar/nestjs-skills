---
name: nestjs-exception-filters
description: >-
  NestJS exception handling patterns. Use when throwing HTTP exceptions,
  creating custom exception filters, catching all exceptions globally,
  handling RPC or WS exceptions in microservices, using built-in HTTP
  exceptions, logging exceptions, or binding filters at controller, method,
  or global scope. Triggers: "exception", "error handling", "HttpException",
  "throw", "custom filter", "exception filter", "catch error", "400", "404",
  "500", "NotFoundException", "BadRequestException".
---

# NestJS Exception Filters

> Ref: https://docs.nestjs.com/exception-filters

## Overview

NestJS has a built-in **exceptions layer** that catches all unhandled exceptions
and returns a JSON error response. You can extend this with custom filters.

Default response for unrecognised exceptions:
```json
{ "statusCode": 500, "message": "Internal server error" }
```

---

## Throwing Built-in HTTP Exceptions

```typescript
import {
  BadRequestException, UnauthorizedException, ForbiddenException,
  NotFoundException, ConflictException, UnprocessableEntityException,
  InternalServerErrorException, HttpException, HttpStatus,
} from '@nestjs/common';

// Simple — message only
throw new NotFoundException('Cat not found');

// With custom response body
throw new HttpException({ status: HttpStatus.FORBIDDEN, error: 'Custom message' }, HttpStatus.FORBIDDEN);

// With error cause (for logging — not serialized to response)
throw new HttpException('Forbidden', HttpStatus.FORBIDDEN, { cause: originalError });
```

### All Built-in HTTP Exceptions

| Class | Status |
|---|---|
| `BadRequestException` | 400 |
| `UnauthorizedException` | 401 |
| `ForbiddenException` | 403 |
| `NotFoundException` | 404 |
| `MethodNotAllowedException` | 405 |
| `NotAcceptableException` | 406 |
| `RequestTimeoutException` | 408 |
| `ConflictException` | 409 |
| `GoneException` | 410 |
| `UnprocessableEntityException` | 422 |
| `TooManyRequestsException` | 429 |
| `InternalServerErrorException` | 500 |
| `NotImplementedException` | 501 |
| `ServiceUnavailableException` | 503 |

---

## Custom Exception Class

```typescript
// common/exceptions/business.exception.ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class BusinessException extends HttpException {
  constructor(message: string, code: string) {
    super({ statusCode: HttpStatus.UNPROCESSABLE_ENTITY, message, code }, HttpStatus.UNPROCESSABLE_ENTITY);
  }
}

// Usage
throw new BusinessException('Insufficient stock', 'STOCK_INSUFFICIENT');
```

---

## Custom Exception Filter

```typescript
// common/filters/http-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, Logger } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    const body = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: typeof exceptionResponse === 'string'
        ? exceptionResponse
        : (exceptionResponse as any).message,
    };

    this.logger.error(`${request.method} ${request.url} → ${status}`, exception.stack);
    response.status(status).json(body);
  }
}
```

---

## Catch-All Filter (Every Exception)

```typescript
// common/filters/all-exceptions.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, HttpStatus, Logger } from '@nestjs/common';

@Catch() // no argument = catches everything
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    const message = exception instanceof HttpException
      ? exception.message
      : 'Internal server error';

    this.logger.error(`Unhandled exception on ${request.url}`, exception instanceof Error ? exception.stack : String(exception));

    response.status(status).json({ statusCode: status, message, timestamp: new Date().toISOString(), path: request.url });
  }
}
```

---

## Binding Filters

```typescript
// Method-level
@Post()
@UseFilters(HttpExceptionFilter)
create(@Body() dto: CreateCatDto) { ... }

// Controller-level
@UseFilters(HttpExceptionFilter)
@Controller('cats')
export class CatsController { ... }

// Global — in AppModule (supports DI)
import { APP_FILTER } from '@nestjs/core';
providers: [{ provide: APP_FILTER, useClass: AllExceptionsFilter }]

// Global — in main.ts (no DI support)
app.useGlobalFilters(new AllExceptionsFilter());
```

> Prefer `APP_FILTER` in `AppModule` over `useGlobalFilters` in `main.ts` — it supports dependency injection.

---

## Microservice Exception Filter

```typescript
import { Catch, RpcExceptionFilter, ArgumentsHost } from '@nestjs/common';
import { RpcException } from '@nestjs/microservices';
import { throwError } from 'rxjs';

@Catch(RpcException)
export class RpcExceptionFilter implements RpcExceptionFilter<RpcException> {
  catch(exception: RpcException, host: ArgumentsHost) {
    return throwError(() => exception.getError());
  }
}
```

---

## Checklist

- [ ] All `throw` statements use built-in NestJS exception classes, not raw `Error`
- [ ] Custom filter created for logging exceptions with request context (method, URL, timestamp)
- [ ] Global `AllExceptionsFilter` registered to catch unexpected non-HTTP errors
- [ ] `APP_FILTER` used for global registration (not `useGlobalFilters`) to enable DI
- [ ] `{ cause: err }` passed to `HttpException` constructor for inner error logging
- [ ] Microservices use `RpcException` — not `HttpException`
