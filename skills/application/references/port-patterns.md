# Port Patterns — Cross-Module Communication

A port is an interface that allows one module to depend on another without
creating a direct import. The **consumer** module defines the interface; the
**provider** module supplies the adapter.

This follows the Dependency Inversion Principle: the consumer owns the contract,
the provider conforms to it.

---

## Ownership rule

```
Consumer module (needs data)         Provider module (has data)
└── application/ports/               └── infrastructure/adapters/
    └── company.port.ts                  └── company-port.adapter.ts
        interface CompanyPort                implements CompanyPort
        COMPANY_PORT_TOKEN
```

The adapter lives in the provider module's `infrastructure/adapters/` — not in the
consumer module.

---

## Port Interface + TOKEN

Both the interface and its token live in the same file in the consumer module:

```typescript
// <consumer-module>/application/ports/company.port.ts

export const COMPANY_PORT_TOKEN = Symbol('CompanyPort');

export interface CompanyPort {
  findById(id: string): Promise<CompanyData | null>;
  isActive(id: string): Promise<boolean>;
}

export interface CompanyData {
  id: string;
  name: string;
  isActive: boolean;
  withdrawalFee?: { percentage: number; fixed: number };
}
```

---

## Adapter Implementation

The adapter lives in the provider module. It implements the interface from the
consumer module:

```typescript
// <provider-module>/infrastructure/adapters/company-port.adapter.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@/shared/infrastructure/database/prisma.service';

import type { CompanyPort, CompanyData } from
  '<consumer-module>/application/ports/company.port';

@Injectable()
export class CompanyPortAdapter implements CompanyPort {
  constructor(private readonly prisma: PrismaService) {}

  async findById(id: string): Promise<CompanyData | null> {
    const record = await this.prisma.company.findUnique({ where: { id } });
    if (!record) return null;
    return {
      id: record.id,
      name: record.name,
      isActive: record.isActive,
      withdrawalFee: record.withdrawalFeePercentage
        ? {
            percentage: record.withdrawalFeePercentage,
            fixed: record.withdrawalFeeFixed ?? 0,
          }
        : undefined,
    };
  }

  async isActive(id: string): Promise<boolean> {
    const record = await this.prisma.company.findUnique({
      where: { id },
      select: { isActive: true },
    });
    return record?.isActive ?? false;
  }
}
```

---

## Provider Module Registration

```typescript
// <provider-module>/infrastructure/<provider>.module.ts
import { Module } from '@nestjs/common';
import { COMPANY_PORT_TOKEN } from '<consumer-module>/application/ports/company.port';
import { CompanyPortAdapter } from './adapters/company-port.adapter';

@Module({
  providers: [
    CompanyPortAdapter,
    {
      provide: COMPANY_PORT_TOKEN,
      useExisting: CompanyPortAdapter,
    },
  ],
  exports: [COMPANY_PORT_TOKEN], // export the token so consumer module can import it
})
export class CompanyModule {}
```

---

## Consumer Module Import

```typescript
// <consumer-module>/infrastructure/withdrawal-requests.module.ts
import { Module } from '@nestjs/common';
import { CompanyModule } from '<provider-module>/infrastructure/company.module';
import { COMPANY_PORT_TOKEN } from '../application/ports/company.port';
import { CreateWithdrawalRequestHandler } from '../application/commands/create-withdrawal-request.handler';

@Module({
  imports: [CompanyModule], // imports the module that exports COMPANY_PORT_TOKEN
  providers: [CreateWithdrawalRequestHandler],
})
export class WithdrawalRequestsModule {}
```

---

## Injection in Handler or UseCase

In a handler (Pattern B/C):

```typescript
@CommandHandler(CreateWithdrawalRequestCommand)
export class CreateWithdrawalRequestHandler
  implements ICommandHandler<CreateWithdrawalRequestCommand>
{
  constructor(
    @Inject(COMPANY_PORT_TOKEN)
    private readonly companyPort: CompanyPort,
    // ...
  ) {}
}
```

