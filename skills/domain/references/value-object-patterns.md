# Value Object Patterns

Value Objects (VOs) are immutable, self-validating objects identified by their **value**, not by identity. Two VOs with the same value are equal. They capture domain concepts that primitive types cannot express safely.

## Base Class

All VOs extend `ValueObject<T>` from `@/shared/base-classes/value-object`. The constructor calls `this.validate()` automatically, so invalid VOs can never be constructed.

```typescript
export abstract class ValueObject<T> {
  protected readonly _value: T;

  constructor(value: T) {
    this._value = value;
    this.validate();   // called on every construction — invariant enforced
  }

  protected abstract validate(): void;

  get value(): T { return this._value; }

  equals(vo?: ValueObject<T>): boolean {
    if (!vo) return false;
    if (vo.constructor !== this.constructor) return false;
    return JSON.stringify(vo.value) === JSON.stringify(this.value);
  }

  toString(): string { return String(this._value); }
  toJSON(): T { return this._value; }
}
```

---

## Pattern 1: Scalar VO

Use for a single primitive value with domain-specific invariants.

**Examples:** `EmailVO`, `PhoneVO`, `AmountVO`, `PercentageVO`, `UrlVO`

```typescript
// domain/value-objects/email.vo.ts
import { ValueObject } from '@/shared/base-classes/value-object';
import { InvalidArgumentError } from '@/shared/domain-errors/errors';

export class EmailVO extends ValueObject<string> {
  private constructor(value: string) {
    super(value);
  }

  protected validate(): void {
    const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!EMAIL_REGEX.test(this._value)) {
      throw new InvalidArgumentError(`Invalid email address: "${this._value}"`);
    }
    if (this._value.length > 254) {
      throw new InvalidArgumentError('Email address exceeds maximum length of 254 characters');
    }
  }

  static create(email: string): EmailVO {
    return new EmailVO(email.toLowerCase().trim());
  }

  get domain(): string   { return this._value.split('@')[1]; }
  get username(): string { return this._value.split('@')[0]; }
}
```

**Amount VO (numeric with constraints):**

```typescript
// domain/value-objects/amount.vo.ts
import { ValueObject } from '@/shared/base-classes/value-object';
import { InvalidArgumentError } from '@/shared/domain-errors/errors';

export class AmountVO extends ValueObject<number> {
  private constructor(value: number) { super(value); }

  protected validate(): void {
    if (!Number.isFinite(this._value) || this._value < 0) {
      throw new InvalidArgumentError(`Amount must be a non-negative finite number, got: ${this._value}`);
    }
    if (!Number.isInteger(this._value)) {
      throw new InvalidArgumentError('Amount must be an integer (in cents)');
    }
  }

  static create(cents: number): AmountVO { return new AmountVO(cents); }
  static zero(): AmountVO { return new AmountVO(0); }

  add(other: AmountVO): AmountVO     { return new AmountVO(this._value + other.value); }
  subtract(other: AmountVO): AmountVO {
    if (other.value > this._value) throw new InvalidArgumentError('Cannot subtract: result would be negative');
    return new AmountVO(this._value - other.value);
  }

  toCurrency(divisor = 100): number { return this._value / divisor; }
}
```

### Test Template — Scalar VO

```typescript
// domain/value-objects/__tests__/email.vo.spec.ts
import { describe, it, expect } from 'vitest';
import { EmailVO } from '../email.vo';

describe('EmailVO', () => {
  describe('create()', () => {
    it('should normalise to lowercase', () => {
      expect(EmailVO.create('USER@EXAMPLE.COM').value).toBe('user@example.com');
    });

    it('should trim whitespace', () => {
      expect(EmailVO.create('  user@example.com  ').value).toBe('user@example.com');
    });

    it('should throw on missing @', () => {
      expect(() => EmailVO.create('notanemail')).toThrow();
    });

    it('should throw on empty string', () => {
      expect(() => EmailVO.create('')).toThrow();
    });

    it('should throw when exceeding 254 chars', () => {
      expect(() => EmailVO.create('a'.repeat(250) + '@b.com')).toThrow();
    });
  });

  describe('equals()', () => {
    it('should be equal when value matches', () => {
      const a = EmailVO.create('a@b.com');
      const b = EmailVO.create('a@b.com');
      expect(a.equals(b)).toBe(true);
    });

    it('should not be equal when value differs', () => {
      expect(EmailVO.create('a@b.com').equals(EmailVO.create('x@b.com'))).toBe(false);
    });
  });

  describe('domain / username getters', () => {
    it('should expose domain and username parts', () => {
      const email = EmailVO.create('alice@example.com');
      expect(email.domain).toBe('example.com');
      expect(email.username).toBe('alice');
    });
  });
});
```

