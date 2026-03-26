# DTO Patterns

DTOs (Data Transfer Objects) carry data between layers. The application layer has
two kinds: Application DTOs (contracts between application and domain) and Output
Mappers (entity → response shape).

Do not confuse these with Request DTOs (`*.request.dto.ts` in infrastructure/controllers/dtos/)
which use `class-validator` decorators.

---

## Layer Responsibilities

```
infrastructure/controllers/dtos/  → Request DTOs  (class-validator, Swagger)
application/dtos/                 → Application DTOs + Output Mappers (plain TS)
domain/                           → Value Objects (manual validation, no frameworks)
```

Never put `class-validator` decorators in `application/dtos/`.
Never put business logic in Request DTOs.

---

## 1. Namespace Pattern (preferred for write operations)

Groups Input and Output types under a named namespace in one file.

```typescript
// application/dtos/create-<context>.dto.ts
export namespace Create<Context>Dto {
  export interface Input {
    companyId: string;
    name: string;
    description?: string;
    metadata?: Record<string, unknown>;
  }

  export interface Output {
    id: string;
  }
}
```

Usage in use case:

```typescript
import type { Create<Context>Dto } from '../dtos/create-<context>.dto';

export class UseCase {
  async execute(input: Create<Context>Dto.Input): Promise<Create<Context>Dto.Output> {
    // ...
    return { id: entity.id };
  }
}
```

Usage in command:

```typescript
import { Command } from '@nestjs/cqrs';
import type { Create<Context>Dto } from '../dtos/create-<context>.dto';

export class Create<Context>Command extends Command<Create<Context>Dto.Output> {
  constructor(
    public readonly companyId: string,
    public readonly name: string,
  ) { super(); }
}
```

---

## 2. Inline in Command Class

Acceptable for simple commands where a separate DTO file would be boilerplate.

```typescript
// application/commands/create-<context>.command.ts
import { Command } from '@nestjs/cqrs';

export class Create<Context>Command extends Command<{ id: string }> {
  constructor(
    public readonly companyId: string,
    public readonly data: {
      name: string;
      description?: string;
    },
  ) {
    super();
  }
}
```

Use inline when:
- Output is a simple `{ id: string }`.
- Input has 2-4 fields and won't be reused.
- The command is internal and not exposed via HTTP directly.

Use a separate DTO file when:
- Output has a complex shape that maps from an entity.
- The same Input/Output type is referenced by multiple files.
- The DTO is part of a public API contract.

---

## 3. Output Mapper

Converts a domain entity to a response DTO. Keeps mapping logic out of handlers
and use cases.

```typescript
// application/dtos/<context>-output.dto.ts
import type { <Context>Entity } from '../../domain/entities/<context>.entity';

export interface <Context>Output {
  id: string;
  companyId: string;
  name: string;
  description?: string;
  status: string;
  createdAt: Date;
  updatedAt?: Date;
}

export class <Context>OutputMapper {
  static toOutput(entity: <Context>Entity): <Context>Output {
    return {
      id: entity.id,
      companyId: entity.companyId,
      name: entity.name,
      description: entity.description,
      status: entity.status.toString(),
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    };
  }

  static toList(entities: <Context>Entity[]): <Context>Output[] {
    return entities.map(<Context>OutputMapper.toOutput);
  }
}
```

---

## 4. Separate Get / List DTOs

For read operations, define query-specific DTOs:

```typescript
// application/dtos/get-<context>.dto.ts
import type { <Context>Output } from './<context>-output.dto';

export namespace Get<Context>Dto {
  export interface Input {
    id: string;
    companyId: string;
  }

  export type Output = <Context>Output;
}
```

```typescript
// application/dtos/list-<contexts>.dto.ts
import type { <Context>Output } from './<context>-output.dto';

export namespace List<Contexts>Dto {
  export interface Input {
    companyId: string;
    page?: number;
    perPage?: number;
  }

  export interface Output {
    items: <Context>Output[];
    total: number;
    currentPage: number;
    perPage: number;
    lastPage: number;
  }
}
```

---

## 5. Nested Output DTOs

When the output includes related entities:

```typescript
// application/dtos/customer-output.dto.ts
export interface AddressOutput {
  id: string;
  street: string;
  number: string;
  complement?: string;
  city: string;
  state: string;
  country: string;
}

export interface CustomerOutput {
  id: string;
  companyId: string;
  name: string;
  email?: string;
  phone?: string;
  document: {
    number: string;
    type: string;
    formatted: string;
  };
  address?: AddressOutput;
  createdAt: Date;
  updatedAt?: Date;
}

export class CustomerOutputMapper {
  static toOutput(entity: CustomerEntity): CustomerOutput {
    return {
      id: entity.id,
      companyId: entity.companyId,
      name: entity.name,
      email: entity.email,
      phone: entity.phone,
      document: {
        number: entity.document.getValue(),
        type: entity.document.getType(),
        formatted: entity.document.getFormattedValue(),
      },
      address: entity.address
        ? {
            id: entity.address.id,
            street: entity.address.street,
            number: entity.address.number,
            complement: entity.address.complement,
            city: entity.address.city,
            state: entity.address.state,
            country: entity.address.country,
          }
        : undefined,
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    };
  }
}
```

---

## 6. Request DTO (infrastructure layer, NOT application)

This is shown for contrast. Request DTOs live in
`infrastructure/controllers/dtos/` and use `class-validator`:

```typescript
// infrastructure/controllers/dtos/create-<context>.request.dto.ts
import { IsNotEmpty, IsOptional, IsString } from 'class-validator';

export class Create<Context>RequestDto {
  @IsString()
  @IsNotEmpty()
  name!: string;

  @IsString()
  @IsOptional()
  description?: string;
}
```

The controller maps the Request DTO to the Application DTO Input:

```typescript
@Post()
async create(@Body() dto: Create<Context>RequestDto, @Req() req: FastifyRequest) {
  return this.commandBus.execute(
    new Create<Context>Command(req.user.companyId, dto.name, dto.description),
  );
}
```

---

## Decision Guide

| Situation | Use |
|-----------|-----|
| Output has 5+ fields from an entity | Separate `<context>-output.dto.ts` with mapper |
| Write command, output is just `{ id }` | Inline in command or `Command<{ id: string }>` |
| Input shared by 2+ files | Namespace in `application/dtos/` |
| Read query with pagination | Separate `list-<contexts>.dto.ts` |
| HTTP request validation | Request DTO in `infrastructure/controllers/dtos/` with class-validator |
| HTTP response shape | Output DTO in `application/dtos/` (plain interface) |

---

## Checklist

- [ ] Application DTOs have no `class-validator` decorators.
- [ ] Request DTOs have no business logic.
- [ ] Output mapper is a static class method, not an inline function.
- [ ] Namespace `Input`/`Output` used when the same DTO crosses multiple files.
- [ ] `toList()` helper added to mapper when list queries exist.
- [ ] Nested types (address, document) defined as separate interfaces in the same file.
