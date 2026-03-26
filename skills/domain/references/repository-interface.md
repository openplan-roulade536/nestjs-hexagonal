# Repository Interface

The repository interface is the domain's **port** for persistence. It defines what persistence operations the domain needs — not how they are implemented. The implementation lives in `infrastructure/repositories/`.

Rules:
- No `@Injectable`, no Prisma, no NestJS imports
- Lives in `domain/repositories/<name>.repository.ts`
- Exported via a `Symbol` token for DI wiring in the module
- Uses the namespace pattern so `SearchParams` and `SearchResult` are scoped to the repository

---

## Full Namespace Pattern

```typescript
// domain/repositories/order.repository.ts
import type { OrderEntity } from '../entities/order.entity';
import {
  SearchParams as DefaultSearchParams,
  SearchResult as DefaultSearchResult,
  SearchableRepositoryInterface,
} from '@/shared/repository-contracts/searchable-repository-contracts';

export namespace OrderRepository {
  /**
   * Filter type: use `string` for simple text search.
   * Expand to an interface when the repository accepts structured filter inputs.
   */
  export type Filter = string;

  /** Extended SearchParams — add custom filter fields here. */
  export class SearchParams extends DefaultSearchParams<Filter> {
    public readonly tenantId?: string;
    public readonly status?: string;

    constructor(props: ConstructorParameters<typeof DefaultSearchParams<Filter>>[0] & {
      tenantId?: string;
      status?: string;
    } = {}) {
      super(props);
      this.tenantId = props.tenantId;
      this.status = props.status;
    }
  }

  export class SearchResult extends DefaultSearchResult<OrderEntity, Filter> {}

  export interface Repository
    extends SearchableRepositoryInterface<
      OrderEntity,
      Filter,
      SearchParams,
      SearchResult
    > {
    /** Basic CRUD */
    findById(id: string): Promise<OrderEntity | null>;
    findByIdOrFail(id: string): Promise<OrderEntity>;
    save(entity: OrderEntity): Promise<void>;
    delete(id: string): Promise<void>;

    /** Domain-specific queries */
    findByTenant(tenantId: string): Promise<OrderEntity[]>;
    findByCustomer(customerId: string, tenantId: string): Promise<OrderEntity[]>;
    findPendingOrders(tenantId: string): Promise<OrderEntity[]>;
    existsById(id: string): Promise<boolean>;

    /** Search */
    searchByTenant(
      params: SearchParams,
      tenantId: string,
    ): Promise<SearchResult>;
  }
}

/** DI token — inject this symbol to receive the repository. */
export const ORDER_REPOSITORY = Symbol('OrderRepository');
```

---

## Simple Repository (no search)

For contexts that do not need pagination or filtering:

```typescript
// domain/repositories/webhook-log.repository.ts
import type { WebhookLogEntity } from '../entities/webhook-log.entity';

export namespace WebhookLogRepository {
  export interface Repository {
    findById(id: string): Promise<WebhookLogEntity | null>;
    findByTenant(tenantId: string, limit?: number): Promise<WebhookLogEntity[]>;
    save(entity: WebhookLogEntity): Promise<void>;
    deleteOlderThan(date: Date): Promise<void>;
  }
}

export const WEBHOOK_LOG_REPOSITORY = Symbol('WebhookLogRepository');
```

---

## Extended SearchParams with Structured Filters

When the filter is more than a text string, define an interface and extend `SearchParams`:

```typescript
// domain/repositories/transaction.repository.ts
import type { TransactionEntity } from '../entities/transaction.entity';
import {
  SearchParams as DefaultSearchParams,
  SearchResult as DefaultSearchResult,
  SearchableRepositoryInterface,
} from '@/shared/repository-contracts/searchable-repository-contracts';

export interface TransactionFilter {
  text?: string;
  status?: string;
  customerId?: string;
  dateFrom?: Date;
  dateTo?: Date;
  minAmount?: number;
  maxAmount?: number;
}

export namespace TransactionRepository {
  export type Filter = TransactionFilter;

  export class SearchParams extends DefaultSearchParams<Filter> {
    public readonly tenantId: string;

    constructor(
      props: ConstructorParameters<typeof DefaultSearchParams<Filter>>[0] & {
        tenantId: string;
      },
    ) {
      super(props);
      this.tenantId = props.tenantId;
    }
  }

  export class SearchResult extends DefaultSearchResult<TransactionEntity, Filter> {}

  export interface Repository
    extends SearchableRepositoryInterface<
      TransactionEntity, Filter, SearchParams, SearchResult
    > {
    findById(id: string, tenantId: string): Promise<TransactionEntity | null>;
    save(entity: TransactionEntity): Promise<void>;
    findByExternalId(externalId: string, tenantId: string): Promise<TransactionEntity | null>;
  }
}

export const TRANSACTION_REPOSITORY = Symbol('TransactionRepository');
```

