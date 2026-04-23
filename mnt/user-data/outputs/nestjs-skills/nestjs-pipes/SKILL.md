---
name: nestjs-pipes
description: >-
  NestJS pipe patterns for validation and transformation. Use when adding
  ValidationPipe, parsing route params, transforming request data, writing
  custom pipes, using class-validator decorators on DTOs, applying pipes
  globally or per-route, or using built-in pipes like ParseIntPipe,
  ParseUUIDPipe, DefaultValuePipe. Triggers: "validate", "pipe", "DTO",
  "ValidationPipe", "ParseIntPipe", "class-validator", "transform input",
  "whitelist", "forbidNonWhitelisted", "custom pipe".
---

# NestJS Pipes

> Ref: https://docs.nestjs.com/pipes

## Overview

Pipes run **before** the route handler and have two jobs:
- **Transformation** — coerce input to the expected type (string → int, etc.)
- **Validation** — throw `BadRequestException` if input is invalid

Pipes run inside the exceptions zone — any exception they throw is handled by the active exception filter.

---

## Built-in Pipes

| Pipe | Purpose |
|---|---|
| `ValidationPipe` | Validates DTOs using `class-validator` decorators |
| `ParseIntPipe` | Converts string → integer; throws 400 if not parseable |
| `ParseFloatPipe` | Converts string → float |
| `ParseBoolPipe` | Converts `'true'`/`'false'` → boolean |
| `ParseArrayPipe` | Parses comma-separated strings into arrays |
| `ParseUUIDPipe` | Validates UUID format |
| `ParseEnumPipe` | Validates value is a valid enum member |
| `DefaultValuePipe` | Supplies a default when value is `null`/`undefined` |

---

## Global ValidationPipe (Recommended)

Register once in `main.ts` — applies to all routes:

```typescript
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,            // strip unknown properties
      forbidNonWhitelisted: true, // throw 400 if unknown props sent
      transform: true,            // auto-transform payloads to DTO class instances
      transformOptions: {
        enableImplicitConversion: true, // convert query string types automatically
      },
    }),
  );
  await app.listen(3000);
}
```

Or with DI support via `APP_PIPE` in `AppModule`:
```typescript
import { APP_PIPE } from '@nestjs/core';
providers: [{ provide: APP_PIPE, useValue: new ValidationPipe({ whitelist: true, transform: true }) }]
```

---

## DTO with class-validator

```typescript
import { IsString, IsInt, IsOptional, Min, Max, IsEmail, IsUUID } from 'class-validator';
import { Type } from 'class-transformer';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  @Min(0)
  @Max(30)
  @Type(() => Number)   // required when transform: true is not set globally
  age: number;

  @IsString()
  breed: string;

  @IsEmail()
  @IsOptional()
  ownerEmail?: string;
}

export class UpdateCatDto extends PartialType(CreateCatDto) {}  // all fields optional
```

---

## Binding Built-in Pipes on Route Params

```typescript
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // id is guaranteed to be a number; 400 thrown automatically if not
  return this.catsService.findOne(id);
}

@Get(':uuid')
findByUuid(@Param('uuid', ParseUUIDPipe) uuid: string) {
  return this.catsService.findByUuid(uuid);
}

@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('limit', new DefaultValuePipe(20), ParseIntPipe) limit: number,
) {
  return this.catsService.findAll({ page, limit });
}
```

---

## Custom Pipe

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class PositiveIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val) || val <= 0) {
      throw new BadRequestException(`${metadata.data} must be a positive integer`);
    }
    return val;
  }
}

// Usage
@Get(':id')
findOne(@Param('id', PositiveIntPipe) id: number) { ... }
```

---

## Schema-Based Validation with Zod

```typescript
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';
import { ZodSchema } from 'zod';

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    const result = this.schema.safeParse(value);
    if (!result.success) {
      throw new BadRequestException(result.error.format());
    }
    return result.data;
  }
}

// Usage
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
create(@Body() dto: CreateCatDto) { ... }
```

---

## Checklist

- [ ] `ValidationPipe` registered globally with `whitelist: true` and `transform: true`
- [ ] All `@Body()` parameters use typed DTO classes with `class-validator` decorators
- [ ] `UpdateDto` uses `PartialType(CreateDto)` — no duplicate field definitions
- [ ] `ParseIntPipe` / `ParseUUIDPipe` used on `:id` route params — no manual `parseInt`
- [ ] `@Type(() => Number)` added to numeric DTO fields when relying on implicit conversion from query strings
- [ ] Custom pipes implement `PipeTransform` and throw `BadRequestException` on invalid input
