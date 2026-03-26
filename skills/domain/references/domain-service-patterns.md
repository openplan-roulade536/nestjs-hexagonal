# Domain Service Patterns

A domain service contains domain logic that does not naturally belong to a single entity or value object. Domain services are pure classes — no decorators, no dependency injection, no I/O.

## When to Use a Domain Service

| Situation | Solution |
|---|---|
| Logic clearly belongs to one entity | Put it in the entity method |
| Logic involves multiple aggregates | Domain service |
| A calculation or rule is shared by 2+ entity methods | Domain service |
| Logic requires I/O (DB, queue, HTTP) | Application service in `application/services/` |
| Logic is stateless and has no side effects | Static methods on a domain service |

---

## Pattern 1: Stateless Service (static methods)

Use for pure calculations, validations across multiple values, or permission checks that depend only on entity state.

```typescript
// domain/services/fee-calculation.service.ts
import { InvalidArgumentError } from '@/shared/domain-errors/errors';

export class FeeCalculationService {
  private static readonly MAX_FEE_PERCENTAGE = 50;

  /**
   * Calculates the fee amount in cents from a base amount and percentage.
   * Returns integers to avoid floating-point precision errors.
   */
  static calculate(amountCents: number, feePercentage: number): number {
    if (amountCents <= 0) {
      throw new InvalidArgumentError('Amount must be positive');
    }
    if (feePercentage < 0 || feePercentage > FeeCalculationService.MAX_FEE_PERCENTAGE) {
      throw new InvalidArgumentError(
        `Fee percentage must be between 0 and ${FeeCalculationService.MAX_FEE_PERCENTAGE}`,
      );
    }
    return Math.round(amountCents * (feePercentage / 100));
  }

  /**
   * Calculates the net amount after deducting a percentage fee and a fixed fee.
   */
  static calculateNet(
    amountCents: number,
    feePercentage: number,
    fixedFeeCents: number,
  ): { feeCents: number; netCents: number } {
    const percentageFee = FeeCalculationService.calculate(amountCents, feePercentage);
    const totalFee = percentageFee + fixedFeeCents;
    const net = amountCents - totalFee;

    if (net < 0) {
      throw new InvalidArgumentError('Fees exceed the total amount');
    }

    return { feeCents: totalFee, netCents: net };
  }
}
```

---

## Pattern 2: Permission / Policy Service

Validates whether an operation is allowed based on the current state of one or more entities. Throws a domain error when the operation is not allowed.

```typescript
// domain/services/attachment-permission.service.ts
import { ConflictError } from '@/shared/domain-errors/errors';
import { WithdrawalRequestEntity } from '../entities/withdrawal-request.entity';

export class AttachmentPermissionService {
  static canModifyAttachments(request: WithdrawalRequestEntity): boolean {
    return request.status.isPending() || request.status.isApproved();
  }

  static assertCanUpload(request: WithdrawalRequestEntity): void {
    if (!AttachmentPermissionService.canModifyAttachments(request)) {
      throw new ConflictError(
        `Cannot upload attachments for a withdrawal request with status "${request.status.value}". ` +
        `Only PENDING and APPROVED requests accept attachments.`,
      );
    }
  }

  static assertCanDelete(request: WithdrawalRequestEntity): void {
    if (!AttachmentPermissionService.canModifyAttachments(request)) {
      throw new ConflictError(
        `Cannot delete attachments from a withdrawal request with status "${request.status.value}".`,
      );
    }
  }
}
```

---

## Pattern 3: Cross-Aggregate Rule Service

When a business rule spans two or more aggregates, neither can own it — the domain service enforces it.

```typescript
// domain/services/order-transfer.service.ts
import { InvalidArgumentError } from '@/shared/domain-errors/errors';
import { OrderEntity } from '../entities/order.entity';
import { CustomerEntity } from '../entities/customer.entity';

export class OrderTransferService {
  /**
   * Validates that an order can be transferred to the target customer.
   * Rules:
   *   - Order must be in PENDING or CONFIRMED status
   *   - Target customer must belong to the same tenant
   *   - Customer must be active
   */
  static assertCanTransfer(
    order: OrderEntity,
    targetCustomer: CustomerEntity,
  ): void {
    if (!order.status.isPending() && !order.status.isConfirmed()) {
      throw new InvalidArgumentError(
        `Order ${order.id} cannot be transferred from status "${order.status.value}"`,
      );
    }

    if (order.tenantId !== targetCustomer.tenantId) {
      throw new InvalidArgumentError('Cannot transfer order across tenants');
    }

    if (!targetCustomer.isActive) {
      throw new InvalidArgumentError(
        `Customer ${targetCustomer.id} is not active and cannot receive orders`,
      );
    }
  }
}
```

