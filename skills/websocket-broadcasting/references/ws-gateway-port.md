# WsGatewayPort — Hexagonal Abstraction

The port interface is the only contract that domain and application layers may reference. The concrete gateway class is an infrastructure adapter that must never be imported outside the infrastructure layer.

---

## Port Interface

```typescript
// shared/ports/ws-gateway.port.ts

export const WS_GATEWAY_TOKEN = Symbol('WsGateway');

export interface WsGatewayPort {
  /**
   * Emit to all connected clients of an organization.
   * Use for all business events — the default routing target.
   */
  emitToOrganization(organizationId: string, event: string, data: Record<string, unknown>): void;

  /**
   * Emit to a specific user across all their connected sockets.
   * Use for personal notifications, DMs, or targeted updates.
   */
  emitToUser(userId: string, event: string, data: Record<string, unknown>): void;

  /**
   * Emit to all connected clients regardless of organization.
   * Use only for system-level events with no tenant-scoped data.
   */
  emitGlobal(event: string, data: Record<string, unknown>): void;
}
```

---

## Domain-Specific Port Extension

When a bounded context needs type-safe, named broadcast methods, extend the base port in `application/ports/`:

```typescript
// orders/application/ports/order-ws.port.ts
import type { WsGatewayPort } from '@/shared/ports/ws-gateway.port';

export const ORDER_WS_PORT_TOKEN = Symbol('OrderWsPort');

export interface OrderSummary {
  id: string;
  customerId: string;
  total: number;
  status: string;
  createdAt: string;
  organizationId: string;
}

export interface OrderWsPort extends WsGatewayPort {
  broadcastOrderCreated(organizationId: string, order: OrderSummary): void;
  broadcastOrderCancelled(organizationId: string, orderId: string, reason: string): void;
}
```

The gateway implements both `WsGatewayPort` and `OrderWsPort`. Register both tokens in the module:

```typescript
providers: [
  AppGateway,
  { provide: WS_GATEWAY_TOKEN, useExisting: AppGateway },
  { provide: ORDER_WS_PORT_TOKEN, useExisting: AppGateway },
],
exports: [WS_GATEWAY_TOKEN, ORDER_WS_PORT_TOKEN],
```

---

## Module Wiring

### Gateway module (infrastructure)

```typescript
// infrastructure/gateway.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';

import { WS_GATEWAY_TOKEN } from '@/shared/ports/ws-gateway.port';
import { AppGateway } from './gateways/app.gateway';

@Module({
  imports: [JwtModule.register({ secret: process.env.JWT_SECRET })],
  providers: [
    AppGateway,
    { provide: WS_GATEWAY_TOKEN, useExisting: AppGateway },
  ],
  exports: [WS_GATEWAY_TOKEN],
})
export class GatewayModule {}
```

### Feature module consuming the port

```typescript
// orders/infrastructure/orders.module.ts
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';
import { GatewayModule } from '@/shared/infrastructure/gateway.module';
import { OrderCreatedBroadcastHandler } from './listeners/order-created-broadcast.handler';

@Module({
  imports: [CqrsModule, GatewayModule],
  providers: [
    OrderCreatedBroadcastHandler,
    // ... other providers
  ],
})
export class OrdersModule {}
```

---

## Injecting the Port

In CQRS handlers and infrastructure services:

```typescript
import { Inject } from '@nestjs/common';
import { WS_GATEWAY_TOKEN } from '@/shared/ports/ws-gateway.port';
import type { WsGatewayPort } from '@/shared/ports/ws-gateway.port';

@EventsHandler(SomeEvent)
export class SomeBroadcastHandler implements IEventHandler<SomeEvent> {
  constructor(
    @Inject(WS_GATEWAY_TOKEN) private readonly gateway: WsGatewayPort,
  ) {}
}
```

---

## Mock Implementation for Testing

