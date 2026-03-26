---
name: review-subdomain
description: Reviews a NestJS bounded context implementation for Hexagonal Architecture + DDD + CQRS compliance. Checks 6 dimensions — domain purity, application patterns, infrastructure isolation, presentation concerns, testing coverage, and module organization. Produces a structured Pass/Warning/Fail report.
argument-hint: Path to the bounded context directory (e.g., "src/enterprise/billing/invoices")
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash
  - Agent
---

# review-subdomain

Review a bounded context for architectural compliance. Accept the BC path as the argument (e.g., `src/enterprise/billing/invoices`). Run each dimension in order. Produce a structured report at the end.

The rubric with scoring criteria is in `references/review-rubric.md`.

---

## Setup

Resolve the target directory from the argument. If no argument is given, ask the user for the path.

Verify the directory exists:

```bash
ls <bc-path>
```

If the directory does not exist, stop and report an error.

Set `BC_PATH` to the resolved absolute path. All grep and glob searches below use `BC_PATH` as the root.

---

## Dimension 1 — Domain Purity

**Purpose:** Confirm the domain layer has zero framework or infrastructure dependencies.

Run the following checks against `<BC_PATH>/domain/`:

**Check D1 — No NestJS common imports in domain:**
```bash
grep -r "@nestjs/common" <BC_PATH>/domain/ --include="*.ts" -l
```
FAIL if any files are returned. Exception: `@nestjs/cqrs` is allowed (for `AggregateRoot`, `IEvent`).

**Check D2 — No class-validator in VOs:**
```bash
grep -r "class-validator" <BC_PATH>/domain/value-objects/ --include="*.ts" -l
```
FAIL if any files are returned. `class-validator` is allowed in `domain/validators/` only.

**Check D3 — No Prisma imports in domain:**
```bash
grep -r "PrismaClient\|PrismaService\|@prisma/client" <BC_PATH>/domain/ --include="*.ts" -l
```
FAIL if any files are returned.

**Check D4 — Entity uses apply() not addDomainEvent():**
```bash
grep -r "addDomainEvent\|pullDomainEvents" <BC_PATH>/domain/ --include="*.ts" -l
```
FAIL if any files are returned. Entities must use `this.apply(event)`.

**Check D5 — Entity has create() and restore() factories:**

For each entity file found in `<BC_PATH>/domain/entities/`:
- Read the file
- Confirm `static create(` is present
- Confirm `static restore(` is present
- Confirm `private constructor` is present

FAIL if any entity is missing one of these.

**Check D6 — Repository interface is a pure interface (no @Injectable):**
```bash
grep -r "@Injectable" <BC_PATH>/domain/repositories/ --include="*.ts" -l
```
FAIL if any files are returned.

**Check D7 — Data builders exist:**
```bash
ls <BC_PATH>/domain/testing/helpers/
```
WARNING if the directory is empty or does not exist.

---

## Dimension 2 — Application Patterns

**Purpose:** Confirm the application layer follows the chosen pattern correctly and does not hold framework concerns.

**Check A1 — EventPublisher not in use cases:**
```bash
grep -r "EventPublisher" <BC_PATH>/application/ --include="*.ts" -l
```
FAIL if `EventPublisher` appears in `application/usecases/` or `application/dtos/`. It is allowed in `application/commands/` (Pattern B/C handlers).

**Check A2 — No class-validator in application DTOs:**
```bash
grep -r "IsString\|IsNotEmpty\|IsEmail\|IsOptional\|IsUUID\|IsEnum\|IsNumber\|IsBoolean\|IsArray\|ValidateNested" <BC_PATH>/application/ --include="*.ts" -l
```
FAIL if any files are returned. `class-validator` belongs only in presentation request DTOs.

**Check A3 — Pattern A use cases have no @Injectable:**

Find files matching `<BC_PATH>/application/usecases/*.usecase.ts`:
```bash
grep -r "@Injectable" <BC_PATH>/application/usecases/ --include="*.ts" -l
```
FAIL if `@Injectable` appears in use case files.

**Check A4 — Write handlers return void or { id: string }:**

Read each command handler in `<BC_PATH>/application/commands/`:
- Check the return type annotation on `execute()`
- FAIL if return type is an entity class or a full output DTO with many fields
- PASS if return type is `void`, `Promise<void>`, `{ id: string }`, or `Promise<{ id: string }>`

