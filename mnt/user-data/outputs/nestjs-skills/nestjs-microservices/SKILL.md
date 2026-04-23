---
name: nestjs-microservices
description: >-
  NestJS microservice transport patterns. Use when setting up a microservice,
  using message or event patterns, sending messages with ClientProxy, publishing
  events between services, handling timeouts, configuring transporters (TCP,
  Redis, RabbitMQ, Kafka, NATS, gRPC), or building hybrid HTTP + microservice
  apps. Triggers: "microservice", "message pattern", "event pattern",
  "ClientProxy", "send message", "publish event", "transporter", "hybrid app".
---

# NestJS Microservices

> Ref: https://docs.nestjs.com/microservices/basics

## Overview

NestJS microservices communicate via **message patterns** (request/response) or
**event patterns** (fire-and-forget). Transporters include TCP (default), Redis,
RabbitMQ, Kafka, NATS, and gRPC.

---

## Installation

```bash
npm install @nestjs/microservices
# Add transporter-specific package as needed:
# npm install ioredis          # Redis
# npm install amqplib          # RabbitMQ
# npm install kafkajs          # Kafka
# npm install nats             # NATS
# npm install @grpc/grpc-js @grpc/proto-loader  # gRPC
```

---

## Standalone Microservice (main.ts)

```typescript
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
    transport: Transport.TCP,
    options: { host: '0.0.0.0', port: 3001 },
  });
  await app.listen();
}
bootstrap();
```

---

## Handling Messages and Events

```typescript
import { Controller } from '@nestjs/common';
import { MessagePattern, EventPattern, Payload, Ctx, TcpContext } from '@nestjs/microservices';

@Controller()
export class OrdersController {
  // Request/response — caller awaits a reply
  @MessagePattern('get_order')
  async getOrder(@Payload() id: string, @Ctx() ctx: TcpContext): Promise<Order> {
    return this.ordersService.findOne(id);
  }

  // Event — fire-and-forget, no response expected
  @EventPattern('order_created')
  async handleOrderCreated(@Payload() data: OrderCreatedEvent): Promise<void> {
    await this.ordersService.processNew(data);
  }
}
```

---

## Client (Producer) — Sending from Another Service

### Module setup

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'ORDER_SERVICE',           // injection token
        transport: Transport.TCP,
        options: { host: 'localhost', port: 3001 },
      },
    ]),
  ],
  providers: [AppService],
})
export class AppModule {}
```

### Injecting and using ClientProxy

```typescript
import { Inject, Injectable } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { firstValueFrom, timeout } from 'rxjs';

@Injectable()
export class AppService {
  constructor(@Inject('ORDER_SERVICE') private readonly client: ClientProxy) {}

  // Request/response — returns Observable, convert to Promise
  async getOrder(id: string): Promise<Order> {
    return firstValueFrom(
      this.client.send<Order>('get_order', id).pipe(timeout(5000)),
    );
  }

  // Event — fire and forget
  notifyOrderCreated(order: Order): void {
    this.client.emit('order_created', order);
  }
}
```

---

## Handling Timeouts

Always pipe a `timeout` operator — unresponsive services will hang indefinitely otherwise.

```typescript
import { timeout, catchError } from 'rxjs/operators';
import { throwError, TimeoutError } from 'rxjs';

this.client.send('get_order', id).pipe(
  timeout(5000),
  catchError(err => {
    if (err instanceof TimeoutError) {
      return throwError(() => new RequestTimeoutException());
    }
    return throwError(() => err);
  }),
);
```

---

## Hybrid App (HTTP + Microservice)

Run both an HTTP server and a microservice listener in one app:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);  // HTTP

  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: { port: 3001 },
  });

  await app.startAllMicroservices();  // start microservice listener
  await app.listen(3000);            // start HTTP server
}
```

---

## Common Transporters Quick Reference

```typescript
// Redis
{ transport: Transport.REDIS, options: { host: 'localhost', port: 6379 } }

// RabbitMQ
{ transport: Transport.RMQ, options: { urls: ['amqp://localhost'], queue: 'orders_queue' } }

// Kafka
{ transport: Transport.KAFKA, options: { client: { brokers: ['localhost:9092'] }, consumer: { groupId: 'orders-consumer' } } }

// NATS
{ transport: Transport.NATS, options: { servers: ['nats://localhost:4222'] } }
```

---

## Checklist

- [ ] All `send()` calls have a `timeout()` operator applied
- [ ] `firstValueFrom()` used to convert Observable to Promise (not `.toPromise()` which is deprecated)
- [ ] `ClientsModule.register()` uses a string injection token (e.g. `'ORDER_SERVICE'`)
- [ ] `@EventPattern` handlers are void — they don't return a reply
- [ ] `@MessagePattern` handlers return a value — the client receives it
- [ ] Hybrid app calls `startAllMicroservices()` before `listen()`
- [ ] Transporter package installed for non-TCP transport (ioredis, amqplib, etc.)
