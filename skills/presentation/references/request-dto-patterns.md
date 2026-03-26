# Request DTO Patterns — Full Template

`class-validator` and `class-transformer` live ONLY in request DTOs. Global `ValidationPipe` applies them automatically.

## Global ValidationPipe (main.ts)

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,           // auto-transform types (string -> number, etc.)
    whitelist: true,           // strip properties without decorators
    forbidNonWhitelisted: true, // throw 400 for unexpected properties
  }),
);
```

## Basic Create DTO

```typescript
// controllers/dtos/create-<context>.request.dto.ts
import {
  IsString,
  IsNotEmpty,
  IsOptional,
  IsEmail,
  IsUrl,
  IsEnum,
  IsBoolean,
  IsInt,
  Min,
  Max,
  MinLength,
  MaxLength,
  IsUUID,
} from 'class-validator';
import { Transform } from 'class-transformer';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { <Status>Enum } from '../../domain/value-objects/<status>.vo';

export class Create<Context>RequestDto {
  @ApiProperty({ description: 'Name of the <context>', example: 'My <Context>' })
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(100)
  name!: string;

  @ApiPropertyOptional({ description: 'Optional description', example: 'A description' })
  @IsOptional()
  @IsString()
  @MaxLength(500)
  description?: string;

  @ApiProperty({ description: 'Owner email', example: 'owner@example.com' })
  @IsEmail()
  @IsNotEmpty()
  @Transform(({ value }) => (value as string).trim().toLowerCase())
  email!: string;

  @ApiPropertyOptional({ enum: <Status>Enum, description: 'Status' })
  @IsOptional()
  @IsEnum(<Status>Enum)
  status?: <Status>Enum;

  @ApiPropertyOptional({ description: 'Sort order', example: 0 })
  @IsOptional()
  @IsInt()
  @Min(0)
  @Max(9999)
  @Transform(({ value }) => Number(value))
  order?: number;

  @ApiPropertyOptional({ description: 'Active flag', default: true })
  @IsOptional()
  @IsBoolean()
  @Transform(({ value }) => Boolean(value))
  active?: boolean;
}
```

## Update DTO (Partial)

```typescript
// controllers/dtos/update-<context>.request.dto.ts
import { PartialType } from '@nestjs/swagger';
import { Create<Context>RequestDto } from './create-<context>.request.dto';

export class Update<Context>RequestDto extends PartialType(Create<Context>RequestDto) {}
```

## List / Search DTO

```typescript
// controllers/dtos/list-<context>.request.dto.ts
import { IsOptional, IsString, IsInt, Min, IsIn } from 'class-validator';
import { Transform, Type } from 'class-transformer';
import { ApiPropertyOptional } from '@nestjs/swagger';

export class List<Context>RequestDto {
  @ApiPropertyOptional({ example: 1, default: 1 })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  page?: number = 1;

  @ApiPropertyOptional({ example: 15, default: 15 })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  perPage?: number = 15;

  @ApiPropertyOptional({ example: 'search term' })
  @IsOptional()
  @IsString()
  name?: string;

  @ApiPropertyOptional({ example: 'createdAt', enum: ['name', 'createdAt', 'updatedAt'] })
  @IsOptional()
  @IsString()
  @IsIn(['name', 'createdAt', 'updatedAt'])
  sort?: string;

  @ApiPropertyOptional({ example: 'desc', enum: ['asc', 'desc'] })
  @IsOptional()
  @IsIn(['asc', 'desc'])
  sortDir?: 'asc' | 'desc';
}
```

## Nested DTO

```typescript
// controllers/dtos/create-order.request.dto.ts
import {
  IsArray, ValidateNested, ArrayMinSize, IsString, IsNotEmpty,
  IsNumber, IsInt, Min,
} from 'class-validator';
import { Type } from 'class-transformer';
import { ApiProperty } from '@nestjs/swagger';

export class OrderItemDto {
  @ApiProperty({ description: 'Product ID' })
  @IsString()
  @IsNotEmpty()
  productId!: string;

  @ApiProperty({ description: 'Quantity', minimum: 1 })
  @IsInt()
  @Min(1)
  @Type(() => Number)
  quantity!: number;
}

export class CreateOrderRequestDto {
  @ApiProperty({ type: () => [OrderItemDto], minItems: 1 })
  @IsArray()
  @ArrayMinSize(1, { message: 'Order must have at least one item' })
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items!: OrderItemDto[];
}
```

## Custom Cross-Field Validator

```typescript
// validators/<rule>.validator.ts
import {
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
  ValidationOptions,
  registerDecorator,
} from 'class-validator';

