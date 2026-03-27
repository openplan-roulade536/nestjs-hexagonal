# Domain Error Patterns

Domain errors are specialized exceptions that extend the shared error hierarchy with context-specific information. They live in `<bc>/domain/errors/` and carry domain data that helps with debugging, logging, and user-facing messages.

---

## Shared Base Hierarchy (in `shared/domain/errors/`)

```typescript
// Base — all domain errors extend this
export class DomainError extends Error {
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}

// HTTP-mappable errors (mapped by HttpExceptionFilter)
export class NotFoundError extends DomainError {}
export class ConflictError extends DomainError {}
export class BadRequestError extends DomainError {}
export class ForbiddenError extends DomainError {}
export class ValidationError extends DomainError {}
export class InvalidArgumentError extends DomainError {}

// Structured validation errors (field-level)
export class EntityValidationError extends DomainError {
  constructor(public readonly errors: Record<string, string[]>) {
    super('Entity validation failed');
  }
}
```

---

## Specialized Domain Errors (per bounded context)

Create in `<bc>/domain/errors/`. Each error extends a base error and adds domain context.

### Entity Not Found

```typescript
// domain/errors/order-not-found.error.ts
import { NotFoundError } from '@/shared/domain/errors';

export class OrderNotFoundError extends NotFoundError {
  constructor(
    public readonly orderId: string,
    public readonly organizationId?: string,
  ) {
    super(`Order ${orderId} not found`);
  }
}
```

**Usage in repository or use case:**
```typescript
const order = await this.repo.findById(id);
if (!order) {
  throw new OrderNotFoundError(id, organizationId);
}
```

### Business Rule Violation

```typescript
// domain/errors/insufficient-balance.error.ts
import { DomainError } from '@/shared/domain/errors';

export class InsufficientBalanceError extends DomainError {
  constructor(
    public readonly currentBalance: number,
    public readonly requiredAmount: number,
  ) {
    super(
      `Insufficient balance: has ${currentBalance}, needs ${requiredAmount}`,
    );
  }
}
```

### Invalid State Transition

```typescript
// domain/errors/invalid-order-status-transition.error.ts
import { DomainError } from '@/shared/domain/errors';

export class InvalidOrderStatusTransitionError extends DomainError {
  constructor(
    public readonly currentStatus: string,
    public readonly targetStatus: string,
    public readonly orderId: string,
  ) {
    super(
      `Cannot transition order ${orderId} from ${currentStatus} to ${targetStatus}`,
    );
  }
}
```

**Usage in State Machine VO:**
```typescript
transitionTo(target: OrderStatus): OrderStatusVO {
  if (!this.canTransitionTo(target)) {
    throw new InvalidOrderStatusTransitionError(
      this.value, target, 'unknown',
    );
  }
  return new OrderStatusVO(target);
}
```

### Duplicate / Conflict

```typescript
// domain/errors/duplicate-order.error.ts
import { ConflictError } from '@/shared/domain/errors';

export class DuplicateOrderError extends ConflictError {
  constructor(
    public readonly externalId: string,
    public readonly organizationId: string,
  ) {
    super(
      `Order with external ID ${externalId} already exists in organization ${organizationId}`,
    );
  }
}
```

### Authorization / Permission

```typescript
// domain/errors/order-not-owned.error.ts
import { ForbiddenError } from '@/shared/domain/errors';

export class OrderNotOwnedError extends ForbiddenError {
  constructor(
    public readonly orderId: string,
    public readonly requestedBy: string,
  ) {
    super(`User ${requestedBy} does not own order ${orderId}`);
  }
}
```

### Validation (Field-Level)

```typescript
// domain/errors/invalid-order-amount.error.ts
import { InvalidArgumentError } from '@/shared/domain/errors';

export class InvalidOrderAmountError extends InvalidArgumentError {
  constructor(public readonly amount: number) {
    super(`Order amount must be positive, got ${amount}`);
  }
}
```