---

## Pattern 2: Composed VO

Use when a domain concept has multiple related properties that must be validated together.

**Examples:** `AddressVO`, `BankDetailsVO`, `MoneyVO`, `DateRangeVO`

```typescript
// domain/value-objects/bank-details.vo.ts
import { ValueObject } from '@/shared/base-classes/value-object';
import { InvalidArgumentError } from '@/shared/domain-errors/errors';

export interface BankDetailsProps {
  bankName?: string;
  bankBranch?: string;
  bankAccount?: string;
  accountHolder?: string;
  pixKey?: string;
}

export class BankDetailsVO extends ValueObject<BankDetailsProps> {
  private constructor(props: BankDetailsProps) { super(props); }

  protected validate(): void {
    const hasPixKey = !!this._value.pixKey;
    const hasBankAccount =
      !!this._value.bankName &&
      !!this._value.bankAccount &&
      !!this._value.accountHolder;

    if (!hasPixKey && !hasBankAccount) {
      throw new InvalidArgumentError(
        'Bank details must include either a PIX key or complete bank account info (bank name, account, holder)',
      );
    }
  }

  static create(props: BankDetailsProps): BankDetailsVO {
    return new BankDetailsVO(props);
  }

  static withPixKey(pixKey: string): BankDetailsVO {
    return new BankDetailsVO({ pixKey });
  }

  static withBankAccount(props: Required<Pick<BankDetailsProps, 'bankName' | 'bankAccount' | 'accountHolder'>> & Pick<BankDetailsProps, 'bankBranch'>): BankDetailsVO {
    return new BankDetailsVO(props);
  }

  get pixKey(): string | undefined        { return this._value.pixKey; }
  get bankName(): string | undefined      { return this._value.bankName; }
  get bankAccount(): string | undefined   { return this._value.bankAccount; }
  get accountHolder(): string | undefined { return this._value.accountHolder; }

  hasPixKey(): boolean      { return !!this.pixKey; }
  hasBankAccount(): boolean { return !!this.bankName && !!this.bankAccount && !!this.accountHolder; }

  toJSON(): BankDetailsProps {
    return {
      bankName: this.bankName,
      bankBranch: this._value.bankBranch,
      bankAccount: this.bankAccount,
      accountHolder: this.accountHolder,
      pixKey: this.pixKey,
    };
  }
}
```

### Test Template — Composed VO

```typescript
// domain/value-objects/__tests__/bank-details.vo.spec.ts
import { describe, it, expect } from 'vitest';
import { BankDetailsVO } from '../bank-details.vo';

describe('BankDetailsVO', () => {
  describe('create()', () => {
    it('should create with a PIX key', () => {
      const vo = BankDetailsVO.withPixKey('pix@example.com');
      expect(vo.hasPixKey()).toBe(true);
      expect(vo.hasBankAccount()).toBe(false);
    });

    it('should create with full bank account details', () => {
      const vo = BankDetailsVO.withBankAccount({
        bankName: 'Nubank', bankAccount: '12345-6', accountHolder: 'Alice',
      });
      expect(vo.hasBankAccount()).toBe(true);
    });

    it('should throw when neither PIX key nor bank details are provided', () => {
      expect(() => BankDetailsVO.create({})).toThrow();
    });

    it('should throw with partial bank details (missing accountHolder)', () => {
      expect(() =>
        BankDetailsVO.create({ bankName: 'Nubank', bankAccount: '12345-6' }),
      ).toThrow();
    });
  });

  describe('equals()', () => {
    it('should be equal when all props match', () => {
      const a = BankDetailsVO.withPixKey('key');
      const b = BankDetailsVO.withPixKey('key');
      expect(a.equals(b)).toBe(true);
    });
  });
});
```

---

## Pattern 3: Enum VO

Use when a domain concept has a fixed set of named values. Provides type safety beyond TypeScript enums by adding domain-specific predicates.

**Examples:** `PaymentMethodVO`, `ChannelTypeVO`, `UserRoleVO`, `CurrencyVO`