---

## Wiring in NestJS Module

```typescript
// infrastructure/order.module.ts
import { Module } from '@nestjs/common';
import { ORDER_REPOSITORY } from '../domain/repositories/order.repository';
import { PrismaOrderRepository } from './repositories/prisma-order.repository';

@Module({
  providers: [
    PrismaOrderRepository,
    {
      provide: ORDER_REPOSITORY,
      useExisting: PrismaOrderRepository,
    },
  ],
  exports: [ORDER_REPOSITORY], // export the TOKEN, never the Prisma class
})
export class OrderModule {}
```

---

## Custom Query Method Guidelines

| Method naming | When to use |
|---|---|
| `findById(id)` | Always — primary lookup |
| `findByIdOrFail(id)` | When caller expects entity to exist; throws `NotFoundError` |
| `findBy<Field>(value, tenantId)` | Domain-specific lookup with tenant isolation |
| `existsBy<Field>(value, tenantId)` | Cheap existence check (COUNT query) |
| `findPending<Entity>(tenantId)` | Filtered status query — include status in the method name |
| `search(params, tenantId)` | Paginated text/filter search |
| `save(entity)` | Upsert — insert or update based on entity.id |
| `delete(id, tenantId)` | Hard delete with tenant guard |

**Multi-tenant rule:** Every query that touches a tenant's data MUST include `tenantId` as a parameter. Never expose a method that can leak cross-tenant data.

---

## In-Memory Implementation (for tests)

Use the shared `InMemoryRepository` base or implement directly:

```typescript
// domain/testing/repositories/in-memory-order.repository.ts
import { OrderEntity } from '../../entities/order.entity';
import type { OrderRepository } from '../../repositories/order.repository';

export class InMemoryOrderRepository implements OrderRepository.Repository {
  public items: OrderEntity[] = [];
  public sortableFields = ['createdAt', 'totalAmount'];

  async findById(id: string): Promise<OrderEntity | null> {
    return this.items.find((e) => e.id === id) ?? null;
  }

  async findByIdOrFail(id: string): Promise<OrderEntity> {
    const entity = await this.findById(id);
    if (!entity) throw new Error(`OrderEntity ${id} not found`);
    return entity;
  }

  async findByTenant(tenantId: string): Promise<OrderEntity[]> {
    return this.items.filter((e) => e.tenantId === tenantId);
  }

  async save(entity: OrderEntity): Promise<void> {
    const index = this.items.findIndex((e) => e.id === entity.id);
    if (index >= 0) {
      this.items[index] = entity;
    } else {
      this.items.push(entity);
    }
  }

  async delete(id: string): Promise<void> {
    this.items = this.items.filter((e) => e.id !== id);
  }

  async existsById(id: string): Promise<boolean> {
    return this.items.some((e) => e.id === id);
  }

  async findByCustomer(customerId: string, tenantId: string): Promise<OrderEntity[]> {
    return this.items.filter(
      (e) => e.customerId === customerId && e.tenantId === tenantId,
    );
  }

  async findPendingOrders(tenantId: string): Promise<OrderEntity[]> {
    return this.items.filter(
      (e) => e.tenantId === tenantId && e.status.isPending(),
    );
  }

  async search(params: OrderRepository.SearchParams): Promise<OrderRepository.SearchResult> {
    const items = this.items.filter((e) =>
      params.tenantId ? e.tenantId === params.tenantId : true,
    );
    return new OrderRepository.SearchResult({
      items,
      total: items.length,
      currentPage: params.page ?? 1,
      perPage: params.perPage ?? 15,
      sort: params.sort ?? null,
      sortDir: params.sortDir ?? null,
      filter: params.filter ?? null,
    });
  }

  async searchByTenant(
    params: OrderRepository.SearchParams,
    tenantId: string,
  ): Promise<OrderRepository.SearchResult> {
    return this.search({ ...params, tenantId } as OrderRepository.SearchParams);
  }
}
```

Use `InMemoryOrderRepository` in use-case and handler unit tests — no database, no mocking framework needed.