**Check A5 — commit() called in handlers, not in use cases:**
```bash
grep -r "\.commit()" <BC_PATH>/application/usecases/ --include="*.ts" -l
```
FAIL if `.commit()` appears in use case files.

**Check A6 — Ports defined in application/ports/ with TOKEN symbol:**

Find files in `<BC_PATH>/application/ports/`:
- Each port file should export a `Symbol` token
```bash
grep -r "Symbol(" <BC_PATH>/application/ports/ --include="*.ts" -l
```
WARNING if port files exist without a `Symbol(` token export.

---

## Dimension 3 — Infrastructure Isolation

**Purpose:** Confirm infrastructure wires domain to the outside world without leaking domain logic.

**Check I1 — Module exports only tokens:**

Read `<BC_PATH>/infrastructure/*.module.ts`:
- Find the `exports:` array
- FAIL if any class name (not a Symbol constant) appears in the exports array
- PASS if exports contains only token constants (e.g., `<NAME>_REPOSITORY`, `<DEPENDENCY>_PORT_TOKEN`)

**Check I2 — Repository has no event dispatch:**
```bash
grep -r "\.commit()\|EventBus\|EventPublisher\|publish(" <BC_PATH>/infrastructure/database/ --include="*.ts" -l
```
FAIL if any of these appear in repository files.

**Check I3 — Mapper uses restore() not create():**
```bash
grep -r "Entity\.create\|\.create(" <BC_PATH>/infrastructure/database/prisma/models/ --include="*.ts" -l
```
FAIL if entity `create()` is called inside a mapper. Mappers must call `restore()`.

**Check I4 — In-memory repository exists:**
```bash
ls <BC_PATH>/infrastructure/database/in-memory/repositories/
```
WARNING if directory is empty or missing.

**Check I5 — Event handlers use @EventsHandler not @OnEvent:**
```bash
grep -r "@OnEvent" <BC_PATH>/infrastructure/listeners/ --include="*.ts" -l
```
FAIL if `@OnEvent` is used in listeners. All new domain event handlers must use `@EventsHandler`.

**Check I6 — Event handlers have try/catch:**

Read each file in `<BC_PATH>/infrastructure/listeners/`:
- FAIL if `handle()` method does not contain a `try {` block

**Check I7 — organizationId (or tenant id) in all Prisma queries:**

Read each Prisma repository file in `<BC_PATH>/infrastructure/database/prisma/repositories/`:
- Check `search()` has `organizationId` (or tenant field) in `whereClause`
- WARNING if `search()` does not scope by tenant

---

## Dimension 4 — Presentation Concerns

**Purpose:** Confirm controllers are thin HTTP adapters and request DTOs hold all input validation.

**Check P1 — class-validator used only in request DTOs:**
```bash
grep -r "IsString\|IsNotEmpty\|IsEmail\|IsOptional\|IsUUID\|IsEnum\|ValidateNested" <BC_PATH>/infrastructure/controllers/dtos/ --include="*.ts" -l
```
This should return files. If no files returned: WARNING (validation may be missing).
```bash
grep -r "IsString\|IsNotEmpty\|IsEmail\|IsOptional\|IsUUID\|IsEnum\|ValidateNested" <BC_PATH>/infrastructure/controllers/ --include="*.ts" -l
```
Cross-check: all results should be inside `dtos/`, not in the controller itself.

**Check P2 — organizationId not taken from request body:**
```bash
grep -r "body\.organizationId\|dto\.organizationId\|req\.body.*organizationId" <BC_PATH>/infrastructure/controllers/ --include="*.ts" -l
```
FAIL if `organizationId` is read from the request body in a controller. Must come from `@CurrentOrganization()`.

**Check P3 — Swagger decorators present:**
```bash
grep -r "@ApiOperation\|@ApiResponse\|@ApiTags" <BC_PATH>/infrastructure/controllers/ --include="*.ts" -l
```
WARNING if no Swagger decorators found in controller files.

**Check P4 — Guards applied:**
```bash
grep -r "@UseGuards\|@ApiBearerAuth" <BC_PATH>/infrastructure/controllers/ --include="*.ts" -l
```
WARNING if no guards found on any controller.

**Check P5 — No business logic in controllers:**

Read each controller file. Flag as FAIL if any of these patterns appear directly in a controller method (not in a called service):
- Direct repository calls (`this.repository.findById`)
- Domain entity instantiation (`XxxEntity.create`)
- Business rule conditions beyond simple null checks

---