In a use case (Pattern A), the port is passed via `useFactory` in the module:

```typescript
{
  provide: CREATE_WITHDRAWAL_REQUEST_USE_CASE_TOKEN,
  useFactory: (
    repo: WithdrawalRequestRepository.Repository,
    companyPort: CompanyPort,
  ) => new CreateWithdrawalRequestUseCase.UseCase(repo, companyPort),
  inject: [WITHDRAWAL_REQUEST_REPOSITORY_TOKEN, COMPANY_PORT_TOKEN],
},
```

---

## Common Port Patterns

### Query port (read-only data from another module)

```typescript
// application/ports/transaction-query.port.ts
export const TRANSACTION_QUERY_PORT_TOKEN = Symbol('TransactionQueryPort');

export interface TransactionSearchParams {
  companyId: string;
  customerId: string;
  page?: number;
  perPage?: number;
}

export interface TransactionSearchResult {
  items: TransactionData[];
  total: number;
  currentPage: number;
  perPage: number;
  lastPage: number;
}

export interface TransactionData {
  id: string;
  amount: number;
  status: string;
  createdAt: Date | null;
}

export interface TransactionQueryPort {
  searchByCustomer(params: TransactionSearchParams): Promise<TransactionSearchResult>;
  findById(id: string, companyId: string): Promise<TransactionData | null>;
}
```

### Notification port (fire-and-forget side effect)

```typescript
// application/ports/email-notification.port.ts
export const EMAIL_NOTIFICATION_PORT_TOKEN = Symbol('EmailNotificationPort');

export interface EmailNotificationPort {
  sendWithdrawalCreatedEmail(
    to: string,
    userName: string,
    withdrawalId: string,
    amount: number,
    requestedAt: Date,
  ): Promise<void>;

  sendWithdrawalApprovedEmail(
    to: string,
    userName: string,
    withdrawalId: string,
    amount: number,
    reviewedAt: Date,
  ): Promise<void>;
}
```

### User query port (auth/identity data)

```typescript
// application/ports/user-query.port.ts
export const USER_QUERY_PORT_TOKEN = Symbol('UserQueryPort');

export interface UserData {
  id: string;
  email: string;
  name: string;
  companyId: string;
  role: string;
}

export interface UserQueryPort {
  findById(id: string): Promise<UserData | null>;
  findByEmail(email: string): Promise<UserData | null>;
}
```

---

## Port vs. Repository

| Port | Repository |
|------|-----------|
| Cross-module communication | Within the same module |
| Interface owned by consumer | Interface owned by domain |
| Adapter in provider's `infrastructure/` | Implementation in same module's `infrastructure/` |
| Can expose any data shape | Exposes domain entities |
| TOKEN exported from `application/ports/` | TOKEN exported from `domain/repositories/` |

---

## Testing Ports

Mock the port in isolation using a simple object:

```typescript
const mockCompanyPort: CompanyPort = {
  findById: vi.fn().mockResolvedValue({
    id: 'company-1',
    name: 'Acme Corp',
    isActive: true,
  }),
  isActive: vi.fn().mockResolvedValue(true),
};
```

For adapter tests, use the real Prisma client with a test database or
use `prismaMock` from vitest-mock-extended.

---

## Checklist

- [ ] Port interface defined in **consumer** module's `application/ports/`.
- [ ] TOKEN symbol in the same file as the interface.
- [ ] Adapter implements the interface from the consumer module.
- [ ] Adapter lives in **provider** module's `infrastructure/adapters/`.
- [ ] Provider module exports the TOKEN (not the adapter class).
- [ ] Consumer module imports the provider module (not a circular dependency).
- [ ] No direct Prisma / infrastructure imports crossing module boundaries.
- [ ] Unit tests mock the port interface, not the adapter.
