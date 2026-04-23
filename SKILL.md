---
name: nestjs-logging
description: >-
  NestJS logging and observability patterns. Use when adding logs, configuring
  log levels, enabling JSON logging, creating custom loggers, extending the
  built-in logger, integrating Pino, setting up request logging interceptors,
  or using Logger with dependency injection. Triggers: "add logging", "logger",
  "log levels", "JSON logs", "custom logger", "Pino", "observability".
---

# NestJS Logging

> Ref: https://docs.nestjs.com/techniques/logger

## Overview

NestJS provides a built-in `Logger` from `@nestjs/common`. You can disable it,
filter by level, format as JSON, extend it, or swap it for an external library like Pino.

---

## Basic Setup in main.ts

```typescript
import { NestFactory } from '@nestjs/core';
import { ConsoleLogger } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    // Option A: filter by level
    logger: ['error', 'warn', 'log'],

    // Option B: JSON logging (for AWS ECS, Datadog, etc.)
    // logger: new ConsoleLogger({ json: true }),

    // Option C: disable entirely
    // logger: false,
  });
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

### ConsoleLogger options

| Option | Type | Default | Description |
|---|---|---|---|
| `logLevels` | string[] | all | Active log levels |
| `json` | boolean | false | Structured JSON output |
| `colors` | boolean | true | Colorized output (auto-false when json=true) |
| `timestamp` | boolean | false | Time diff between log messages |
| `prefix` | string | `'Nest'` | Prefix on each message |
| `compact` | boolean | true | Single-line output |

---

## Per-Service Logger (Recommended Pattern)

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class OrderService {
  // Pass class name as context — appears in every log line
  private readonly logger = new Logger(OrderService.name);

  async create(dto: CreateOrderDto) {
    this.logger.log(`Creating order for customer: ${dto.customerId}`);
    try {
      const result = await this.repo.save(dto);
      this.logger.log(`Order created: ${result.id}`);
      return result;
    } catch (err) {
      // Always pass err.stack as second arg to logger.error
      this.logger.error(`Failed to create order`, err.stack);
      throw err;
    }
  }
}
```

### Log level methods

```typescript
this.logger.log('Normal flow');           // INFO
this.logger.warn('Recoverable issue');    // WARN
this.logger.error('Failed', err.stack);  // ERROR — always include stack
this.logger.debug('Dev detail');         // DEBUG
this.logger.verbose('Fine-grained');     // VERBOSE
this.logger.fatal('App is dying');       // FATAL
```

---

## JSON Logging (Production)

```typescript
const app = await NestFactory.create(AppModule, {
  logger: new ConsoleLogger({ json: true }),
});
```

Output per log line:
```json
{ "level": "log", "pid": 1234, "timestamp": 1607370779834, "message": "Starting...", "context": "NestFactory" }
```

---

## Extend the Built-in Logger

```typescript
import { ConsoleLogger, Injectable } from '@nestjs/common';

@Injectable()
export class AppLogger extends ConsoleLogger {
  log(message: string, context?: string) {
    // Add custom fields, forward to external sink, etc.
    super.log(message, context);
  }

  error(message: string, stack?: string, context?: string) {
    // e.g. send to Sentry here
    super.error(message, stack, context);
  }
}
```

Register it in `main.ts`:
```typescript
const app = await NestFactory.create(AppModule, { logger: new AppLogger() });
```

Or via DI in `AppModule`:
```typescript
{ provide: Logger, useClass: AppLogger }
```

---

## Request Logging Interceptor

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, Logger } from '@nestjs/common';
import { Observable, tap } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const { method, url } = context.switchToHttp().getRequest();
    const now = Date.now();
    return next.handle().pipe(
      tap(() => this.logger.log(`${method} ${url} — ${Date.now() - now}ms`)),
    );
  }
}
```

Register globally in `AppModule`:
```typescript
import { APP_INTERCEPTOR } from '@nestjs/core';
providers: [{ provide: APP_INTERCEPTOR, useClass: LoggingInterceptor }]
```

---

## External Logger: Pino

```bash
npm install pino pino-http nestjs-pino
```

```typescript
import { LoggerModule } from 'nestjs-pino';

@Module({
  imports: [
    LoggerModule.forRoot({
      pinoHttp: { level: process.env.NODE_ENV !== 'production' ? 'debug' : 'info' },
    }),
  ],
})
export class AppModule {}
```

```typescript
// main.ts
import { Logger } from 'nestjs-pino';
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(Logger));
```

---

## Checklist

- [ ] Every `@Injectable()` and `@Controller()` has `private readonly logger = new Logger(ClassName.name)`
- [ ] `logger.error()` always called with `err.stack` as second argument
- [ ] No `console.log` in `src/`
- [ ] JSON logging enabled in production (`ConsoleLogger({ json: true })`)
- [ ] `LoggingInterceptor` registered globally for HTTP request tracing
- [ ] Sensitive data (passwords, tokens, PII) never logged
