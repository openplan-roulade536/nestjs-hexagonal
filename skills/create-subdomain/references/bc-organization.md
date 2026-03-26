# Bounded Context Organization

Guidelines for structuring a bounded context as a single module or splitting it into sub-modules.

---

## When to Keep a Flat BC Structure

Use a flat structure (`<context>/domain`, `<context>/application`, `<context>/infrastructure`) when:

- The BC has 1–2 entities with a single aggregate root
- All entities share the same lifecycle (created, updated, deleted together)
- There are fewer than 5–6 use cases / handlers
- The BC does not own multiple distinct business processes

Example: `invoices/` — one aggregate root (`InvoiceEntity`), straightforward CRUD + status transitions.

```
invoices/
├── domain/
│   ├── entities/
│   ├── value-objects/
│   ├── events/
│   ├── repositories/
│   └── testing/
├── application/
│   ├── dtos/
│   ├── commands/
│   ├── queries/
│   └── ports/
└── infrastructure/
    ├── invoices.module.ts
    ├── controllers/
    ├── database/
    ├── adapters/
    └── listeners/
```

---

## When to Split into Sub-Modules

Split a BC into sub-modules when:

- 3 or more entities have **distinct lifecycles** (created/updated independently)
- Sub-domains have different business owners or deployment cadences
- A single flat module would exceed ~15 use cases / handlers
- Cross-sub-module communication requires explicit contracts (ports)

**Do not split just to organise files.** Splitting increases module complexity. Apply YAGNI.

---

## Sub-Module Structure

Example: `subscription/` splits into `plans/`, `subscriptions/`, `payments/`, and `core/`.

```
subscription/
├── core/                        # shared types, listeners, adapters for the BC
│   ├── domain/
│   │   └── events/              # shared BC-level events (if any)
│   ├── application/
│   │   └── ports/               # ports consumed by sub-modules from external BCs
│   └── infrastructure/
│       ├── subscription-core.module.ts
│       ├── adapters/            # implements external ports
│       └── listeners/           # BC-level event side effects
├── plans/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
│       └── plans.module.ts
├── payments/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
│       └── payments.module.ts
├── subscriptions/
│   ├── domain/
│   ├── application/
│   └── infrastructure/
│       └── subscriptions.module.ts
└── subscription.module.ts       # parent module — imports all sub-modules
```

### Parent module

The parent module aggregates all sub-modules and re-exports the port tokens that external BCs need.

```typescript
// subscription/subscription.module.ts
import { Module } from '@nestjs/common';
import { SubscriptionCoreModule } from './core/infrastructure/subscription-core.module';
import { PlansModule } from './plans/infrastructure/plans.module';
import { PaymentsModule } from './payments/infrastructure/payments.module';
import { SubscriptionsModule } from './subscriptions/infrastructure/subscriptions.module';
import { PLAN_REPOSITORY } from './plans/domain/repositories/plan.repository';

@Module({
  imports: [SubscriptionCoreModule, PlansModule, PaymentsModule, SubscriptionsModule],
  exports: [PLAN_REPOSITORY], // export only what external BCs need
})
export class SubscriptionModule {}
```

---

## Cross-Sub-Module Communication

Sub-modules within the same BC communicate via ports, not direct imports of each other's repositories or services.

**Rule:** The consumer sub-module defines the port; the provider sub-module implements the adapter.

### Step-by-step

1. Consumer sub-module creates a port interface in its `application/ports/`:

```typescript
// subscriptions/application/ports/plan.port.ts
export const PLAN_PORT_TOKEN = Symbol('PlanPort');

export interface PlanPort {
  findById(id: string, organizationId: string): Promise<{
    id: string;
    price: number;
    isActive: boolean;
  } | null>;
}
```

2. Provider sub-module creates an adapter in its `infrastructure/adapters/`:

```typescript
// plans/infrastructure/adapters/plan.adapter.ts
import { Injectable } from '@nestjs/common';
import { PlanPort } from '../../../subscriptions/application/ports/plan.port';
import { PlanRepository } from '../../domain/repositories/plan.repository';
import { PLAN_REPOSITORY } from '../../domain/repositories/plan.repository';
import { Inject } from '@nestjs/common';

@Injectable()
export class PlanAdapter implements PlanPort {
  constructor(
    @Inject(PLAN_REPOSITORY)
    private readonly repository: PlanRepository.Repository,
  ) {}

  async findById(id: string, organizationId: string) {
    const entity = await this.repository.findById(id);
    if (!entity || entity.organizationId !== organizationId) return null;
    return { id: entity.id, price: entity.price.value, isActive: entity.isActive };
  }
}
```

3. Provider's module exports the port token:

```typescript
// plans/infrastructure/plans.module.ts
providers: [
  PlanAdapter,
  { provide: PLAN_PORT_TOKEN, useExisting: PlanAdapter },
],
exports: [PLAN_PORT_TOKEN],
```

4. Consumer imports the provider module and injects the token:

```typescript
// subscriptions/infrastructure/subscriptions.module.ts
imports: [PlansModule],
// ...handler injects @Inject(PLAN_PORT_TOKEN)
```

---

## Core Sub-Module

Use a `core/` sub-module when the BC has:
- Shared event handlers (listeners) that react to events from multiple sub-modules
- Adapters for external BC ports shared by multiple sub-modules
- BC-level domain types or events not owned by a single sub-module

The `core/` module is a sibling of the other sub-modules, imported by the parent module. It does not depend on sibling sub-modules; siblings may depend on `core/`.

---

## Anti-Patterns

| Anti-pattern | What to do instead |
|---|---|
| Sub-module A imports sub-module B's repository directly | Define a port in A, adapter in B |
| Parent module re-exports use cases | Parent exports only port token Symbols |
| Splitting a BC with only 2 entities | Keep it flat; split when complexity demands it |
| `core/` module depending on sibling sub-modules | `core/` depends only on external BCs and shared packages |
| Every entity gets its own sub-module | Group entities by lifecycle, not by count |
