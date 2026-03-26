# Validator Patterns

Domain validators enforce field-level invariants on entity props using `class-validator` decorators. They are used exclusively in the **domain layer** for entity validation — not for HTTP request validation.

## Separation of Concerns

```
Presentation layer (controllers/dtos/)
  → class-validator for HTTP input: format, presence, types
  → class-transformer for payload parsing

Domain layer (domain/validators/)
  → class-validator for DOMAIN invariants: business rules, cross-field validation
  → Thrown as EntityValidationError (not HttpException)

Value Objects (domain/value-objects/)
  → Manual validation in validate(): throw InvalidArgumentError
  → Never import class-validator
```

The validator pattern described here is for the **domain layer only**.

---

## ClassValidatorFields Base Class

Provided by `@/shared/base-classes/class-validator-fields`:

```typescript
// shared/base-classes/class-validator-fields.ts
import { ValidationError, validateSync } from 'class-validator';
import type { ValidatorFieldsInterface, FieldsErrors } from './validator-fields.interface';

export abstract class ClassValidatorFields<PropsValidated>
  implements ValidatorFieldsInterface<PropsValidated>
{
  errors: FieldsErrors = null;
  validatedData: PropsValidated = null;

  validate(data: unknown): boolean {
    const errors = validateSync(data as object);
    this.errors = errors.length ? {} : null;
    this.validatedData = errors.length ? null : (data as PropsValidated);
    errors.forEach((error) => this.extractErrors(error, error.property));
    return !errors.length;
  }

  private extractErrors(error: ValidationError, path: string): void {
    if (error.constraints) {
      this.errors![path] = Object.values(error.constraints);
    }
    if (error.children?.length) {
      error.children.forEach((child) =>
        this.extractErrors(child, `${path}.${child.property}`),
      );
    }
  }
}
```

---

## Full Validator Template

```typescript
// domain/validators/order.validator.ts
import {
  IsUUID,
  IsNotEmpty,
  IsString,
  IsNumber,
  IsPositive,
  IsOptional,
  IsDate,
  MaxLength,
} from 'class-validator';
import { ClassValidatorFields } from '@/shared/base-classes/class-validator-fields';
import type { OrderProps } from '../entities/order.entity';

/**
 * Rules class — mirrors OrderProps with class-validator decorators.
 * Constructor assigns props via Object.assign for compact construction.
 */
class OrderRules {
  @IsUUID()
  @IsNotEmpty()
  tenantId: string;

  @IsUUID()
  @IsNotEmpty()
  customerId: string;

  @IsNumber()
  @IsPositive()
  totalAmount: number;

  @IsOptional()
  @IsString()
  @MaxLength(1000)
  notes?: string;

  @IsDate()
  createdAt: Date;

  @IsOptional()
  @IsDate()
  updatedAt?: Date;

  constructor(props: OrderProps) {
    Object.assign(this, props);
  }
}

/**
 * Validator — wraps ClassValidatorFields with the concrete Rules class.
 * Override validate() to add cross-field or compound rules.
 */
class OrderValidator extends ClassValidatorFields<OrderRules> {
  validate(data: OrderProps): boolean {
    // Optional: handle null/undefined for VO fields before passing to base
    if (!data.tenantId) {
      this.errors = { tenantId: ['tenantId should not be empty'] };
      return false;
    }
    return super.validate(new OrderRules(data));
  }
}

/**
 * Factory — the entity calls this static method to get a fresh validator instance.
 */
export class OrderValidatorFactory {
  static create(): OrderValidator {
    return new OrderValidator();
  }
}
```

---

## Usage in Entity

```typescript
// domain/entities/order.entity.ts (relevant section)

static validate(props: OrderProps): void {
  const validator = OrderValidatorFactory.create();
  if (!validator.validate(props)) {
    throw new EntityValidationError(validator.errors);
  }
}

// Call in static create() before returning the entity
static create(input: CreateOrderInput, id?: string): OrderEntity {
  const props = { ...input, status: OrderStatusVO.pending(), createdAt: new Date() };
  OrderEntity.validate(props);
  const entity = new OrderEntity(props, id);
  entity.apply(new OrderPlacedEvent(entity.id, entity.tenantId));
  return entity;
}

// Optionally call in mutating methods when props change
update(notes: string): void {
  this.props.notes = notes;
  OrderEntity.validate(this.props); // re-validate after change
  this.touch();
}
```

---

## EntityValidationError

The error thrown when validation fails. Provided by the shared domain-errors package:

```typescript
// shared/domain-errors/validation-error.ts
export type FieldsErrors = Record<string, string[]> | null;

export class EntityValidationError extends Error {
  constructor(public readonly errors: FieldsErrors) {
    super('Entity validation error');
    this.name = 'EntityValidationError';
  }
}
```

---

## Validator with Custom Cross-Field Rules

When `class-validator` decorators are not sufficient, add custom logic in the `validate()` override:

