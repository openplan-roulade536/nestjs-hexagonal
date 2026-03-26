# Pattern A — Plain UseCase with TOKEN

Full template for modules that do not use CQRS. The use case is a pure TypeScript class;
NestJS wiring happens exclusively in the module provider via `useFactory`.

---

## Directory layout

```
application/
├── dtos/
│   ├── create-<context>.dto.ts
│   └── <context>-output.dto.ts
└── usecases/
    ├── create-<context>.usecase.ts
    └── __tests__/
        └── create-<context>.usecase.spec.ts
```

---

## DTO Namespace

```typescript
// application/dtos/create-<context>.dto.ts
export namespace Create<Context>Dto {
  export interface Input {
    companyId: string;
    name: string;
    description?: string;
  }

  export interface Output {
    id: string;
    companyId: string;
    name: string;
    description?: string;
    createdAt: Date;
  }
}
```

---

## Output Mapper

```typescript
// application/dtos/<context>-output.dto.ts
import type { <Context>Entity } from '../../domain/entities/<context>.entity';

export interface <Context>Output {
  id: string;
  companyId: string;
  name: string;
  description?: string;
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
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    };
  }
}
```

---

## UseCase

No `@Injectable`, no `@Inject`, no NestJS imports. The TOKEN symbol is exported from this file.

```typescript
// application/usecases/create-<context>.usecase.ts
import type { <Context>Repository } from '../../domain/repositories/<context>.repository';
import type { Create<Context>Dto } from '../dtos/create-<context>.dto';
import { <Context>Entity } from '../../domain/entities/<context>.entity';
import { <Context>OutputMapper } from '../dtos/<context>-output.dto';

export const CREATE_<CONTEXT>_USE_CASE_TOKEN = Symbol('Create<Context>UseCase');

export namespace Create<Context>UseCase {
  export type Input = Create<Context>Dto.Input;
  export type Output = Create<Context>Dto.Output;

  export class UseCase {
    constructor(
      private readonly repository: <Context>Repository.Repository,
    ) {}

    async execute(input: Input): Promise<Output> {
      const entity = <Context>Entity.create({
        companyId: input.companyId,
        name: input.name,
        description: input.description,
      });

      await this.repository.insert(entity);

      return <Context>OutputMapper.toOutput(entity);
    }
  }
}
```

---

## Module Registration

```typescript
// infrastructure/<context>.module.ts
import { Module } from '@nestjs/common';

import {
  CREATE_<CONTEXT>_USE_CASE_TOKEN,
  Create<Context>UseCase,
} from '../application/usecases/create-<context>.usecase';
import { <CONTEXT>_REPOSITORY_TOKEN } from '../domain/repositories/<context>.repository';
import type { <Context>Repository } from '../domain/repositories/<context>.repository';
import { <Context>Controller } from './controllers/<context>.controller';
import { Prisma<Context>Repository } from './repositories/prisma-<context>.repository';

@Module({
  controllers: [<Context>Controller],
  providers: [
    Prisma<Context>Repository,
    {
      provide: <CONTEXT>_REPOSITORY_TOKEN,
      useExisting: Prisma<Context>Repository,
    },
    {
      provide: CREATE_<CONTEXT>_USE_CASE_TOKEN,
      useFactory: (repo: <Context>Repository.Repository) =>
        new Create<Context>UseCase.UseCase(repo),
      inject: [<CONTEXT>_REPOSITORY_TOKEN],
    },
  ],
  exports: [<CONTEXT>_REPOSITORY_TOKEN], // NEVER export use cases
})
export class <Context>Module {}
```

---

## Controller Injection

```typescript
// infrastructure/controllers/<context>.controller.ts
import { Body, Controller, Inject, Post, Req } from '@nestjs/common';
import type { FastifyRequest } from 'fastify';

import {
  CREATE_<CONTEXT>_USE_CASE_TOKEN,
  Create<Context>UseCase,
} from '../../application/usecases/create-<context>.usecase';
import { Create<Context>RequestDto } from './dtos/create-<context>.request.dto';

@Controller('<contexts>')
export class <Context>Controller {
  constructor(
    @Inject(CREATE_<CONTEXT>_USE_CASE_TOKEN)
    private readonly createUseCase: Create<Context>UseCase.UseCase,
  ) {}

  @Post()
  async create(
    @Body() dto: Create<Context>RequestDto,
    @Req() req: FastifyRequest,
  ) {
    return this.createUseCase.execute({
      companyId: req.user.companyId,
      name: dto.name,
      description: dto.description,
    });
  }
}
```

---

## Unit Test

The use case is instantiated directly with mocked dependencies — no NestJS test module needed.

```typescript
// application/usecases/__tests__/create-<context>.usecase.spec.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';

import { <Context>Entity } from '../../../domain/entities/<context>.entity';
import { Create<Context>UseCase } from '../create-<context>.usecase';

const mockRepository = {
  insert: vi.fn(),
  findById: vi.fn(),
  findByCompany: vi.fn(),
  update: vi.fn(),
  delete: vi.fn(),
};

describe('Create<Context>UseCase', () => {
  let useCase: Create<Context>UseCase.UseCase;

  beforeEach(() => {
    vi.clearAllMocks();
    useCase = new Create<Context>UseCase.UseCase(mockRepository);
  });

  it('creates entity and returns output', async () => {
    const input: Create<Context>UseCase.Input = {
      companyId: 'company-1',
      name: 'Test Item',
    };

    const result = await useCase.execute(input);

    expect(mockRepository.insert).toHaveBeenCalledOnce();
    expect(result.id).toBeDefined();
    expect(result.companyId).toBe('company-1');
    expect(result.name).toBe('Test Item');
  });

  it('throws if entity invariant is violated', async () => {
    await expect(
      useCase.execute({ companyId: 'company-1', name: '' }),
    ).rejects.toThrow();
  });
});
```

---

## When to Add a Second Dependency

If the use case needs a port (cross-module interface), add it to the constructor:

```typescript
export class Create<Context>UseCase.UseCase {
  constructor(
    private readonly repository: <Context>Repository.Repository,
    private readonly notificationPort: NotificationPort,
  ) {}
}
```

And update the module `useFactory`:

```typescript
{
  provide: CREATE_<CONTEXT>_USE_CASE_TOKEN,
  useFactory: (
    repo: <Context>Repository.Repository,
    notification: NotificationPort,
  ) => new Create<Context>UseCase.UseCase(repo, notification),
  inject: [<CONTEXT>_REPOSITORY_TOKEN, NOTIFICATION_PORT_TOKEN],
},
```

---

## Checklist

- [ ] TOKEN symbol exported from use case file, not from module.
- [ ] UseCase class has no `@Injectable`, no NestJS imports.
- [ ] Module uses `useFactory` + `inject`, never `useClass`.
- [ ] Module exports only the repository PORT token.
- [ ] Controller injects via `@Inject(TOKEN)` with interface type annotation.
- [ ] Unit test instantiates use case directly (no `Test.createTestingModule`).
- [ ] Unit test mocks all repository methods that the use case calls.
