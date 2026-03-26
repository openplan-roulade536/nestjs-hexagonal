# Module Wiring — Full Template

## File Location

```
infrastructure/<context>.module.ts
```

## Full Module (CQRS Pattern)

```typescript
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';

// Shared infra
import { DatabaseModule } from '@/shared/infrastructure/database/database.module';

// Domain
import { <CONTEXT>_REPOSITORY } from '../domain/repositories/<context>.repository';

// Infrastructure
import { Prisma<Context>Repository } from './database/prisma/repositories/prisma-<context>.repository';

// CQRS Handlers
import { Create<Context>Handler } from '../application/commands/create-<context>.handler';
import { Update<Context>Handler } from '../application/commands/update-<context>.handler';
import { Delete<Context>Handler } from '../application/commands/delete-<context>.handler';
import { Get<Context>Handler } from '../application/queries/get-<context>.handler';
import { List<Context>Handler } from '../application/queries/list-<context>.handler';

// Event handlers (side effects)
import { <Context>CreatedHandler } from './listeners/<context>-created.handler';
import { <Context>UpdatedProjection } from './listeners/<context>-updated.projection';

// Adapters (ports implemented by this module)
import { <Context>Adapter } from './adapters/<context>.adapter';

// External port token (consumed by other modules)
import { <CONTEXT>_PORT } from '@/<area>/<consumer>/application/ports/<context>.port';

// Controllers
import { <Context>Controller } from './controllers/<context>.controller';

@Module({
  imports: [CqrsModule, DatabaseModule],
  controllers: [<Context>Controller],
  providers: [
    // Repository implementation + port binding
    Prisma<Context>Repository,
    { provide: <CONTEXT>_REPOSITORY, useExisting: Prisma<Context>Repository },

    // CQRS Command Handlers (auto-registered via @CommandHandler)
    Create<Context>Handler,
    Update<Context>Handler,
    Delete<Context>Handler,

    // CQRS Query Handlers (auto-registered via @QueryHandler)
    Get<Context>Handler,
    List<Context>Handler,

    // Domain Event Handlers (auto-registered via @EventsHandler)
    <Context>CreatedHandler,
    <Context>UpdatedProjection,

    // Adapter + external port binding
    <Context>Adapter,
    { provide: <CONTEXT>_PORT, useExisting: <Context>Adapter },
  ],
  // ONLY port token Symbols — never use cases, never Prisma classes
  exports: [<CONTEXT>_REPOSITORY, <CONTEXT>_PORT],
})
export class <Context>Module {}
```

## Module (Plain UseCase Pattern)

When the context uses plain use cases instead of CQRS:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from '@/shared/infrastructure/database/database.module';
import { <CONTEXT>_REPOSITORY } from '../domain/repositories/<context>.repository';
import { Prisma<Context>Repository } from './database/prisma/repositories/prisma-<context>.repository';
import {
  CREATE_<CONTEXT>_USE_CASE,
  Create<Context>UseCase,
} from '../application/use-cases/create-<context>.use-case';
import {
  GET_<CONTEXT>_USE_CASE,
  Get<Context>UseCase,
} from '../application/use-cases/get-<context>.use-case';
import {
  LIST_<CONTEXT>S_USE_CASE,
  List<Context>sUseCase,
} from '../application/use-cases/list-<context>s.use-case';
import { <Context>Controller } from './controllers/<context>.controller';
import type { <Context>Repository } from '../domain/repositories/<context>.repository';

@Module({
  imports: [DatabaseModule],
  controllers: [<Context>Controller],
  providers: [
    Prisma<Context>Repository,
    { provide: <CONTEXT>_REPOSITORY, useExisting: Prisma<Context>Repository },

    {
      provide: CREATE_<CONTEXT>_USE_CASE,
      useFactory: (repo: <Context>Repository.Repository) =>
        new Create<Context>UseCase(repo),
      inject: [<CONTEXT>_REPOSITORY],
    },
    {
      provide: GET_<CONTEXT>_USE_CASE,
      useFactory: (repo: <Context>Repository.Repository) =>
        new Get<Context>UseCase(repo),
      inject: [<CONTEXT>_REPOSITORY],
    },
    {
      provide: LIST_<CONTEXT>S_USE_CASE,
      useFactory: (repo: <Context>Repository.Repository) =>
        new List<Context>sUseCase(repo),
      inject: [<CONTEXT>_REPOSITORY],
    },
  ],
  exports: [<CONTEXT>_REPOSITORY],
})
export class <Context>Module {}
```

## forTest() Static Method

Add `forTest()` when integration or e2e tests need to build a partial module with a real database:

```typescript
@Module({ ... })
export class <Context>Module {
  static forTest(prismaClient: PrismaService): DynamicModule {
    return {
      module: <Context>Module,
      imports: [],
      providers: [
        { provide: PrismaService, useValue: prismaClient },
        Prisma<Context>Repository,
        { provide: <CONTEXT>_REPOSITORY, useExisting: Prisma<Context>Repository },
        Create<Context>Handler,
        Get<Context>Handler,
      ],
      exports: [<CONTEXT>_REPOSITORY],
    };
  }
}
```

## Cross-Module Integration

When module A needs a port from module B:

```typescript
// In Module A:
@Module({
  imports: [
    CqrsModule,
    DatabaseModule,
    forwardRef(() => ModuleBModule),  // use forwardRef only for circular deps
  ],
  ...
})

// Module B must export the port token:
// exports: ['<ExternalPort>'] or exports: [EXTERNAL_PORT_TOKEN]
```

## Rules

- `exports` contains ONLY Symbol port tokens — never concrete classes.
- Use cases registered with `useFactory` + `inject` — never `useClass` for use cases.
- CQRS handlers (`@CommandHandler`, `@QueryHandler`, `@EventsHandler`) are listed as class values — NestJS auto-registers them.
- `forwardRef()` only for genuine circular dependencies. Prefer restructuring the dependency graph.
- Always import `CqrsModule` when using CQRS handlers.
- Always import `DatabaseModule` (or equivalent) when using `PrismaService`.
