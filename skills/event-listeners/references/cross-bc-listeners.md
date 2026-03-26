# Cross-BC Listeners

Listeners that react to events emitted by OTHER bounded contexts. Live in the CONSUMING BC, NOT the emitting one.

---

## Placement Rule

```
OrderBC (emitter)                    BillingBC (consumer)
  domain/events/                       infrastructure/listeners/
    order-created.event.ts               order-created-invoice.handler.ts
                                         ← imports event from OrderBC
                                         ← dispatches CreateInvoiceCommand in BillingBC
```

**Why consumer owns the listener:**
- Listener logic belongs with the code that uses it
- Reduces public API surface of emitting BC
- Each BC owns its reactions to external events
- Emitting BC doesn't need to know who listens

---

## Full Template

```typescript
// billing/infrastructure/listeners/order-created-invoice.handler.ts
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { CommandBus } from '@nestjs/cqrs';
import { Logger } from '@nestjs/common';

// Import event from emitting BC — SAFE (events are pure data, no deps)
import { OrderCreatedEvent } from '@/enterprise/orders/domain/events/order-created.event';

// Import own command
import { CreateInvoiceCommand } from '../../application/commands/create-invoice.command';

@EventsHandler(OrderCreatedEvent)
export class OrderCreatedInvoiceHandler implements IEventHandler<OrderCreatedEvent> {
  private readonly logger = new Logger(OrderCreatedInvoiceHandler.name);

  constructor(private readonly commandBus: CommandBus) {}

  async handle(event: OrderCreatedEvent): Promise<void> {
    try {
      // Guard: skip if not relevant to billing
      if (event.total <= 0) {
        this.logger.debug(`Skipping zero-amount order ${event.aggregateId}`);
        return;
      }

      // Dispatch command in OWN bounded context
      await this.commandBus.execute(
        new CreateInvoiceCommand(
          event.organizationId,
          event.aggregateId,
          event.total,
          event.customerName,
        ),
      );

      this.logger.log(`Invoice created for order ${event.aggregateId}`);
    } catch (error) {
      // NEVER re-throw — cross-BC failure must not affect emitting BC
      this.logger.error(
        `Failed to create invoice for order ${event.aggregateId}`,
        error instanceof Error ? error.stack : String(error),
      );
    }
  }
}
```

---

## Why Event Imports Are Safe

Events are immutable value objects. They have NO transitive dependencies:

```typescript
// orders/domain/events/order-created.event.ts
import { IEvent } from '@nestjs/cqrs';

export class OrderCreatedEvent implements IEvent {
  constructor(
    public readonly aggregateId: string,
    public readonly organizationId: string,
    public readonly total: number,
    public readonly status: string,
    public readonly customerName: string,
    public readonly occurredOn: Date = new Date(),
  ) {}
}
```

This class imports ONLY `IEvent` from `@nestjs/cqrs`. No entity, no repository, no service. Safe to import from any BC without circular dependencies.

---

## Cross-BC Listener Patterns

### Pattern A: Dispatch Own Command (preferred)

Listener transforms the external event into a command in its own BC.

```typescript
async handle(event: OrderCreatedEvent): Promise<void> {
  await this.commandBus.execute(new CreateInvoiceCommand(...));
}
```

**Best for:** When the reaction requires its own domain logic (create entity, validate, persist).

### Pattern B: Direct Service Call (simple cases)

Listener calls an application service directly.

```typescript
async handle(event: OrderCreatedEvent): Promise<void> {
  await this.notificationService.sendOrderConfirmation(event.organizationId, event.aggregateId);
}
```

**Best for:** Simple side effects without domain logic (send notification, log analytics).

### Pattern C: Multiple Events from Same BC

When a BC reacts to multiple events from another BC, group listeners in a directory:

```
billing/infrastructure/listeners/orders/
  order-created-invoice.handler.ts
  order-refunded-credit-note.handler.ts
  order-cancelled-void-invoice.handler.ts
```

---

## Module Wiring

Register in the CONSUMING module:

```typescript
// billing/infrastructure/billing.module.ts
import { CqrsModule } from '@nestjs/cqrs';

const CrossBcListeners = [
  OrderCreatedInvoiceHandler,
  OrderRefundedCreditNoteHandler,
];

@Module({
  imports: [CqrsModule],
  providers: [
    ...CrossBcListeners,
    // ... own providers
  ],
})
export class BillingModule {}
```

**No need to import OrdersModule** — the event flows through the global EventBus. Only the event CLASS is imported.

---

## Circular Dependency Prevention

```
SAFE:
  OrderBC → emits OrderCreatedEvent
  BillingBC → imports OrderCreatedEvent (pure data class)
  BillingBC → dispatches CreateInvoiceCommand (own BC)

UNSAFE (never do):
  OrderBC → imports BillingService (creates circular dep)
  BillingBC → imports OrderRepository (crosses BC boundary)
```

**Rule:** Cross-BC listeners dispatch commands in their OWN BC. They never call services or repositories from the emitting BC.

---

## Testing

```typescript
describe('OrderCreatedInvoiceHandler', () => {
  let handler: OrderCreatedInvoiceHandler;
  let commandBus: { execute: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    commandBus = { execute: vi.fn() };
    handler = new OrderCreatedInvoiceHandler(commandBus as any);
  });

  it('should dispatch CreateInvoiceCommand', async () => {
    const event = new OrderCreatedEvent('order-1', 'org-1', 500, 'PENDING', 'John');

    await handler.handle(event);

    expect(commandBus.execute).toHaveBeenCalledWith(
      expect.objectContaining({
        organizationId: 'org-1',
        orderId: 'order-1',
        amount: 500,
      }),
    );
  });

  it('should skip zero-amount orders', async () => {
    const event = new OrderCreatedEvent('order-1', 'org-1', 0, 'PENDING', 'John');

    await handler.handle(event);

    expect(commandBus.execute).not.toHaveBeenCalled();
  });

  it('should not throw on command failure', async () => {
    commandBus.execute.mockRejectedValue(new Error('DB error'));
    const event = new OrderCreatedEvent('order-1', 'org-1', 500, 'PENDING', 'John');

    await expect(handler.handle(event)).resolves.toBeUndefined();
  });
});
```