```typescript
// test/mocks/mock-ws-gateway.ts
import type { WsGatewayPort } from '@/shared/ports/ws-gateway.port';

export class MockWsGateway implements WsGatewayPort {
  readonly emittedToOrganization: Array<{
    organizationId: string;
    event: string;
    data: Record<string, unknown>;
  }> = [];

  readonly emittedToUser: Array<{
    userId: string;
    event: string;
    data: Record<string, unknown>;
  }> = [];

  readonly emittedGlobal: Array<{ event: string; data: Record<string, unknown> }> = [];

  emitToOrganization(organizationId: string, event: string, data: Record<string, unknown>): void {
    this.emittedToOrganization.push({ organizationId, event, data });
  }

  emitToUser(userId: string, event: string, data: Record<string, unknown>): void {
    this.emittedToUser.push({ userId, event, data });
  }

  emitGlobal(event: string, data: Record<string, unknown>): void {
    this.emittedGlobal.push({ event, data });
  }

  // Query helpers for assertions
  lastOrgEmit(): typeof this.emittedToOrganization[0] | undefined {
    return this.emittedToOrganization.at(-1);
  }

  lastUserEmit(): typeof this.emittedToUser[0] | undefined {
    return this.emittedToUser.at(-1);
  }

  reset(): void {
    this.emittedToOrganization.length = 0;
    this.emittedToUser.length = 0;
    this.emittedGlobal.length = 0;
  }
}
```

---

## Testing Patterns

### Unit test — dedicated handler

```typescript
// __tests__/order-created-broadcast.handler.spec.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { MockWsGateway } from '@/test/mocks/mock-ws-gateway';
import { OrderInMemoryRepository } from '../../database/in-memory/order-in-memory.repository';
import { OrderCreatedBroadcastHandler } from '../order-created-broadcast.handler';
import { OrderCreatedEvent } from '../../../domain/events/order-created.event';
import { makeOrder } from '@/test/factories/order.factory';

describe('OrderCreatedBroadcastHandler', () => {
  let gateway: MockWsGateway;
  let repo: OrderInMemoryRepository;
  let handler: OrderCreatedBroadcastHandler;

  beforeEach(() => {
    gateway = new MockWsGateway();
    repo = new OrderInMemoryRepository();
    handler = new OrderCreatedBroadcastHandler(gateway, repo);
  });

  it('emits order:created to the organization when order exists', async () => {
    const order = makeOrder({ id: 'order-1', organizationId: 'org-1' });
    await repo.save(order);

    await handler.handle(new OrderCreatedEvent('order-1', 'org-1'));

    const emit = gateway.lastOrgEmit();
    expect(emit).toBeDefined();
    expect(emit?.organizationId).toBe('org-1');
    expect(emit?.event).toBe('order:created');
    expect(emit?.data).toMatchObject({ id: 'order-1', organizationId: 'org-1' });
  });

  it('does nothing when the order is not found', async () => {
    await handler.handle(new OrderCreatedEvent('order-missing', 'org-1'));

    expect(gateway.emittedToOrganization).toHaveLength(0);
  });

  it('does nothing when organizationId is absent', async () => {
    await handler.handle(new OrderCreatedEvent('order-1', ''));

    expect(gateway.emittedToOrganization).toHaveLength(0);
  });
});
```

### Integration test — tenant isolation

```typescript
import { io } from 'socket.io-client';

it('org-A socket does not receive events for org-B', async () => {
  const socketA = io('http://localhost:3000/events', { auth: { token: tokenForOrgA } });
  const socketB = io('http://localhost:3000/events', { auth: { token: tokenForOrgB } });

  const receivedByA: unknown[] = [];
  socketA.on('order:created', (data) => receivedByA.push(data));

  // Emit only to org-B
  gateway.emitToOrganization('org-b-id', 'order:created', { id: 'order-b' });

  await new Promise((r) => setTimeout(r, 100));

  expect(receivedByA).toHaveLength(0);

  socketA.disconnect();
  socketB.disconnect();
});
```