```typescript
// domain/value-objects/user-role.vo.ts
import { ValueObject } from '@/shared/base-classes/value-object';
import { InvalidArgumentError } from '@/shared/domain-errors/errors';

export enum UserRoleEnum {
  ADMIN = 'ADMIN',
  MANAGER = 'MANAGER',
  AGENT = 'AGENT',
  VIEWER = 'VIEWER',
}

export class UserRoleVO extends ValueObject<UserRoleEnum> {
  private constructor(value: UserRoleEnum) { super(value); }

  protected validate(): void {
    if (!Object.values(UserRoleEnum).includes(this._value)) {
      throw new InvalidArgumentError(`Invalid user role: "${this._value}"`);
    }
  }

  // Named factories — callers never pass raw strings
  static admin():   UserRoleVO { return new UserRoleVO(UserRoleEnum.ADMIN); }
  static manager(): UserRoleVO { return new UserRoleVO(UserRoleEnum.MANAGER); }
  static agent():   UserRoleVO { return new UserRoleVO(UserRoleEnum.AGENT); }
  static viewer():  UserRoleVO { return new UserRoleVO(UserRoleEnum.VIEWER); }

  // For deserialization from DB / external input
  static from(value: string): UserRoleVO {
    return new UserRoleVO(value as UserRoleEnum);
  }

  isAdmin():   boolean { return this._value === UserRoleEnum.ADMIN; }
  isManager(): boolean { return this._value === UserRoleEnum.MANAGER; }
  isAgent():   boolean { return this._value === UserRoleEnum.AGENT; }
  isViewer():  boolean { return this._value === UserRoleEnum.VIEWER; }

  canManageUsers(): boolean    { return this.isAdmin(); }
  canManageTeams(): boolean    { return this.isAdmin() || this.isManager(); }
  canAssignChats(): boolean    { return !this.isViewer(); }
}
```

### Test Template — Enum VO

```typescript
// domain/value-objects/__tests__/user-role.vo.spec.ts
import { describe, it, expect } from 'vitest';
import { UserRoleVO } from '../user-role.vo';

describe('UserRoleVO', () => {
  describe('named factories', () => {
    it('should create each role correctly', () => {
      expect(UserRoleVO.admin().isAdmin()).toBe(true);
      expect(UserRoleVO.agent().isAgent()).toBe(true);
      expect(UserRoleVO.viewer().isViewer()).toBe(true);
    });
  });

  describe('from()', () => {
    it('should create from valid string', () => {
      expect(UserRoleVO.from('ADMIN').isAdmin()).toBe(true);
    });

    it('should throw on invalid string', () => {
      expect(() => UserRoleVO.from('SUPERUSER')).toThrow();
    });
  });

  describe('permission predicates', () => {
    it('only ADMIN can manage users', () => {
      expect(UserRoleVO.admin().canManageUsers()).toBe(true);
      expect(UserRoleVO.manager().canManageUsers()).toBe(false);
    });

    it('VIEWER cannot assign chats', () => {
      expect(UserRoleVO.viewer().canAssignChats()).toBe(false);
      expect(UserRoleVO.agent().canAssignChats()).toBe(true);
    });
  });
});
```

---

## Pattern 4: State Machine VO

Use when a domain concept follows a lifecycle with explicit allowed transitions. The VO enforces transition legality at construction time.

**Examples:** `OrderStatusVO`, `WithdrawalStatusVO`, `ConversationStatusVO`, `PaymentStatusVO`

