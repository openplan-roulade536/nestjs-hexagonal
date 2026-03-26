# Controller Patterns — Full Template

## Full CRUD Controller (CQRS)

```typescript
// infrastructure/controllers/<context>.controller.ts
import {
  Controller,
  Post,
  Get,
  Patch,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
  UseGuards,
  NotFoundException,
} from '@nestjs/common';
import { CommandBus, QueryBus } from '@nestjs/cqrs';
import {
  ApiTags,
  ApiBearerAuth,
  ApiOperation,
  ApiResponse,
  ApiBody,
  ApiParam,
  ApiQuery,
} from '@nestjs/swagger';
import { AuthGuard } from '@/auth/guards/auth.guard';
import { RolesGuard } from '@/auth/guards/roles.guard';
import { Roles } from '@/auth/decorators/roles.decorator';
import { CurrentUser, User } from '@/auth/decorators/current-user.decorator';
import { CurrentOrganization, OrgContext } from '@/auth/decorators/current-organization.decorator';
import { Create<Context>RequestDto } from './dtos/create-<context>.request.dto';
import { Update<Context>RequestDto } from './dtos/update-<context>.request.dto';
import { List<Context>RequestDto } from './dtos/list-<context>.request.dto';
import { Create<Context>Command } from '../../application/commands/create-<context>.command';
import { Update<Context>Command } from '../../application/commands/update-<context>.command';
import { Delete<Context>Command } from '../../application/commands/delete-<context>.command';
import { Get<Context>Query } from '../../application/queries/get-<context>.query';
import { List<Context>Query } from '../../application/queries/list-<context>.query';

@ApiTags('<Contexts>')
@Controller('<contexts>')
@UseGuards(AuthGuard, RolesGuard)
@ApiBearerAuth()
export class <Context>Controller {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly queryBus: QueryBus,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @Roles(['admin', 'member'])
  @ApiOperation({
    summary: 'Create a new <context>',
    description: 'Creates a <context> scoped to the current organization.',
  })
  @ApiBody({ type: Create<Context>RequestDto })
  @ApiResponse({ status: 201, description: '<Context> created successfully' })
  @ApiResponse({ status: 400, description: 'Validation error — check request body' })
  @ApiResponse({ status: 401, description: 'Authentication required' })
  @ApiResponse({ status: 403, description: 'Insufficient permissions' })
  async create(
    @Body() dto: Create<Context>RequestDto,
    @CurrentOrganization() org: OrgContext,
  ): Promise<{ id: string }> {
    return this.commandBus.execute(
      new Create<Context>Command(org.id, dto.name, dto.description),
    );
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get <context> by ID' })
  @ApiParam({ name: 'id', description: '<Context> UUID' })
  @ApiResponse({ status: 200, description: '<Context> found' })
  @ApiResponse({ status: 404, description: '<Context> not found' })
  async findOne(
    @Param('id') id: string,
    @CurrentOrganization() org: OrgContext,
  ) {
    const result = await this.queryBus.execute(new Get<Context>Query(id, org.id));
    if (!result) throw new NotFoundException('<Context> not found');
    return result;
  }

  @Get()
  @ApiOperation({ summary: 'List <contexts> with pagination and filtering' })
  @ApiQuery({ name: 'page', required: false, type: Number, example: 1 })
  @ApiQuery({ name: 'perPage', required: false, type: Number, example: 15 })
  @ApiQuery({ name: 'name', required: false, type: String })
  @ApiResponse({ status: 200, description: 'Paginated list of <contexts>' })
  async findAll(
    @Query() query: List<Context>RequestDto,
    @CurrentOrganization() org: OrgContext,
  ) {
    return this.queryBus.execute(
      new List<Context>Query({
        organizationId: org.id,
        page: query.page,
        perPage: query.perPage,
        name: query.name,
        sort: query.sort,
        sortDir: query.sortDir,
      }),
    );
  }

  @Patch(':id')
  @ApiOperation({ summary: 'Update a <context>' })
  @ApiParam({ name: 'id', description: '<Context> UUID' })
  @ApiBody({ type: Update<Context>RequestDto })
  @ApiResponse({ status: 200, description: '<Context> updated' })
  @ApiResponse({ status: 404, description: '<Context> not found' })
  async update(
    @Param('id') id: string,
    @Body() dto: Update<Context>RequestDto,
    @CurrentOrganization() org: OrgContext,
  ): Promise<{ id: string }> {
    return this.commandBus.execute(
      new Update<Context>Command(id, org.id, dto.name, dto.description),
    );
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @Roles(['admin'])
  @ApiOperation({ summary: 'Delete a <context>' })
  @ApiParam({ name: 'id', description: '<Context> UUID' })
  @ApiResponse({ status: 204, description: 'Deleted' })
  @ApiResponse({ status: 404, description: '<Context> not found' })
  async remove(
    @Param('id') id: string,
    @CurrentOrganization() org: OrgContext,
  ): Promise<void> {
    await this.commandBus.execute(new Delete<Context>Command(id, org.id));
  }
}
```