// Example: require either 'entityId' OR 'entityData', not both, not neither
@ValidatorConstraint({ name: 'requireOneOf', async: false })
export class RequireOneOfValidator implements ValidatorConstraintInterface {
  validate(_value: unknown, args: ValidationArguments): boolean {
    const obj = args.object as Record<string, unknown>;
    const [field1, field2] = args.constraints as [string, string];
    const has1 = obj[field1] !== undefined && obj[field1] !== null;
    const has2 = obj[field2] !== undefined && obj[field2] !== null;
    return has1 !== has2; // XOR: exactly one must be present
  }

  defaultMessage(args: ValidationArguments): string {
    const [field1, field2] = args.constraints as [string, string];
    return `Exactly one of '${field1}' or '${field2}' must be provided`;
  }
}

export function RequireOneOf(
  field1: string,
  field2: string,
  options?: ValidationOptions,
) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options,
      constraints: [field1, field2],
      validator: RequireOneOfValidator,
    });
  };
}

// Usage in DTO:
export class CreateChargeDto {
  @IsOptional()
  @ValidateNested()
  @Type(() => CustomerDataDto)
  @RequireOneOf('customer', 'customerId', {
    message: 'Provide either customer data or customerId',
  })
  customer?: CustomerDataDto;

  @IsOptional()
  @IsString()
  customerId?: string;
}
```

## Custom Format Validator

```typescript
// validators/document.validator.ts
import {
  ValidatorConstraint, ValidatorConstraintInterface,
  ValidationOptions, registerDecorator,
} from 'class-validator';

@ValidatorConstraint({ name: 'isBrazilianDocument', async: false })
export class BrazilianDocumentValidator implements ValidatorConstraintInterface {
  validate(value: unknown): boolean {
    if (typeof value !== 'string') return false;
    const digits = value.replace(/\D/g, '');
    return digits.length === 11 || digits.length === 14; // CPF or CNPJ
  }

  defaultMessage(): string {
    return 'Document must be a valid CPF (11 digits) or CNPJ (14 digits)';
  }
}

export function IsBrazilianDocument(options?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options,
      constraints: [],
      validator: BrazilianDocumentValidator,
    });
  };
}
```

## DTO Tests

```typescript
// controllers/dtos/__tests__/create-<context>.request.dto.spec.ts
import { describe, it, expect } from 'vitest';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';
import { Create<Context>RequestDto } from '../create-<context>.request.dto';

async function validateDto(plain: Record<string, unknown>) {
  const dto = plainToInstance(Create<Context>RequestDto, plain);
  return validate(dto);
}

describe('Create<Context>RequestDto', () => {
  it('accepts a valid payload', async () => {
    const errors = await validateDto({ name: 'Valid Name', email: 'test@example.com' });
    expect(errors).toHaveLength(0);
  });

  it('rejects missing name', async () => {
    const errors = await validateDto({ email: 'test@example.com' });
    expect(errors.some((e) => e.property === 'name')).toBe(true);
  });

  it('rejects invalid email format', async () => {
    const errors = await validateDto({ name: 'Name', email: 'not-an-email' });
    expect(errors.some((e) => e.property === 'email')).toBe(true);
  });

  it('normalizes email to lowercase', async () => {
    const dto = plainToInstance(Create<Context>RequestDto, {
      name: 'Name',
      email: 'TEST@EXAMPLE.COM',
    });
    expect(dto.email).toBe('test@example.com');
  });

  it('strips unknown properties (whitelist)', async () => {
    const dto = plainToInstance(Create<Context>RequestDto, {
      name: 'Name',
      email: 'test@example.com',
      unknownField: 'should be stripped',
    });
    expect((dto as Record<string, unknown>).unknownField).toBeUndefined();
  });
});
```

## Rules

- `@IsOptional()` must always come BEFORE other decorators on optional fields.
- Use `@Type(() => NestedDto)` for nested objects and `@Type(() => Number)` for numeric query params.
- Use `@Transform` sparingly — prefer `@Type` for type coercion.
- `PartialType` (from `@nestjs/swagger`, not `@nestjs/mapped-types`) preserves Swagger decorators.
- Never put business logic in DTOs — validation only (format, presence, types).
- Never import from domain layer in DTOs — keep presentation isolated.
