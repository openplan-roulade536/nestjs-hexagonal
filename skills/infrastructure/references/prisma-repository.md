# Prisma Repository — Full Template

## File Location

```
infrastructure/database/prisma/repositories/prisma-<context>.repository.ts
```

## Full Implementation

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@/shared/infrastructure/database/prisma.service';
import { <Context>Entity } from '../../../domain/entities/<context>.entity';
import type { <Context>Repository } from '../../../domain/repositories/<context>.repository';
import { <Context>ModelMapper } from '../models/<context>-model.mapper';

@Injectable()
export class Prisma<Context>Repository implements <Context>Repository.Repository {
  sortableFields: string[] = ['name', 'createdAt', 'updatedAt'];

  constructor(private readonly prisma: PrismaService) {}

  async findById(id: string): Promise<<Context>Entity | null> {
    const record = await this.prisma.<context>.findUnique({
      where: { id },
    });
    if (!record) return null;
    return <Context>ModelMapper.toEntity(record);
  }

  async findByOrganization(organizationId: string): Promise<<Context>Entity[]> {
    const records = await this.prisma.<context>.findMany({
      where: { organizationId },
      orderBy: { createdAt: 'desc' },
    });
    return records.map(<Context>ModelMapper.toEntity);
  }

  async save(entity: <Context>Entity): Promise<void> {
    const data = <Context>ModelMapper.toModel(entity);
    const existing = await this.prisma.<context>.findUnique({
      where: { id: entity.id },
      select: { id: true },
    });

    if (existing) {
      await this.prisma.<context>.update({
        where: { id: entity.id },
        data: { ...data, updatedAt: new Date() },
      });
    } else {
      await this.prisma.<context>.create({ data });
    }
  }

  async search(
    props: <Context>Repository.SearchParams,
  ): Promise<<Context>Repository.SearchResult> {
    const sortable = this.sortableFields.includes(props.sort ?? '') || false;
    const orderByField = sortable ? props.sort! : 'createdAt';
    const orderByDir = (sortable ? props.sortDir : 'desc') ?? 'desc';

    const whereClause: Record<string, unknown> = {};

    if (props.filter?.organizationId) {
      whereClause.organizationId = props.filter.organizationId;
    }

    if (typeof props.filter?.name === 'string' && props.filter.name !== '') {
      whereClause.name = { contains: props.filter.name, mode: 'insensitive' };
    }

    const page = props.page ?? 1;
    const perPage = props.perPage ?? 15;

    const [count, records] = await Promise.all([
      this.prisma.<context>.count({ where: whereClause }),
      this.prisma.<context>.findMany({
        where: whereClause,
        orderBy: { [orderByField]: orderByDir },
        skip: page > 0 ? (page - 1) * perPage : 0,
        take: perPage > 0 ? perPage : 15,
      }),
    ]);

    return new <Context>Repository.SearchResult({
      items: records.map(<Context>ModelMapper.toEntity),
      total: count,
      currentPage: page,
      perPage,
      sort: orderByField,
      sortDir: orderByDir,
      filter: props.filter ?? null,
    });
  }

  async delete(id: string, organizationId: string): Promise<void> {
    await this.prisma.<context>.delete({
      where: { id, organizationId },
    });
  }
}
```

## Model Mapper

```typescript
// infrastructure/database/prisma/models/<context>-model.mapper.ts
import { <Context>Entity, type <Context>Props } from '../../../domain/entities/<context>.entity';
import type { <Context> } from '@prisma/client';

export class <Context>ModelMapper {
  static toEntity(model: <Context>): <Context>Entity {
    return <Context>Entity.restore(
      {
        organizationId: model.organizationId,
        name: model.name,
        createdAt: model.createdAt,
        updatedAt: model.updatedAt ?? undefined,
        // ... other props
      } satisfies <Context>Props,
      model.id,
    );
  }

  static toModel(entity: <Context>Entity): Omit<<Context>, 'id'> & { id: string } {
    return {
      id: entity.id,
      organizationId: entity.organizationId,
      name: entity.name,
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt ?? null,
    };
  }
}
```

## Integration Test

```typescript
// infrastructure/database/prisma/repositories/__tests__/prisma-<context>.repository.spec.ts
import { describe, it, beforeEach, afterEach, expect } from 'vitest';
import { Test } from '@nestjs/testing';
import { PrismaService } from '@/shared/infrastructure/database/prisma.service';
import { DatabaseModule } from '@/shared/infrastructure/database/database.module';
import { Prisma<Context>Repository } from '../prisma-<context>.repository';
import { <Context>Entity } from '../../../../domain/entities/<context>.entity';

describe('Prisma<Context>Repository (integration)', () => {
  let repo: Prisma<Context>Repository;
  let prisma: PrismaService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      imports: [DatabaseModule],
      providers: [Prisma<Context>Repository],
    }).compile();

    repo = module.get(Prisma<Context>Repository);
    prisma = module.get(PrismaService);
    await prisma.$connect();
    // Truncate relevant tables for isolation
    await prisma.$executeRaw`TRUNCATE TABLE "<contexts>" CASCADE`;
  });

  afterEach(async () => {
    await prisma.$disconnect();
  });

  it('saves and retrieves an entity by id', async () => {
    const entity = <Context>Entity.create({
      organizationId: 'org-1',
      name: 'Test',
    });

    await repo.save(entity);

    const found = await repo.findById(entity.id);
    expect(found).not.toBeNull();
    expect(found?.name).toBe('Test');
    expect(found?.organizationId).toBe('org-1');
  });

  it('returns null for non-existent id', async () => {
    const result = await repo.findById('non-existent-id');
    expect(result).toBeNull();
  });

  it('updates an existing entity on save', async () => {
    const entity = <Context>Entity.create({ organizationId: 'org-1', name: 'Original' });
    await repo.save(entity);

    entity.rename('Updated');
    await repo.save(entity);

    const found = await repo.findById(entity.id);
    expect(found?.name).toBe('Updated');
  });

  it('deletes an entity scoped to organizationId', async () => {
    const entity = <Context>Entity.create({ organizationId: 'org-1', name: 'ToDelete' });
    await repo.save(entity);

    await repo.delete(entity.id, 'org-1');

    const found = await repo.findById(entity.id);
    expect(found).toBeNull();
  });

  it('search returns paginated results filtered by organizationId', async () => {
    await Promise.all([
      repo.save(<Context>Entity.create({ organizationId: 'org-1', name: 'Alpha' })),
      repo.save(<Context>Entity.create({ organizationId: 'org-1', name: 'Beta' })),
      repo.save(<Context>Entity.create({ organizationId: 'org-2', name: 'Gamma' })),
    ]);

    const result = await repo.search(
      new <Context>Repository.SearchParams({
        page: 1,
        perPage: 10,
        filter: { organizationId: 'org-1' },
      }),
    );

    expect(result.total).toBe(2);
    expect(result.items).toHaveLength(2);
    expect(result.items.every((i) => i.organizationId === 'org-1')).toBe(true);
  });
});
```

## Notes

- Always use `Promise.all([count, findMany])` for `search()` — avoids two round-trips.
- `delete()` must include `organizationId` in the where clause to prevent cross-tenant deletion.
- Mapper `toEntity()` always calls `Entity.restore()` — never `Entity.create()` (no events emitted from DB reads).
- Use `select: { id: true }` in the upsert existence check to minimize data transfer.