## Controller with Plain Use Case (non-CQRS)

```typescript
// When the context uses plain use cases instead of CQRS:
import { Inject } from '@nestjs/common';
import {
  CREATE_<CONTEXT>_USE_CASE,
  type Create<Context>UseCase,
} from '../../application/use-cases/create-<context>.use-case';

@Controller('<contexts>')
export class <Context>Controller {
  constructor(
    @Inject(CREATE_<CONTEXT>_USE_CASE)
    private readonly createUseCase: Create<Context>UseCase,
  ) {}

  @Post()
  async create(
    @Body() dto: Create<Context>RequestDto,
    @CurrentOrganization() org: OrgContext,
  ): Promise<{ id: string }> {
    return this.createUseCase.execute({
      organizationId: org.id,
      name: dto.name,
    });
  }
}
```

## Presenter Pattern

When the use case output needs reshaping before sending to the client:

```typescript
// presentation/presenters/create-<context>.presenter.ts
import type { Create<Context>Command } from '../../application/commands/create-<context>.command';
import type { Create<Context>RequestDto } from '../dtos/create-<context>.http-dto';
import type { OrgContext } from '@/auth/decorators/current-organization.decorator';

export class Create<Context>Presenter {
  static toCommandInput(
    dto: Create<Context>RequestDto,
    org: OrgContext,
  ): ConstructorParameters<typeof Create<Context>Command>[0] {
    return {
      organizationId: org.id,
      name: dto.name.trim(),
    };
  }

  static toHttpResponse(output: { id: string; name: string }): Record<string, unknown> {
    return {
      id: output.id,
      name: output.name,
      createdAt: new Date().toISOString(),
    };
  }
}
```

## Controller Unit Test

```typescript
// controllers/__tests__/<context>.controller.spec.ts
import { describe, it, beforeEach, expect, vi } from 'vitest';
import { Test } from '@nestjs/testing';
import { CommandBus, QueryBus } from '@nestjs/cqrs';
import { <Context>Controller } from '../<context>.controller';

describe('<Context>Controller', () => {
  let controller: <Context>Controller;
  const commandBus = { execute: vi.fn() };
  const queryBus = { execute: vi.fn() };
  const org = { id: 'org-1', name: 'Test Org' };

  beforeEach(async () => {
    vi.clearAllMocks();
    const module = await Test.createTestingModule({
      controllers: [<Context>Controller],
      providers: [
        { provide: CommandBus, useValue: commandBus },
        { provide: QueryBus, useValue: queryBus },
      ],
    }).compile();

    controller = module.get(<Context>Controller);
  });

  it('create() dispatches Create<Context>Command and returns id', async () => {
    commandBus.execute.mockResolvedValue({ id: 'new-id' });

    const result = await controller.create({ name: 'My <Context>' }, org as any);

    expect(commandBus.execute).toHaveBeenCalledWith(
      expect.objectContaining({ name: 'My <Context>', organizationId: 'org-1' }),
    );
    expect(result).toStrictEqual({ id: 'new-id' });
  });

  it('findOne() throws NotFoundException when query returns null', async () => {
    queryBus.execute.mockResolvedValue(null);

    await expect(controller.findOne('non-existent', org as any)).rejects.toThrow(
      'not found',
    );
  });

  it('remove() dispatches Delete<Context>Command', async () => {
    commandBus.execute.mockResolvedValue(undefined);

    await controller.remove('entity-id', org as any);

    expect(commandBus.execute).toHaveBeenCalledWith(
      expect.objectContaining({ id: 'entity-id', organizationId: 'org-1' }),
    );
  });
});
```

## Pagination Response Format

```typescript
// Standard paginated response shape:
{
  items: <Context>View[];
  pagination: {
    page: number;
    perPage: number;
    total: number;
    lastPage: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

// Computed from SearchResult:
function toPaginationMeta(result: SearchResult<any>): PaginationMeta {
  return {
    page: result.currentPage,
    perPage: result.perPage,
    total: result.total,
    lastPage: result.lastPage,
    hasNext: result.currentPage < result.lastPage,
    hasPrev: result.currentPage > 1,
  };
}
```
