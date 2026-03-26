# Error Filter Patterns — Full Template

`HttpExceptionFilter` maps domain errors to HTTP status codes. The domain layer never imports NestJS — the filter is the bridge.

## File Location

```
shared/infrastructure/filters/http-exception.filter.ts
```

Or per-module at `infrastructure/filters/http-exception.filter.ts`.

## Full Implementation

```typescript
// shared/infrastructure/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import type { FastifyReply, FastifyRequest } from 'fastify';

// Domain errors — import from shared layer
import { NotFoundError } from '@/shared/domain/errors/not-found-error';
import { BusinessRuleViolationError } from '@/shared/domain/errors/business-rule-violation.error';
import { ConflictError } from '@/shared/domain/errors/conflict.error';
import { UnauthorizedError } from '@/shared/domain/errors/unauthorized.error';
import { ForbiddenError } from '@/shared/domain/errors/forbidden.error';
import { ValidationError } from '@/shared/domain/errors/validation.error';

// Map domain error classes to HTTP status codes
// Order matters: more specific subclasses first
const DOMAIN_ERROR_MAP: Array<[new (...args: never[]) => Error, HttpStatus]> = [
  [UnauthorizedError,          HttpStatus.UNAUTHORIZED],
  [ForbiddenError,             HttpStatus.FORBIDDEN],
  [NotFoundError,              HttpStatus.NOT_FOUND],
  [ConflictError,              HttpStatus.CONFLICT],
  [ValidationError,            HttpStatus.BAD_REQUEST],
  [BusinessRuleViolationError, HttpStatus.UNPROCESSABLE_ENTITY],
];

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const reply = ctx.getResponse<FastifyReply>();
    const request = ctx.getRequest<FastifyRequest>();

    const { status, message } = this.resolve(exception);

    if (status === HttpStatus.INTERNAL_SERVER_ERROR) {
      this.logger.error(
        `Unhandled error on ${request.method} ${request.url}`,
        exception,
      );
    } else {
      this.logger.debug(`Domain error [${status}]: ${message}`);
    }

    void reply.status(status).send({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }

  private resolve(exception: unknown): { status: HttpStatus; message: string } {
    // NestJS built-in HttpException (ValidationPipe, guards, etc.)
    if (exception instanceof HttpException) {
      const response = exception.getResponse();
      const message =
        typeof response === 'string'
          ? response
          : (response as Record<string, unknown>).message;

      return {
        status: exception.getStatus(),
        message: Array.isArray(message) ? message.join(', ') : String(message),
      };
    }

    // Domain errors
    for (const [ErrorClass, httpStatus] of DOMAIN_ERROR_MAP) {
      if (exception instanceof ErrorClass) {
        return {
          status: httpStatus,
          message: exception instanceof Error ? exception.message : String(exception),
        };
      }
    }

    // Unknown errors — 500, message hidden from client
    return {
      status: HttpStatus.INTERNAL_SERVER_ERROR,
      message: 'Internal server error',
    };
  }
}
```

## Domain Error Classes

Define these in `shared/domain/errors/`. No NestJS imports.

```typescript
// shared/domain/errors/domain.error.ts
export abstract class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

// shared/domain/errors/not-found-error.ts
export class NotFoundError extends DomainError {}

// shared/domain/errors/conflict.error.ts
export class ConflictError extends DomainError {}

// shared/domain/errors/business-rule-violation.error.ts
export class BusinessRuleViolationError extends DomainError {}

// shared/domain/errors/unauthorized.error.ts
export class UnauthorizedError extends DomainError {}

// shared/domain/errors/forbidden.error.ts
export class ForbiddenError extends DomainError {}

// shared/domain/errors/validation.error.ts
export class ValidationError extends DomainError {}
```

## Global Registration

```typescript
// In AppModule or main.ts:
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}

// Or in main.ts (less preferred — bypasses DI):
app.useGlobalFilters(new HttpExceptionFilter());
```

## Module-Level Registration

For bounded context-specific error types:

```typescript
// In <context>.module.ts providers:
{
  provide: APP_FILTER,
  useClass: HttpExceptionFilter,
},

// Or on a controller:
@UseFilters(HttpExceptionFilter)
@Controller('<contexts>')
export class <Context>Controller {}
```

## Throwing Domain Errors (in use cases / handlers)

```typescript
// application/commands/get-<context>.handler.ts
import { NotFoundError } from '@/shared/domain/errors/not-found-error';
import { ForbiddenError } from '@/shared/domain/errors/forbidden.error';

async execute(query: Get<Context>Query) {
  const entity = await this.repository.findById(query.id);
  if (!entity) {
    throw new NotFoundError(`<Context> ${query.id} not found`);
    // → filter maps to HTTP 404
  }

  if (entity.organizationId !== query.organizationId) {
    throw new ForbiddenError('Access denied');
    // → filter maps to HTTP 403
  }

  return entity;
}
```

## Testing the Filter

```typescript
// filters/__tests__/http-exception.filter.spec.ts
import { describe, it, expect, vi } from 'vitest';
import { HttpExceptionFilter } from '../http-exception.filter';
import { NotFoundError } from '@/shared/domain/errors/not-found-error';
import { ArgumentsHost, HttpStatus } from '@nestjs/common';

function mockHost(url = '/test'): ArgumentsHost {
  const reply = { status: vi.fn().mockReturnThis(), send: vi.fn() };
  const request = { url, method: 'GET' };
  return {
    switchToHttp: () => ({
      getResponse: () => reply,
      getRequest: () => request,
    }),
  } as unknown as ArgumentsHost;
}

describe('HttpExceptionFilter', () => {
  const filter = new HttpExceptionFilter();

  it('maps NotFoundError to 404', () => {
    const host = mockHost();
    filter.catch(new NotFoundError('Entity not found'), host);

    const reply = host.switchToHttp().getResponse() as any;
    expect(reply.status).toHaveBeenCalledWith(HttpStatus.NOT_FOUND);
    expect(reply.send).toHaveBeenCalledWith(
      expect.objectContaining({ statusCode: 404, message: 'Entity not found' }),
    );
  });

  it('maps unknown errors to 500 with generic message', () => {
    const host = mockHost();
    filter.catch(new Error('DB connection refused'), host);

    const reply = host.switchToHttp().getResponse() as any;
    expect(reply.status).toHaveBeenCalledWith(HttpStatus.INTERNAL_SERVER_ERROR);
    expect(reply.send).toHaveBeenCalledWith(
      expect.objectContaining({ message: 'Internal server error' }),
    );
  });

  it('handles ValidationPipe errors (HttpException array)', () => {
    const host = mockHost();
    const exception = { getStatus: () => 400, getResponse: () => ({ message: ['name must not be empty'] }) };
    filter.catch(exception as any, host);

    const reply = host.switchToHttp().getResponse() as any;
    expect(reply.status).toHaveBeenCalledWith(400);
    expect(reply.send).toHaveBeenCalledWith(
      expect.objectContaining({ message: 'name must not be empty' }),
    );
  });
});
```

## Response Format

All error responses follow this shape:

```json
{
  "statusCode": 404,
  "message": "Customer abc-123 not found",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "path": "/v1/customers/abc-123"
}
```

## Status Code Reference

| Domain Error                  | HTTP Status |
| ----------------------------- | ----------- |
| `NotFoundError`               | 404         |
| `ConflictError`               | 409         |
| `BusinessRuleViolationError`  | 422         |
| `ValidationError`             | 400         |
| `UnauthorizedError`           | 401         |
| `ForbiddenError`              | 403         |
| Unknown / unhandled           | 500         |

## Rules

- Never leak internal error details to the client in 500 responses.
- Log full stack trace at `error` level for 500 responses; use `debug` for domain errors.
- Domain error classes must not import NestJS or `http-status-codes`.
- The filter is the ONLY place that maps domain errors to HTTP codes.
- Add new domain error types to `DOMAIN_ERROR_MAP` — never add `if` branches in catch clauses.
