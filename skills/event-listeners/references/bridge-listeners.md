# Bridge Listeners

Transform domain events into external outputs: WebSocket broadcast, message broker publish, email, webhook.

---

## WebSocket Broadcast Bridge

Emits event to connected frontend clients via `WsGatewayPort`.

```typescript
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { Inject, Logger } from '@nestjs/common';

import { OrderCreatedEvent } from '../../domain/events/order-created.event';
import { WS_GATEWAY_TOKEN, type WsGatewayPort } from '@/shared/ports/ws-gateway.port';

@EventsHandler(OrderCreatedEvent)
export class OrderCreatedBroadcastHandler implements IEventHandler<OrderCreatedEvent> {
  private readonly logger = new Logger(OrderCreatedBroadcastHandler.name);

  constructor(
    @Inject(WS_GATEWAY_TOKEN) private readonly gateway: WsGatewayPort,
  ) {}

  async handle(event: OrderCreatedEvent): Promise<void> {
    try {
      this.gateway.emitToOrganization(event.organizationId, 'order:created', {
        id: event.aggregateId,
        total: event.total,
        status: event.status,
      });
    } catch (error) {
      this.logger.error(`Broadcast failed for order ${event.aggregateId}`, error);
    }
  }
}
```

**When to enrich payload:**
If the event doesn't carry enough data for the frontend, fetch from repository BEFORE emitting:

```typescript
async handle(event: OrderCreatedEvent): Promise<void> {
  try {
    // Enrich: fetch full order with relations
    const order = await this.orderRepo.findById(event.aggregateId);
    if (!order) return;

    this.gateway.emitToOrganization(event.organizationId, 'order:created', {
      id: order.id,
      total: order.total,
      status: order.status,
      customer: { id: order.customerId, name: order.customerName },
      items: order.items.map(i => ({ name: i.name, quantity: i.quantity })),
    });
  } catch (error) {
    this.logger.error(`Broadcast failed`, error);
  }
}
```

See `websocket-broadcasting` skill for full gateway setup.

---

## Message Broker Bridge (RabbitMQ / Kafka)

Publishes an integration event to a message broker for consumption by external microservices.

```typescript
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { Inject, Logger } from '@nestjs/common';

import { OrderPaidEvent } from '../../domain/events/order-paid.event';
import { MESSAGE_BROKER_TOKEN, type MessageBrokerPort } from '@/shared/ports/message-broker.port';

@EventsHandler(OrderPaidEvent)
export class OrderPaidBrokerPublisher implements IEventHandler<OrderPaidEvent> {
  private readonly logger = new Logger(OrderPaidBrokerPublisher.name);

  constructor(
    @Inject(MESSAGE_BROKER_TOKEN)
    private readonly broker: MessageBrokerPort,
  ) {}

  async handle(event: OrderPaidEvent): Promise<void> {
    try {
      // Transform domain event → integration event (different schema)
      await this.broker.publish('order.paid', {
        orderId: event.aggregateId,
        organizationId: event.organizationId,
        amount: event.amount,
        paidAt: event.occurredOn.toISOString(),
        // Only include what external consumers need
      });

      this.logger.log(`Published order.paid for ${event.aggregateId}`);
    } catch (error) {
      this.logger.error(`Broker publish failed for ${event.aggregateId}`, error);
      // Consider: retry queue, dead letter, or manual intervention flag
    }
  }
}
```

**MessageBrokerPort interface:**
```typescript
export const MESSAGE_BROKER_TOKEN = Symbol('MessageBroker');

export interface MessageBrokerPort {
  publish(routingKey: string, payload: Record<string, unknown>): Promise<void>;
}
```

**Domain Event vs Integration Event:**
- **Domain event**: internal, rich payload, tied to aggregate
- **Integration event**: external contract, minimal payload, stable schema
- The bridge listener transforms one into the other

---

## Email Notification Bridge

```typescript
@EventsHandler(OrderShippedEvent)
export class OrderShippedEmailHandler implements IEventHandler<OrderShippedEvent> {
  constructor(
    @Inject(EMAIL_PORT_TOKEN) private readonly email: EmailPort,
    @Inject(ORDER_REPOSITORY) private readonly orderRepo: OrderRepository.Repository,
  ) {}

  async handle(event: OrderShippedEvent): Promise<void> {
    try {
      const order = await this.orderRepo.findById(event.aggregateId);
      if (!order?.customerEmail) return;

      await this.email.send({
        to: order.customerEmail,
        template: 'order-shipped',
        data: {
          orderNumber: order.orderNumber,
          trackingCode: event.trackingCode,
          estimatedDelivery: event.estimatedDelivery,
        },
      });
    } catch (error) {
      console.error(`[OrderShippedEmail] Failed:`, error);
    }
  }
}
```

---

## Webhook Bridge

```typescript
@EventsHandler(PaymentReceivedEvent)
export class PaymentReceivedWebhookHandler implements IEventHandler<PaymentReceivedEvent> {
  constructor(
    @Inject(WEBHOOK_PORT_TOKEN) private readonly webhook: WebhookPort,
  ) {}

  async handle(event: PaymentReceivedEvent): Promise<void> {
    try {
      await this.webhook.dispatch(event.organizationId, 'payment.received', {
        paymentId: event.aggregateId,
        amount: event.amount,
        method: event.method,
        occurredAt: event.occurredOn.toISOString(),
      });
    } catch (error) {
      console.error(`[PaymentWebhook] Failed:`, error);
    }
  }
}
```

---

## Bridge Listener Rules

1. **try/catch mandatory** — broadcast/publish failure must never break the domain flow
2. **Transform payload** — frontend/external consumers have different schemas than domain events
3. **Minimal payload** — only send what the consumer needs, not the entire aggregate
4. **Ports for external services** — inject via TOKEN, never import infrastructure directly
5. **One listener per output** — don't combine WS broadcast + email in one handler (SRP)
6. **Idempotency in consumer** — bridge listeners may retry, consumers must handle duplicates

---

## Testing

```typescript
describe('OrderCreatedBroadcastHandler', () => {
  let handler: OrderCreatedBroadcastHandler;
  let gateway: { emitToOrganization: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    gateway = {
      emitToOrganization: vi.fn(),
      emitToUser: vi.fn(),
      emitGlobal: vi.fn(),
    };
    handler = new OrderCreatedBroadcastHandler(gateway);
  });

  it('should emit to organization room', async () => {
    const event = new OrderCreatedEvent('o1', 'org-1', 1000, 'PENDING');

    await handler.handle(event);

    expect(gateway.emitToOrganization).toHaveBeenCalledWith(
      'org-1',
      'order:created',
      { id: 'o1', total: 1000, status: 'PENDING' },
    );
  });

  it('should not throw on gateway failure', async () => {
    gateway.emitToOrganization.mockImplementation(() => { throw new Error('WS down'); });

    await expect(handler.handle(
      new OrderCreatedEvent('o1', 'org-1', 1000, 'PENDING'),
    )).resolves.toBeUndefined();
  });
});
```