---

## Pattern 4: Stateful Domain Service (instance)

When the service needs to accumulate state during a computation:

```typescript
// domain/services/order-totals.service.ts
import { OrderItemEntity } from '../entities/order-item.entity';
import { InvalidArgumentError } from '@/shared/domain-errors/errors';

export class OrderTotalsService {
  private items: OrderItemEntity[] = [];

  add(item: OrderItemEntity): this {
    this.items.push(item);
    return this;
  }

  calculateSubtotal(): number {
    return this.items.reduce((sum, item) => sum + item.subtotal, 0);
  }

  calculateTax(taxRate: number): number {
    if (taxRate < 0 || taxRate > 1) {
      throw new InvalidArgumentError('Tax rate must be between 0 and 1');
    }
    return Math.round(this.calculateSubtotal() * taxRate);
  }

  calculateTotal(taxRate: number): number {
    return this.calculateSubtotal() + this.calculateTax(taxRate);
  }
}
```

Instantiate directly where needed — no DI required:

```typescript
// In the entity or command handler:
const totals = new OrderTotalsService();
items.forEach((item) => totals.add(item));
const total = totals.calculateTotal(0.1);
```

---

## Instantiation

Domain services are instantiated directly — no DI container:

```typescript
// Stateless — call static methods directly
const fee = FeeCalculationService.calculate(10000, 2.5);

// Stateful — construct and use
const totals = new OrderTotalsService();

// In an entity method
confirm(items: OrderItemEntity[]): void {
  OrderTransferService.assertCanTransfer(this, targetCustomer);
  // ...
}
```

If a domain service needs to be injected for testability (rare), create it as an application service in `application/services/` with `@Injectable()`.

---

## Test Template

Domain services require no mocks — pure unit tests:

```typescript
// domain/services/__tests__/fee-calculation.service.spec.ts
import { describe, it, expect } from 'vitest';
import { FeeCalculationService } from '../fee-calculation.service';

describe('FeeCalculationService', () => {
  describe('calculate()', () => {
    it('should calculate 10% of 10000 cents = 1000', () => {
      expect(FeeCalculationService.calculate(10000, 10)).toBe(1000);
    });

    it('should round fractional cents', () => {
      // 10000 * 1.5% = 150.0 — exact
      expect(FeeCalculationService.calculate(10000, 1.5)).toBe(150);
      // 10001 * 1.5% = 150.015 → rounds to 150
      expect(FeeCalculationService.calculate(10001, 1.5)).toBe(150);
    });

    it('should throw for non-positive amount', () => {
      expect(() => FeeCalculationService.calculate(0, 10)).toThrow();
      expect(() => FeeCalculationService.calculate(-100, 10)).toThrow();
    });

    it('should throw for negative fee percentage', () => {
      expect(() => FeeCalculationService.calculate(1000, -1)).toThrow();
    });

    it('should throw when fee percentage exceeds 50', () => {
      expect(() => FeeCalculationService.calculate(1000, 51)).toThrow();
    });
  });

  describe('calculateNet()', () => {
    it('should return correct fee and net amounts', () => {
      const result = FeeCalculationService.calculateNet(10000, 2, 100);
      expect(result.feeCents).toBe(300); // 2% of 10000 = 200, + fixed 100 = 300
      expect(result.netCents).toBe(9700);
    });

    it('should throw when fees exceed amount', () => {
      expect(() => FeeCalculationService.calculateNet(100, 50, 200)).toThrow();
    });
  });
});
```

```typescript
// domain/services/__tests__/attachment-permission.service.spec.ts
import { describe, it, expect, vi } from 'vitest';
import { AttachmentPermissionService } from '../attachment-permission.service';
import { WithdrawalStatusVO } from '../../value-objects/withdrawal-status.vo';

describe('AttachmentPermissionService', () => {
  const makeRequest = (status: WithdrawalStatusVO) =>
    ({ status } as unknown as import('../../entities/withdrawal-request.entity').WithdrawalRequestEntity);

  it('should allow upload when PENDING', () => {
    expect(() =>
      AttachmentPermissionService.assertCanUpload(
        makeRequest(WithdrawalStatusVO.createPending()),
      ),
    ).not.toThrow();
  });

  it('should allow upload when APPROVED', () => {
    expect(() =>
      AttachmentPermissionService.assertCanUpload(
        makeRequest(WithdrawalStatusVO.createApproved()),
      ),
    ).not.toThrow();
  });

  it('should throw when COMPLETED', () => {
    expect(() =>
      AttachmentPermissionService.assertCanUpload(
        makeRequest(WithdrawalStatusVO.createCompleted()),
      ),
    ).toThrow();
  });
});
```
