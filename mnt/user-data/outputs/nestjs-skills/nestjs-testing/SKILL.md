---
name: nestjs-testing
description: >-
  NestJS unit and end-to-end testing patterns. Use when writing tests, setting
  up Test.createTestingModule, mocking providers or repositories, testing
  controllers or services in isolation, configuring e2e tests with supertest,
  overriding global enhancers in tests, or using auto-mocking. Triggers:
  "write a test", "unit test", "e2e test", "mock service", "testing module",
  "jest", "supertest", "spec file".
---

# NestJS Testing

> Ref: https://docs.nestjs.com/fundamentals/testing

## Overview

NestJS integrates with Jest out of the box. Two test types:
- **Unit tests** — test a single class in isolation; mock all dependencies
- **End-to-end (e2e) tests** — spin up the full app; test real HTTP behaviour via `supertest`

---

## Installation

```bash
npm install --save-dev @nestjs/testing jest @types/jest ts-jest supertest @types/supertest
```

---

## Unit Testing a Service

```typescript
// cats/cats.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { CatsService } from './cats.service';
import { Cat } from './cat.entity';
import { NotFoundException } from '@nestjs/common';

// Typed partial mock helper — use for any TypeORM Repository
type MockRepo<T> = Partial<Record<keyof Repository<T>, jest.Mock>>;
const createMockRepo = <T>(): MockRepo<T> => ({
  find: jest.fn(),
  findOne: jest.fn(),
  create: jest.fn(),
  save: jest.fn(),
  softDelete: jest.fn(),
});

describe('CatsService', () => {
  let service: CatsService;
  let repo: MockRepo<Cat>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CatsService,
        { provide: getRepositoryToken(Cat), useValue: createMockRepo<Cat>() },
      ],
    }).compile();

    service = module.get<CatsService>(CatsService);
    repo = module.get(getRepositoryToken(Cat));
  });

  afterEach(() => jest.clearAllMocks());

  describe('findOne', () => {
    it('returns the cat when found', async () => {
      const cat = { id: 1, name: 'Whiskers' } as Cat;
      repo.findOne!.mockResolvedValue(cat);
      expect(await service.findOne(1)).toEqual(cat);
    });

    it('throws NotFoundException when not found', async () => {
      repo.findOne!.mockResolvedValue(null);
      await expect(service.findOne(99)).rejects.toThrow(NotFoundException);
    });
  });
});
```

---

## Unit Testing a Controller

Controllers should only verify routing and that the correct service method is called — no business logic assertions.

```typescript
// cats/cats.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let controller: CatsController;
  let service: jest.Mocked<CatsService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [CatsController],
      providers: [
        {
          provide: CatsService,
          useValue: { findAll: jest.fn(), findOne: jest.fn(), create: jest.fn() },
        },
      ],
    }).compile();

    controller = module.get<CatsController>(CatsController);
    service = module.get(CatsService);
  });

  afterEach(() => jest.clearAllMocks());

  it('calls service.findAll and returns result', async () => {
    service.findAll.mockResolvedValue([{ id: 1, name: 'Cat' }] as any);
    const result = await controller.findAll();
    expect(service.findAll).toHaveBeenCalled();
    expect(result).toHaveLength(1);
  });
});
```

---

## Auto Mocking

`createMock` from `@golevelup/ts-jest` auto-generates mocks for any class:

```bash
npm install --save-dev @golevelup/ts-jest
```

```typescript
import { createMock } from '@golevelup/ts-jest';
import { CatsService } from './cats.service';

const service = createMock<CatsService>();
service.findAll.mockResolvedValue([]);
```

---

## End-to-End Testing

```typescript
// test/cats.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Cats (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [AppModule], // use a test DB via .env.test
    }).compile();

    app = module.createNestApplication();
    // Mirror your main.ts setup exactly
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }));
    await app.init();
  });

  afterAll(() => app.close());

  it('GET /cats returns 200', () =>
    request(app.getHttpServer()).get('/cats').expect(200));

  it('POST /cats returns 201 with valid body', () =>
    request(app.getHttpServer())
      .post('/cats')
      .send({ name: 'Whiskers', age: 3, breed: 'Maine Coon' })
      .expect(201));

  it('POST /cats returns 400 with invalid body', () =>
    request(app.getHttpServer())
      .post('/cats')
      .send({ name: 123 }) // wrong type
      .expect(400));
});
```

---

## Overriding Global Enhancers in Tests

```typescript
// Override a global guard for specific tests
const module = await Test.createTestingModule({ imports: [AppModule] })
  .overrideGuard(AuthGuard)
  .useValue({ canActivate: () => true }) // bypass auth
  .overridePipe(ValidationPipe)
  .useValue(new ValidationPipe({ whitelist: true }))
  .compile();
```

---

## jest.config.js

```javascript
// jest.config.js (unit tests)
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  collectCoverageFrom: ['**/*.(t|j)s'],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
};
```

```json
// test/jest-e2e.json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" }
}
```

---

## Checklist

- [ ] Every `*.service.ts` has `*.service.spec.ts`
- [ ] Every `*.controller.ts` has `*.controller.spec.ts`
- [ ] Repositories mocked via `getRepositoryToken(Entity)` — never instantiated directly
- [ ] `jest.clearAllMocks()` in `afterEach`
- [ ] e2e tests use a dedicated test database — never dev or prod
- [ ] `ValidationPipe` applied in e2e setup to match production
- [ ] Global guards/pipes overridden with `.overrideGuard()` / `.overridePipe()` as needed
- [ ] `app.close()` called in `afterAll` to avoid open handles
