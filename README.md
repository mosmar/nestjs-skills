# NestJS Copilot Skills
> Generated from https://docs.nestjs.com/

Personal GitHub Copilot skills covering NestJS core building blocks.
Install to `~/.copilot/skills/` to make them available across all repos.

## Installation

```bash
mkdir -p ~/.copilot/skills
for skill in nestjs-logging nestjs-testing nestjs-microservices nestjs-exception-filters nestjs-pipes nestjs-guards nestjs-interceptors; do
  cp -r $skill ~/.copilot/skills/
done
```

Add to VS Code `settings.json`:
```json
{
  "chat.agentSkillsLocations": {
    "~/.copilot/skills/**": true
  }
}
```

Type `/skills` in Copilot Chat to confirm they loaded.

---

## Skills

| Skill | Doc | Auto-triggers on... |
|---|---|---|
| `nestjs-logging` | [techniques/logger](https://docs.nestjs.com/techniques/logger) | Logger, log levels, JSON logs, Pino, request logging |
| `nestjs-testing` | [fundamentals/testing](https://docs.nestjs.com/fundamentals/testing) | Unit tests, mocking, e2e, jest, supertest, spec files |
| `nestjs-microservices` | [microservices/basics](https://docs.nestjs.com/microservices/basics) | Message patterns, ClientProxy, transporters, hybrid apps |
| `nestjs-exception-filters` | [exception-filters](https://docs.nestjs.com/exception-filters) | HTTP exceptions, custom filters, throw, 404, 400, RpcException |
| `nestjs-pipes` | [pipes](https://docs.nestjs.com/pipes) | ValidationPipe, DTOs, ParseIntPipe, class-validator, transform |
| `nestjs-guards` | [guards](https://docs.nestjs.com/guards) | Auth guard, RBAC, roles, CanActivate, Reflector, permissions |
| `nestjs-interceptors` | [interceptors](https://docs.nestjs.com/interceptors) | Logging, response transform, timeout, cache, exception mapping |

## Usage

Copilot loads skills automatically when your prompt matches the description.
You can also invoke any skill explicitly:

```
/nestjs-logging
/nestjs-testing
/nestjs-microservices
/nestjs-exception-filters
/nestjs-pipes
/nestjs-guards
/nestjs-interceptors
```

## Request Lifecycle (Skills in Context)

```
Incoming Request
      ↓
  Middleware         (token extraction, correlation ID)
      ↓
  Guards             ← nestjs-guards
      ↓
  Interceptors PRE   ← nestjs-interceptors
      ↓
  Pipes              ← nestjs-pipes
      ↓
  Route Handler
      ↓
  Interceptors POST  ← nestjs-interceptors
      ↓
  Exception Filters  ← nestjs-exception-filters (if error thrown)
      ↓
  Response
```
# nestjs-skills