```typescript
// domain/value-objects/order-status.vo.ts
import { ValueObject } from '@/shared/base-classes/value-object';
import { InvalidArgumentError } from '@/shared/domain-errors/errors';

export enum OrderStatusEnum {
  PENDING    = 'PENDING',
  CONFIRMED  = 'CONFIRMED',
  PROCESSING = 'PROCESSING',
  SHIPPED    = 'SHIPPED',
  COMPLETED  = 'COMPLETED',
  CANCELLED  = 'CANCELLED',
}

const ALLOWED_TRANSITIONS: Record<OrderStatusEnum, OrderStatusEnum[]> = {
  [OrderStatusEnum.PENDING]:    [OrderStatusEnum.CONFIRMED, OrderStatusEnum.CANCELLED],
  [OrderStatusEnum.CONFIRMED]:  [OrderStatusEnum.PROCESSING, OrderStatusEnum.CANCELLED],
  [OrderStatusEnum.PROCESSING]: [OrderStatusEnum.SHIPPED, OrderStatusEnum.CANCELLED],
  [OrderStatusEnum.SHIPPED]:    [OrderStatusEnum.COMPLETED],
  [OrderStatusEnum.COMPLETED]:  [],
  [OrderStatusEnum.CANCELLED]:  [],
};

export class OrderStatusVO extends ValueObject<OrderStatusEnum> {
  private constructor(value: OrderStatusEnum) { super(value); }

  protected validate(): void {
    if (!Object.values(OrderStatusEnum).includes(this._value)) {
      throw new InvalidArgumentError(`Invalid order status: "${this._value}"`);
    }
  }

  // Named factories
  static pending():    OrderStatusVO { return new OrderStatusVO(OrderStatusEnum.PENDING); }
  static confirmed():  OrderStatusVO { return new OrderStatusVO(OrderStatusEnum.CONFIRMED); }
  static processing(): OrderStatusVO { return new OrderStatusVO(OrderStatusEnum.PROCESSING); }
  static shipped():    OrderStatusVO { return new OrderStatusVO(OrderStatusEnum.SHIPPED); }
  static completed():  OrderStatusVO { return new OrderStatusVO(OrderStatusEnum.COMPLETED); }
  static cancelled():  OrderStatusVO { return new OrderStatusVO(OrderStatusEnum.CANCELLED); }

  // Deserialization
  static from(value: string): OrderStatusVO {
    return new OrderStatusVO(value as OrderStatusEnum);
  }

  // Transition guard
  canTransitionTo(target: OrderStatusEnum): boolean {
    return ALLOWED_TRANSITIONS[this._value].includes(target);
  }

  // Immutable transition — returns NEW VO
  transitionTo(target: OrderStatusEnum): OrderStatusVO {
    if (!this.canTransitionTo(target)) {
      throw new InvalidArgumentError(
        `Cannot transition order from "${this._value}" to "${target}"`,
      );
    }
    return new OrderStatusVO(target);
  }

  // Convenience guards used by entity methods
  canConfirm():  boolean { return this.canTransitionTo(OrderStatusEnum.CONFIRMED); }
  canCancel():   boolean { return this.canTransitionTo(OrderStatusEnum.CANCELLED); }
  canShip():     boolean { return this.canTransitionTo(OrderStatusEnum.SHIPPED); }

  // State predicates
  isPending():    boolean { return this._value === OrderStatusEnum.PENDING; }
  isConfirmed():  boolean { return this._value === OrderStatusEnum.CONFIRMED; }
  isCompleted():  boolean { return this._value === OrderStatusEnum.COMPLETED; }
  isCancelled():  boolean { return this._value === OrderStatusEnum.CANCELLED; }
  isTerminal():   boolean { return this.isCompleted() || this.isCancelled(); }
}
```

### Test Template — State Machine VO

```typescript
// domain/value-objects/__tests__/order-status.vo.spec.ts
import { describe, it, expect } from 'vitest';
import { OrderStatusVO, OrderStatusEnum } from '../order-status.vo';

describe('OrderStatusVO', () => {
  describe('named factories', () => {
    it('should create each status', () => {
      expect(OrderStatusVO.pending().isPending()).toBe(true);
      expect(OrderStatusVO.confirmed().isConfirmed()).toBe(true);
      expect(OrderStatusVO.cancelled().isCancelled()).toBe(true);
    });
  });

  describe('canTransitionTo()', () => {
    it('PENDING can go to CONFIRMED', () => {
      expect(OrderStatusVO.pending().canTransitionTo(OrderStatusEnum.CONFIRMED)).toBe(true);
    });

    it('PENDING can go to CANCELLED', () => {
      expect(OrderStatusVO.pending().canTransitionTo(OrderStatusEnum.CANCELLED)).toBe(true);
    });

    it('COMPLETED cannot transition to anything', () => {
      const completed = OrderStatusVO.completed();
      Object.values(OrderStatusEnum).forEach((target) => {
        expect(completed.canTransitionTo(target)).toBe(false);
      });
    });
  });

  describe('transitionTo()', () => {
    it('should return a new VO on valid transition', () => {
      const pending = OrderStatusVO.pending();
      const confirmed = pending.transitionTo(OrderStatusEnum.CONFIRMED);

      expect(confirmed.isConfirmed()).toBe(true);
      expect(pending.isPending()).toBe(true); // original unchanged
    });

    it('should throw on invalid transition', () => {
      expect(() =>
        OrderStatusVO.completed().transitionTo(OrderStatusEnum.PENDING),
      ).toThrow();
    });
  });

  describe('isTerminal()', () => {
    it('should be true for COMPLETED and CANCELLED', () => {
      expect(OrderStatusVO.completed().isTerminal()).toBe(true);
      expect(OrderStatusVO.cancelled().isTerminal()).toBe(true);
    });

    it('should be false for non-terminal statuses', () => {
      expect(OrderStatusVO.pending().isTerminal()).toBe(false);
      expect(OrderStatusVO.confirmed().isTerminal()).toBe(false);
    });
  });

  describe('from()', () => {
    it('should deserialise from valid string', () => {
      expect(OrderStatusVO.from('PENDING').isPending()).toBe(true);
    });

    it('should throw for unknown value', () => {
      expect(() => OrderStatusVO.from('UNKNOWN')).toThrow();
    });
  });
});
```
