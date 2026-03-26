# Adapter Patterns — Full Template

Adapters implement port interfaces defined in other bounded contexts. They translate between the external contract and the local domain without leaking implementation details.

## When to Use

Create an adapter when:
- Another bounded context declares a port interface it needs (e.g., `CustomerPort` needed by `PaymentsModule`)
- You want to expose a stable cross-module API without exporting use cases or repositories

## Port Interface (defined in the consuming context)

```typescript
// <consuming-context>/application/ports/<context>.port.ts
export const <CONTEXT>_PORT = Symbol('<Context>Port');

export interface <Context>Props {
  id: string;
  name: string;
  organizationId: string;
  // lean shape — only what the consumer needs
}

export interface <Context>Port {
  findById(id: string): Promise<<Context>Props | null>;
  findByOrganization(organizationId: string): Promise<<Context>Props[]>;
}
```

## Adapter Implementation (in the providing context)

```typescript
// <providing-context>/infrastructure/adapters/<context>.adapter.ts
import { Injectable } from '@nestjs/common';
import type {
  <Context>Port,
  <Context>Props,
} from '@/<consuming-context>/application/ports/<context>.port';
import { <Context>Service } from '../services/<context>.service';
import type { <Context>Entity } from '../../domain/entities/<context>.entity';

@Injectable()
export class <Context>Adapter implements <Context>Port {
  constructor(private readonly service: <Context>Service) {}

  async findById(id: string): Promise<<Context>Props | null> {
    const entity = await this.service.findById(id);
    return entity ? this.toProps(entity) : null;
  }

  async findByOrganization(organizationId: string): Promise<<Context>Props[]> {
    const entities = await this.service.findByOrganization(organizationId);
    return entities.map(this.toProps);
  }

  private toProps(entity: <Context>Entity): <Context>Props {
    return {
      id: entity.id,
      name: entity.name,
      organizationId: entity.organizationId,
    };
  }
}
```

## Module Registration

```typescript
// In <providing-context>.module.ts providers:
<Context>Adapter,
{
  provide: <CONTEXT>_PORT,
  useExisting: <Context>Adapter,
},

// In exports:
exports: [<CONTEXT>_REPOSITORY, <CONTEXT>_PORT],
```

## Consuming the Port

```typescript
// <consuming-context>/application/commands/create-order.handler.ts
import { Inject } from '@nestjs/common';
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import type { <Context>Port } from '../ports/<context>.port';
import { <CONTEXT>_PORT } from '../ports/<context>.port';

@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(
    @Inject(<CONTEXT>_PORT)
    private readonly <context>Port: <Context>Port,
  ) {}

  async execute(command: CreateOrderCommand): Promise<{ id: string }> {
    const <context> = await this.<context>Port.findById(command.<context>Id);
    if (!<context>) throw new NotFoundException('<Context> not found');
    // ...
  }
}
```

## Infrastructure Adapter (for external services)

When the port connects to an external API rather than another bounded context:

```typescript
// infrastructure/adapters/payment-gateway.adapter.ts
import { Injectable } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';
import type { PaymentGatewayPort } from '../../application/ports/payment-gateway.port';

@Injectable()
export class PaymentGatewayAdapter implements PaymentGatewayPort {
  constructor(private readonly http: HttpService) {}

  async charge(amount: number, token: string): Promise<{ transactionId: string }> {
    const response = await firstValueFrom(
      this.http.post('/charges', { amount, token }),
    );
    return { transactionId: response.data.id };
  }
}

// Module registration:
// PaymentGatewayAdapter,
// { provide: PAYMENT_GATEWAY_PORT, useExisting: PaymentGatewayAdapter },
```

## Testing Adapters

```typescript
// __tests__/<context>.adapter.spec.ts
import { describe, it, beforeEach, expect, vi } from 'vitest';
import { <Context>Adapter } from '../<context>.adapter';

describe('<Context>Adapter', () => {
  const mockService = {
    findById: vi.fn(),
    findByOrganization: vi.fn(),
  };

  let adapter: <Context>Adapter;

  beforeEach(() => {
    vi.clearAllMocks();
    adapter = new <Context>Adapter(mockService as any);
  });

  it('returns null when entity not found', async () => {
    mockService.findById.mockResolvedValue(null);
    expect(await adapter.findById('non-existent')).toBeNull();
  });

  it('maps entity to props shape', async () => {
    const entity = <Context>Entity.restore(
      { organizationId: 'org-1', name: 'Test', createdAt: new Date() },
      'entity-id',
    );
    mockService.findById.mockResolvedValue(entity);

    const result = await adapter.findById('entity-id');

    expect(result).toStrictEqual({
      id: 'entity-id',
      name: 'Test',
      organizationId: 'org-1',
    });
  });
});
```

## Rules

- Adapters map domain entities to lean prop shapes — never return domain entities directly.
- Adapter `toProps()` is a private method — not part of the port interface.
- Always use `useExisting` (not `useClass`) when binding adapter to port token.
- The port interface lives in the CONSUMING context's `application/ports/` directory.
- The adapter class lives in the PROVIDING context's `infrastructure/adapters/` directory.