---

## Directory Structure

```
<bc>/domain/errors/
  ├── order-not-found.error.ts
  ├── duplicate-order.error.ts
  ├── insufficient-balance.error.ts
  ├── invalid-order-status-transition.error.ts
  ├── order-not-owned.error.ts
  ├── invalid-order-amount.error.ts
  └── index.ts                    # barrel export
```

**Barrel export:**
```typescript
// domain/errors/index.ts
export { OrderNotFoundError } from './order-not-found.error';
export { DuplicateOrderError } from './duplicate-order.error';
export { InsufficientBalanceError } from './insufficient-balance.error';
export { InvalidOrderStatusTransitionError } from './invalid-order-status-transition.error';
export { OrderNotOwnedError } from './order-not-owned.error';
export { InvalidOrderAmountError } from './invalid-order-amount.error';
```

---

## Naming Convention

```
<Entity><Condition>Error

Examples:
  OrderNotFoundError
  CustomerAlreadyExistsError
  InsufficientBalanceError
  InvalidOrderStatusTransitionError
  PaymentAlreadyProcessedError
  WithdrawalRequestAlreadyPendingError
```

**Rule:** Error name describes WHAT went wrong, not WHERE it happened.

---

## When to Create a Specialized Error vs Use Generic

| Scenario | Use |
|---|---|
| Entity not found — need entity ID in error | `OrderNotFoundError(orderId)` |
| Generic not found — no extra context needed | `NotFoundError('Resource not found')` |
| State machine transition invalid | `InvalidOrderStatusTransitionError(current, target)` |
| Business rule with numeric context | `InsufficientBalanceError(balance, required)` |
| Simple validation — bad input format | `InvalidArgumentError('Email must be valid')` |
| Complex validation — multiple fields | `EntityValidationError({ email: ['invalid'], name: ['required'] })` |

**Rule:** Specialize when the error carries domain context that helps debugging. Don't specialize for the sake of it — `NotFoundError` is fine when the message is sufficient.

---

## Error Filter Mapping

Specialized errors inherit the HTTP status from their base class:

```
OrderNotFoundError     extends NotFoundError     → 404
DuplicateOrderError    extends ConflictError     → 409
OrderNotOwnedError     extends ForbiddenError    → 403
InvalidOrderAmountError extends InvalidArgumentError → 422
InsufficientBalanceError extends DomainError      → 500 (or register in filter for 422)
```

To map a custom base like `InsufficientBalanceError` to a specific HTTP status, add it to the `HttpExceptionFilter`:

```typescript
// In the exception filter's error map
this.exceptionStatusMap.set(InsufficientBalanceError, HttpStatus.UNPROCESSABLE_ENTITY);
```

---

## Testing Domain Errors

```typescript
describe('OrderEntity', () => {
  it('should throw InvalidOrderStatusTransitionError on invalid transition', () => {
    const order = OrderEntity.restore({
      status: OrderStatusVO.create('DELIVERED'),
      // ...
    }, 'order-1');

    expect(() => order.cancel()).toThrow(InvalidOrderStatusTransitionError);
    expect(() => order.cancel()).toThrow('Cannot transition');
  });

  it('should throw InvalidOrderAmountError for negative amount', () => {
    expect(() =>
      OrderEntity.create({ amount: -100, organizationId: 'org-1' }),
    ).toThrow(InvalidOrderAmountError);
  });
});
```

---

## Anti-Patterns

| Don't | Do |
|---|---|
| `throw new Error('order not found')` | `throw new OrderNotFoundError(orderId)` |
| `throw new NotFoundError('order 123 not found')` when you have the ID | `throw new OrderNotFoundError('123')` |
| Create 20 error classes for a 3-entity BC | Create errors only for cases with domain context |
| Put HTTP status codes in domain errors | Let the error filter handle HTTP mapping |
| Import `HttpException` from NestJS in domain | Extend `DomainError` hierarchy only |
