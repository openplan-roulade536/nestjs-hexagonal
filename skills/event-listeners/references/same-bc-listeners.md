# Same-BC Listeners

Listeners that react to events within their own bounded context. Live in `<bc>/infrastructure/listeners/`.

---

## Projection Updater (Redis Read Model)

Updates a denormalized view in Redis for fast query-side reads (CQRS R/W separation).

```typescript
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { Inject } from '@nestjs/common';

import { OrderStatusChangedEvent } from '../../domain/events/order-status-changed.event';
import { REDIS_READ_MODEL_TOKEN, type ReadModelPort } from '../../application/ports/read-model.port';

@EventsHandler(OrderStatusChangedEvent)
export class OrderStatusProjectionHandler implements IEventHandler<OrderStatusChangedEvent> {
  constructor(
    @Inject(REDIS_READ_MODEL_TOKEN)
    private readonly readModel: ReadModelPort,
  ) {}

  async handle(event: OrderStatusChangedEvent): Promise<void> {
    try {
      const existing = await this.readModel.get(`order:${event.aggregateId}`);
      if (!existing) return; // Projection not yet created — skip

      await this.readModel.upsert(`order:${event.aggregateId}`, {
        ...existing,
        status: event.newStatus,
        previousStatus: event.previousStatus,
        updatedAt: event.occurredOn.toISOString(),
      });
    } catch (error) {
      console.error(`[OrderStatusProjection] Failed for ${event.aggregateId}:`, error);
    }
  }
}
```

---

## Audit Log Writer

Records immutable audit trail of domain changes. Useful for compliance.

```typescript
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { Inject } from '@nestjs/common';

import { OrderCreatedEvent } from '../../domain/events/order-created.event';
import { AUDIT_LOG_REPOSITORY, type AuditLogRepository } from '../../domain/repositories/audit-log.repository';

@EventsHandler(OrderCreatedEvent)
export class OrderCreatedAuditHandler implements IEventHandler<OrderCreatedEvent> {
  constructor(
    @Inject(AUDIT_LOG_REPOSITORY)
    private readonly auditRepo: AuditLogRepository,
  ) {}

  async handle(event: OrderCreatedEvent): Promise<void> {
    try {
      await this.auditRepo.log({
        entityType: 'Order',
        entityId: event.aggregateId,
        action: 'CREATED',
        organizationId: event.organizationId,
        payload: { total: event.total, status: event.status },
        occurredAt: event.occurredOn,
      });
    } catch (error) {
      console.error(`[OrderAudit] Failed:`, error);
    }
  }
}
```

---

## Cache Invalidation

Clears stale cache entries when underlying data changes.

```typescript
@EventsHandler(OrderUpdatedEvent)
export class OrderCacheInvalidator implements IEventHandler<OrderUpdatedEvent> {
  constructor(
    @Inject(CACHE_PORT_TOKEN) private readonly cache: CachePort,
  ) {}

  async handle(event: OrderUpdatedEvent): Promise<void> {
    try {
      await this.cache.delete(`order:${event.aggregateId}`);
      await this.cache.delete(`orders:list:${event.organizationId}`);
    } catch (error) {
      console.error(`[OrderCacheInvalidation] Failed:`, error);
    }
  }
}
```

---

## Testing Pattern

```typescript
describe('OrderStatusProjectionHandler', () => {
  let handler: OrderStatusProjectionHandler;
  let readModel: { get: ReturnType<typeof vi.fn>; upsert: ReturnType<typeof vi.fn> };

  beforeEach(() => {
    readModel = { get: vi.fn(), upsert: vi.fn(), delete: vi.fn() };
    handler = new OrderStatusProjectionHandler(readModel);
  });

  it('should update projection with new status', async () => {
    readModel.get.mockResolvedValue({ id: 'o1', status: 'PENDING', total: 100 });

    await handler.handle(new OrderStatusChangedEvent('o1', 'org-1', 'PENDING', 'PAID'));

    expect(readModel.upsert).toHaveBeenCalledWith('order:o1', expect.objectContaining({
      status: 'PAID',
      previousStatus: 'PENDING',
    }));
  });

  it('should skip if projection does not exist yet', async () => {
    readModel.get.mockResolvedValue(null);

    await handler.handle(new OrderStatusChangedEvent('o1', 'org-1', 'PENDING', 'PAID'));

    expect(readModel.upsert).not.toHaveBeenCalled();
  });

  it('should not throw on failure', async () => {
    readModel.get.mockRejectedValue(new Error('Redis down'));

    await expect(handler.handle(
      new OrderStatusChangedEvent('o1', 'org-1', 'PENDING', 'PAID'),
    )).resolves.toBeUndefined();
  });
});
```
