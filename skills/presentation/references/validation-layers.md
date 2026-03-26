# Validation Layers — 4-Layer Architecture

Each layer has a single, well-defined validation responsibility. Never duplicate validation across layers.

## Diagram

```
┌───────────────────────────────────────────────────────────────────┐
│  LAYER 1 — Request DTO (presentation)                             │
│  Location: infrastructure/controllers/dtos/*.request.dto.ts       │
│  Tool: class-validator + class-transformer + Swagger              │
│  Responsibility: HTTP input format, presence, basic types         │
│  Triggered: Automatically by global ValidationPipe                │
├───────────────────────────────────────────────────────────────────┤
│  LAYER 2 — Application DTO (application)                          │
│  Location: application/dtos/*.dto.ts or inline in Command class   │
│  Tool: TypeScript interfaces and type system                      │
│  Responsibility: Contract between presentation and application     │
│  Triggered: TypeScript compiler (no runtime validation)           │
├───────────────────────────────────────────────────────────────────┤
│  LAYER 3 — Domain Value Objects (domain)                          │
│  Location: domain/value-objects/*.vo.ts                           │
│  Tool: Manual validate() / throw DomainError in constructor       │
│  Responsibility: Business invariants (always valid state)         │
│  Triggered: Entity.create() calls VO constructors                 │
├───────────────────────────────────────────────────────────────────┤
│  LAYER 4 — Queue / Integration Schema (integration)               │
│  Location: shared/schemas/ or inline in consumer handleMessage    │
│  Tool: Zod schemas                                                │
│  Responsibility: Inter-service message contracts                  │
│  Triggered: Consumer validates incoming queue messages            │
└───────────────────────────────────────────────────────────────────┘
```

## Layer 1 — Request DTO

```typescript
// infrastructure/controllers/dtos/create-customer.request.dto.ts
import { IsString, IsNotEmpty, IsEmail } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateCustomerRequestDto {
  @ApiProperty({ example: 'João Silva' })
  @IsString()
  @IsNotEmpty()
  name!: string;

  @ApiProperty({ example: 'joao@example.com' })
  @IsEmail()
  @IsNotEmpty()
  email!: string;

  // class-validator validates: format, presence, basic types
  // Does NOT validate: email domain exists, customer uniqueness, business rules
}
```

Triggered automatically by:

```typescript
app.useGlobalPipes(
  new ValidationPipe({ transform: true, whitelist: true, forbidNonWhitelisted: true }),
);
```

## Layer 2 — Application DTO

```typescript
// application/dtos/create-customer.dto.ts
export namespace CreateCustomerDto {
  export interface Input {
    organizationId: string;
    name: string;
    email: string;
  }

  export interface Output {
    id: string;
    name: string;
    email: string;
    createdAt: Date;
  }
}

// Or inline in a Command class:
// application/commands/create-customer.command.ts
export class CreateCustomerCommand extends Command<{ id: string }> {
  constructor(
    public readonly organizationId: string,  // from auth context
    public readonly name: string,            // from DTO
    public readonly email: string,           // from DTO
  ) {
    super();
  }
}
```

No decorators, no runtime validation. TypeScript enforces the contract at compile time.

## Layer 3 — Domain Value Objects

```typescript
// domain/value-objects/email.vo.ts
import { DomainError } from '@/shared/domain/errors/domain.error';

export class EmailVO {
  private readonly value: string;

  private constructor(value: string) {
    this.value = value;
  }

  static create(raw: string): EmailVO {
    const normalized = raw.trim().toLowerCase();
    if (!normalized.includes('@') || !normalized.includes('.')) {
      throw new DomainError(`Invalid email: ${raw}`);
    }
    return new EmailVO(normalized);
  }

  toString(): string {
    return this.value;
  }

  equals(other: EmailVO): boolean {
    return this.value === other.value;
  }
}

// NEVER import class-validator or @nestjs/* in a VO
// NEVER use @IsEmail() or similar decorators in a VO
```

Triggered when `CustomerEntity.create()` constructs `new EmailVO(...)`.

## Layer 4 — Queue / Integration Schema

```typescript
// shared/schemas/customer-created.schema.ts
import { z } from 'zod';

export const CustomerCreatedSchema = z.object({
  eventType: z.literal('customer.created'),
  organizationId: z.string().uuid(),
  customerId: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  occurredAt: z.string().datetime(),
});

export type CustomerCreatedMessage = z.infer<typeof CustomerCreatedSchema>;

// In consumer:
const parsed = CustomerCreatedSchema.safeParse(rawMessage);
if (!parsed.success) {
  this.logger.error('Invalid message schema', parsed.error.issues);
  return; // or send to DLQ
}
```

## Anti-Patterns — What NOT to Do

### class-validator in a Value Object

```typescript
// BAD — class-validator belongs in Layer 1 only
import { IsEmail } from 'class-validator'; // PROHIBITED in domain layer

export class EmailVO {
  @IsEmail() // NEVER do this
  value: string;
}
```

### Zod in a Request DTO

```typescript
// BAD — Zod belongs in Layer 4 (queue) or Layer 2 (optional)
export class CreateCustomerRequestDto {
  name = z.string().parse(rawName); // PROHIBITED in presentation layer
}
```

### Business logic in a Request DTO

```typescript
// BAD — DTO validates format, not business rules
export class CreateCustomerRequestDto {
  @IsEmail()
  email!: string;

  // PROHIBITED — uniqueness is a business rule, belongs in Layer 3
  async validate() {
    const exists = await db.customer.findFirst({ where: { email: this.email } });
    if (exists) throw new Error('Email already taken');
  }
}
```

### Duplicate validation across layers

```typescript
// BAD — name presence is already validated in Layer 1 (DTO)
// Don't repeat it in the use case
export class CreateCustomerUseCase {
  async execute(input: CreateCustomerDto.Input) {
    if (!input.name) throw new Error('Name required'); // REDUNDANT — Layer 1 already caught this
  }
}

// GOOD — use case validates business rules (uniqueness), not format
export class CreateCustomerUseCase {
  async execute(input: CreateCustomerDto.Input) {
    const existing = await this.repo.findByEmail(input.email);
    if (existing) throw new ConflictError('Email already registered'); // business rule
  }
}
```

## Responsibility Summary

| Concern                      | Layer     | Tool              |
| ---------------------------- | --------- | ----------------- |
| Field presence (required)    | 1 — DTO   | `@IsNotEmpty()`   |
| Type check (string/number)   | 1 — DTO   | `@IsString()`     |
| Format check (email/url)     | 1 — DTO   | `@IsEmail()`      |
| Enum membership              | 1 — DTO   | `@IsEnum()`       |
| Cross-field rule (XOR)       | 1 — DTO   | `@ValidatorConstraint` |
| Layer contract               | 2 — App   | TypeScript        |
| Business invariant           | 3 — Domain VO | manual throw  |
| State machine transition     | 3 — Domain VO | manual throw  |
| Uniqueness constraint        | 3 — Use Case | repo query    |
| Queue message structure      | 4 — Schema | Zod             |
| Queue message type safety    | 4 — Schema | Zod `.infer<>` |