```typescript
// domain/validators/withdrawal-request.validator.ts
import { IsUUID, IsNotEmpty, IsNumber, IsPositive, IsOptional } from 'class-validator';
import { ClassValidatorFields } from '@/shared/base-classes/class-validator-fields';
import type { WithdrawalRequestProps } from '../entities/withdrawal-request.entity';

class WithdrawalRequestRules {
  @IsUUID() @IsNotEmpty() companyId: string;
  @IsNumber() @IsPositive() requestedAmount: number;
  @IsOptional() @IsNumber() withdrawalFeePercentage?: number;

  constructor(props: WithdrawalRequestProps) {
    Object.assign(this, props);
  }
}

class WithdrawalRequestValidator extends ClassValidatorFields<WithdrawalRequestRules> {
  validate(data: WithdrawalRequestProps): boolean {
    // 1. Run class-validator decorators first
    const isValid = super.validate(new WithdrawalRequestRules(data));
    if (!isValid) return false;

    // 2. Cross-field rule: fee must not exceed amount
    if (
      data.withdrawalFeeTotal !== undefined &&
      data.withdrawalFeeTotal > data.requestedAmount
    ) {
      this.errors = {
        withdrawalFeeTotal: ['Fee total cannot exceed the requested amount'],
      };
      return false;
    }

    return true;
  }
}

export class WithdrawalRequestValidatorFactory {
  static create(): WithdrawalRequestValidator {
    return new WithdrawalRequestValidator();
  }
}
```

---

## Test Template

```typescript
// domain/validators/__tests__/order.validator.spec.ts
import { describe, it, expect } from 'vitest';
import { faker } from '@faker-js/faker';
import { OrderValidatorFactory } from '../order.validator';
import { OrderDataBuilder } from '../../testing/helpers/order.data-builder';
import { OrderStatusVO } from '../../value-objects/order-status.vo';

describe('OrderValidator', () => {
  const makeValidator = () => OrderValidatorFactory.create();

  it('should validate valid props', () => {
    const props = {
      ...OrderDataBuilder(),
      status: OrderStatusVO.pending(),
      createdAt: new Date(),
    };
    const validator = makeValidator();
    expect(validator.validate(props)).toBe(true);
    expect(validator.errors).toBeNull();
  });

  it('should reject empty tenantId', () => {
    const validator = makeValidator();
    const isValid = validator.validate({
      ...OrderDataBuilder(),
      tenantId: '',
      status: OrderStatusVO.pending(),
      createdAt: new Date(),
    });
    expect(isValid).toBe(false);
    expect(validator.errors).toHaveProperty('tenantId');
  });

  it('should reject invalid UUID for customerId', () => {
    const validator = makeValidator();
    const isValid = validator.validate({
      ...OrderDataBuilder(),
      customerId: 'not-a-uuid',
      status: OrderStatusVO.pending(),
      createdAt: new Date(),
    });
    expect(isValid).toBe(false);
    expect(validator.errors).toHaveProperty('customerId');
  });

  it('should reject negative totalAmount', () => {
    const validator = makeValidator();
    const isValid = validator.validate({
      ...OrderDataBuilder(),
      totalAmount: -100,
      status: OrderStatusVO.pending(),
      createdAt: new Date(),
    });
    expect(isValid).toBe(false);
    expect(validator.errors).toHaveProperty('totalAmount');
  });

  it('should allow undefined optional notes', () => {
    const validator = makeValidator();
    const isValid = validator.validate({
      ...OrderDataBuilder(),
      notes: undefined,
      status: OrderStatusVO.pending(),
      createdAt: new Date(),
    });
    expect(isValid).toBe(true);
  });

  it('should reject notes exceeding max length', () => {
    const validator = makeValidator();
    const isValid = validator.validate({
      ...OrderDataBuilder(),
      notes: 'x'.repeat(1001),
      status: OrderStatusVO.pending(),
      createdAt: new Date(),
    });
    expect(isValid).toBe(false);
    expect(validator.errors).toHaveProperty('notes');
  });
});
```

---

## Common class-validator Decorators for Domain Validators

| Decorator | Use case |
|---|---|
| `@IsUUID()` | Entity IDs, foreign keys |
| `@IsNotEmpty()` | Required strings |
| `@IsString()` | String fields |
| `@MaxLength(n)` | Length-bounded strings |
| `@IsNumber()` | Numeric fields |
| `@IsPositive()` | Positive numbers (amount, quantity) |
| `@IsInt()` | Integer amounts (cents) |
| `@IsOptional()` | Nullable or optional fields |
| `@IsDate()` | Date objects |
| `@IsEmail()` | Email strings (prefer EmailVO instead) |
| `@IsIn(values)` | Fixed-set string fields |
| `@IsEnum(Enum)` | TypeScript enum fields |
| `@ValidateNested()` | Nested object validation |
| `@Type(() => NestedClass)` | Required with ValidateNested |

**Important:** Do NOT use `@IsEmail()` or format validators on fields that are represented by VOs. The VO handles its own validation. Pass the `.value` (primitive) to the Rules class if you need to validate it here, or skip the decorator and rely on the VO constructor.