## Dimension 5 — Testing Coverage

**Purpose:** Confirm test files exist and follow the test-first structure.

**Check T1 — Entity spec files exist:**
```bash
find <BC_PATH>/domain/entities/__tests__ -name "*.spec.ts" 2>/dev/null
```
WARNING if no spec files found.

**Check T2 — VO spec files exist:**
```bash
find <BC_PATH>/domain/value-objects/__tests__ -name "*.spec.ts" 2>/dev/null
```
WARNING if VO files exist but no specs.

**Check T3 — Application spec files exist:**
```bash
find <BC_PATH>/application -name "*.spec.ts" 2>/dev/null
```
WARNING if application layer has no specs.

**Check T4 — Controller spec files exist:**
```bash
find <BC_PATH>/infrastructure/controllers/__tests__ -name "*.spec.ts" 2>/dev/null
```
WARNING if controllers exist but no specs.

**Check T5 — In-memory repository used in application tests:**
```bash
grep -r "InMemoryRepository" <BC_PATH>/application --include="*.spec.ts" -l
```
WARNING if application specs exist but none use the in-memory repository.

---

## Dimension 6 — Module Organization

**Purpose:** Confirm the directory structure and naming conventions are correct.

**Check M1 — Standard directories present:**

Verify these paths exist under `BC_PATH`:
- `domain/entities/`
- `domain/repositories/`
- `domain/events/`
- `application/`
- `infrastructure/`

WARNING for each missing standard directory.

**Check M2 — Module file exists:**
```bash
find <BC_PATH>/infrastructure -maxdepth 1 -name "*.module.ts" 2>/dev/null
```
FAIL if no module file found.

**Check M3 — No cross-layer imports (domain importing infrastructure):**
```bash
grep -r "from.*infrastructure\|require.*infrastructure" <BC_PATH>/domain/ --include="*.ts" -l
grep -r "from.*infrastructure\|require.*infrastructure" <BC_PATH>/application/ --include="*.ts" -l
```
FAIL if domain or application imports from infrastructure.

**Check M4 — File naming conventions:**

Spot-check a few files:
- Entity files: `<name>.entity.ts`
- VO files: `<name>.vo.ts`
- Event files: `<name>-<verb>ed.event.ts`
- Handler files: `<name>.handler.ts` or `<name>-<verb>ed.handler.ts`
- Repository files: `<name>.repository.ts`, `prisma-<name>.repository.ts`

WARNING if files deviate significantly from these naming conventions.

---

## Report Format

Produce a structured markdown report after all checks:

```
# Architecture Review: <BC_PATH>

## Summary

| Dimension | Status |
|---|---|
| Domain Purity | PASS / FAIL |
| Application Patterns | PASS / FAIL |
| Infrastructure Isolation | PASS / FAIL |
| Presentation Concerns | PASS / WARNING |
| Testing Coverage | PASS / WARNING |
| Module Organization | PASS / FAIL |

Overall: PASS / NEEDS WORK

---

## Findings

### FAIL Items (must fix before merge)

- **[D1] No NestJS common imports in domain:** Found in `domain/entities/order.entity.ts` (line 3: `import { Injectable }`)
  Fix: Remove `@Injectable` — entities have no framework decorators.

- **[I1] Module exports only tokens:** `PrismaOrderRepository` exported from `orders.module.ts`
  Fix: Change to `exports: [ORDER_REPOSITORY]`.

### WARNING Items (recommended improvements)

- **[T1] Entity spec files:** No spec files found in `domain/entities/__tests__/`
  Recommendation: Add entity unit tests covering `create()`, `restore()`, and each mutating method.

- **[D7] Data builders:** `domain/testing/helpers/` is empty
  Recommendation: Add `OrderDataBuilder` with faker defaults for use in all test files.

### PASS Items

- Domain purity: no framework imports in domain
- Application DTOs: no class-validator found
- Mapper uses restore(): confirmed in `order-model.mapper.ts`
- Module exports: only token Symbols exported
```

---

## After the Report

- FAIL items are blocking — the BC is not ready to merge until all FAILs are resolved.
- WARNING items are advisory — present them to the user and ask whether to address now or log as tech debt.
- If all items are PASS or WARNING: report the BC as architecture-compliant.

Suggest specific fixes for each FAIL item, referencing the relevant layer skill (`nestjs-hexagonal:domain`, `nestjs-hexagonal:application`, etc.) for implementation guidance.
