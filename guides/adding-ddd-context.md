# Adding a DDD Bounded Context

**Last Updated**: November 17, 2025  
**Difficulty**: Advanced  
**Time**: 2-4 hours

---

## Overview

This guide shows how to add a new **DDD bounded context** to SkillCore. A bounded context represents a cohesive subdomain with its own **models**, **business logic**, **events**, and **API**.

---

## When to Create a New Context

Create a new bounded context when:

✅ **Distinct Business Domain**: Has unique business rules and terminology  
✅ **Clear Boundaries**: Limited coupling with other contexts  
✅ **Team Ownership**: Can be owned by a specific team  
✅ **Independent Evolution**: Can evolve independently  

**Examples**: Gradebook, Attendance, IEP Management, Library System

❌ **Don't create for**:
- Simple CRUD operations
- UI-only features
- Cross-cutting concerns (use shared modules)

---

## Step-by-Step Guide

### Step 1: Create Context Folder Structure

```bash
cd api/src/contexts
mkdir -p student-portfolio/{domain,application,infrastructure,presentation}
cd student-portfolio

# Create domain layer folders
mkdir -p domain/{entities,value-objects,aggregates,services,events,repositories}

# Create application layer folders
mkdir -p application/{use-cases,services,event-handlers,dto}

# Create infrastructure layer folders
mkdir -p infrastructure/{persistence,di}

# Create presentation layer folders
mkdir -p presentation/{graphql,rest}
```

**Final Structure**:
```
api/src/contexts/student-portfolio/
├── domain/
│   ├── entities/           # Core business entities
│   ├── value-objects/      # Value objects (immutable)
│   ├── aggregates/         # Aggregate roots
│   ├── services/           # Domain services
│   ├── events/             # Domain events
│   └── repositories/       # Repository interfaces
├── application/
│   ├── use-cases/          # Application use cases
│   ├── services/           # Application services
│   ├── event-handlers/     # Event handlers
│   └── dto/                # Data transfer objects
├── infrastructure/
│   ├── persistence/        # Prisma repositories
│   └── di/                 # Dependency injection
└── presentation/
    ├── graphql/            # GraphQL schema & resolvers
    └── rest/               # REST endpoints (if needed)
```

---

### Step 2: Define Domain Entities

**Location**: `domain/entities/Portfolio.ts`

```typescript
import { Entity } from '@/shared/domain/Entity'
import { PortfolioId } from '../value-objects/PortfolioId'
import { PortfolioItem } from './PortfolioItem'

export interface PortfolioProps {
  studentId: string
  title: string
  description?: string
  items: PortfolioItem[]
  isPublic: boolean
  createdAt: Date
  updatedAt: Date
}

export class Portfolio extends Entity<PortfolioProps> {
  private constructor(props: PortfolioProps, id: PortfolioId) {
    super(props, id)
  }
  
  // Factory method
  public static create(
    props: Omit<PortfolioProps, 'items' | 'createdAt' | 'updatedAt'>
  ): Portfolio {
    return new Portfolio(
      {
        ...props,
        items: [],
        createdAt: new Date(),
        updatedAt: new Date(),
      },
      PortfolioId.create()
    )
  }
  
  // Business logic methods
  public addItem(item: PortfolioItem): void {
    if (this.props.items.length >= 50) {
      throw new Error('Portfolio cannot have more than 50 items')
    }
    
    this.props.items.push(item)
    this.props.updatedAt = new Date()
  }
  
  public removeItem(itemId: string): void {
    const index = this.props.items.findIndex(item => item.id.toString() === itemId)
    if (index === -1) {
      throw new Error('Portfolio item not found')
    }
    
    this.props.items.splice(index, 1)
    this.props.updatedAt = new Date()
  }
  
  public makePublic(): void {
    this.props.isPublic = true
    this.props.updatedAt = new Date()
  }
  
  public makePrivate(): void {
    this.props.isPublic = false
    this.props.updatedAt = new Date()
  }
  
  // Getters
  get studentId(): string {
    return this.props.studentId
  }
  
  get title(): string {
    return this.props.title
  }
  
  get items(): PortfolioItem[] {
    return this.props.items
  }
  
  get isPublic(): boolean {
    return this.props.isPublic
  }
}
```

---

### Step 3: Define Value Objects

**Location**: `domain/value-objects/PortfolioId.ts`

```typescript
import { ValueObject } from '@/shared/domain/ValueObject'
import { v4 as uuid } from 'uuid'

interface PortfolioIdProps {
  value: string
}

export class PortfolioId extends ValueObject<PortfolioIdProps> {
  private constructor(props: PortfolioIdProps) {
    super(props)
  }
  
  public static create(id?: string): PortfolioId {
    return new PortfolioId({
      value: id || uuid(),
    })
  }
  
  public toString(): string {
    return this.props.value
  }
}
```

---

### Step 4: Define Domain Events

**Location**: `domain/events/PortfolioCreatedEvent.ts`

```typescript
import { DomainEvent } from '@/shared/domain/events/DomainEvent'

export interface PortfolioCreatedEventPayload {
  portfolioId: string
  studentId: string
  title: string
  districtId: string
  schoolId: string
}

export class PortfolioCreatedEvent extends DomainEvent<PortfolioCreatedEventPayload> {
  constructor(payload: PortfolioCreatedEventPayload) {
    super({
      eventType: 'portfolio.created',
      aggregateId: payload.portfolioId,
      aggregateType: 'Portfolio',
      payload,
      metadata: {
        districtId: payload.districtId,
        schoolId: payload.schoolId,
      },
    })
  }
}
```

---

### Step 5: Define Repository Interface

**Location**: `domain/repositories/IPortfolioRepository.ts`

```typescript
import { Portfolio } from '../entities/Portfolio'
import { PortfolioId } from '../value-objects/PortfolioId'

export interface IPortfolioRepository {
  save(portfolio: Portfolio): Promise<void>
  findById(id: PortfolioId): Promise<Portfolio | null>
  findByStudentId(studentId: string): Promise<Portfolio[]>
  delete(id: PortfolioId): Promise<void>
}
```

---

### Step 6: Implement Prisma Schema

**Location**: `api/prisma/schema.prisma`

```prisma
model Portfolio {
  id          String   @id @default(uuid())
  sourcedId   String   @unique
  
  // Relationships
  studentId   String
  student     Student  @relation(fields: [studentId], references: [id])
  
  // Content
  title       String
  description String?  @db.Text
  isPublic    Boolean  @default(false)
  
  // Items
  items       PortfolioItem[]
  
  // Multi-tenant
  districtId  String
  schoolId    String
  
  // Soft delete
  deletedAt   DateTime?
  
  // Timestamps
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@index([studentId])
  @@index([districtId])
  @@index([schoolId])
  @@index([deletedAt])
}

model PortfolioItem {
  id           String    @id @default(uuid())
  
  // Portfolio
  portfolioId  String
  portfolio    Portfolio @relation(fields: [portfolioId], references: [id])
  
  // Content
  title        String
  description  String?   @db.Text
  itemType     String    // 'work_sample', 'reflection', 'achievement', 'project'
  fileUrl      String?
  thumbnailUrl String?
  
  // Metadata
  order        Int       @default(0)
  
  // Multi-tenant
  districtId   String
  
  // Soft delete
  deletedAt    DateTime?
  
  // Timestamps
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
  
  @@index([portfolioId])
  @@index([districtId])
}
```

**Generate Migration**:
```bash
cd api
pnpm prisma migrate dev --name add_student_portfolio
```

---

### Step 7: Implement Repository

**Location**: `infrastructure/persistence/PrismaPortfolioRepository.ts`

```typescript
import { PrismaClient } from '@prisma/client'
import { IPortfolioRepository } from '../../domain/repositories/IPortfolioRepository'
import { Portfolio } from '../../domain/entities/Portfolio'
import { PortfolioId } from '../../domain/value-objects/PortfolioId'
import { PortfolioMapper } from './mappers/PortfolioMapper'

export class PrismaPortfolioRepository implements IPortfolioRepository {
  constructor(private readonly prisma: PrismaClient) {}
  
  async save(portfolio: Portfolio): Promise<void> {
    const data = PortfolioMapper.toPersistence(portfolio)
    
    await this.prisma.portfolio.upsert({
      where: { id: data.id },
      create: data,
      update: data,
    })
  }
  
  async findById(id: PortfolioId): Promise<Portfolio | null> {
    const portfolioData = await this.prisma.portfolio.findUnique({
      where: { id: id.toString() },
      include: { items: true },
    })
    
    if (!portfolioData) return null
    
    return PortfolioMapper.toDomain(portfolioData)
  }
  
  async findByStudentId(studentId: string): Promise<Portfolio[]> {
    const portfolios = await this.prisma.portfolio.findMany({
      where: { 
        studentId,
        deletedAt: null,
      },
      include: { items: true },
      orderBy: { createdAt: 'desc' },
    })
    
    return portfolios.map(PortfolioMapper.toDomain)
  }
  
  async delete(id: PortfolioId): Promise<void> {
    await this.prisma.portfolio.update({
      where: { id: id.toString() },
      data: { deletedAt: new Date() },
    })
  }
}
```

---

### Step 8: Create Use Cases

**Location**: `application/use-cases/CreatePortfolio.ts`

```typescript
import { IPortfolioRepository } from '../../domain/repositories/IPortfolioRepository'
import { Portfolio } from '../../domain/entities/Portfolio'
import { EventBus } from '@/shared/infrastructure/events/EventBus'
import { PortfolioCreatedEvent } from '../../domain/events/PortfolioCreatedEvent'

export interface CreatePortfolioInput {
  studentId: string
  title: string
  description?: string
  districtId: string
  schoolId: string
}

export class CreatePortfolioUseCase {
  constructor(
    private readonly repository: IPortfolioRepository,
    private readonly eventBus: EventBus
  ) {}
  
  async execute(input: CreatePortfolioInput): Promise<Portfolio> {
    // 1. Create domain entity
    const portfolio = Portfolio.create({
      studentId: input.studentId,
      title: input.title,
      description: input.description,
      isPublic: false,
    })
    
    // 2. Persist
    await this.repository.save(portfolio)
    
    // 3. Publish domain event
    const event = new PortfolioCreatedEvent({
      portfolioId: portfolio.id.toString(),
      studentId: input.studentId,
      title: input.title,
      districtId: input.districtId,
      schoolId: input.schoolId,
    })
    
    await this.eventBus.publish(event)
    
    return portfolio
  }
}
```

---

### Step 9: Create GraphQL Schema

**Location**: `presentation/graphql/portfolio.schema.ts`

```typescript
export const portfolioTypeDefs = `#graphql
  extend type Query {
    portfolio(id: ID!): Portfolio
    studentPortfolios(studentId: ID!): [Portfolio!]!
  }
  
  extend type Mutation {
    createPortfolio(input: CreatePortfolioInput!): CreatePortfolioPayload!
    addPortfolioItem(input: AddPortfolioItemInput!): AddPortfolioItemPayload!
  }
  
  type Portfolio {
    id: ID!
    studentId: ID!
    student: Student!
    title: String!
    description: String
    items: [PortfolioItem!]!
    isPublic: Boolean!
    createdAt: DateTime!
    updatedAt: DateTime!
  }
  
  type PortfolioItem {
    id: ID!
    title: String!
    description: String
    itemType: PortfolioItemType!
    fileUrl: String
    thumbnailUrl: String
    order: Int!
    createdAt: DateTime!
  }
  
  enum PortfolioItemType {
    WORK_SAMPLE
    REFLECTION
    ACHIEVEMENT
    PROJECT
  }
  
  input CreatePortfolioInput {
    studentId: ID!
    title: String!
    description: String
  }
  
  type CreatePortfolioPayload {
    portfolio: Portfolio
    errors: [MutationError!]
  }
  
  input AddPortfolioItemInput {
    portfolioId: ID!
    title: String!
    description: String
    itemType: PortfolioItemType!
    fileUrl: String
  }
  
  type AddPortfolioItemPayload {
    portfolioItem: PortfolioItem
    errors: [MutationError!]
  }
`
```

---

### Step 10: Create GraphQL Resolvers

**Location**: `presentation/graphql/portfolio.resolvers.ts`

```typescript
import { CreatePortfolioUseCase } from '../../application/use-cases/CreatePortfolio'
import { GetPortfolioUseCase } from '../../application/use-cases/GetPortfolio'

export const portfolioResolvers = {
  Query: {
    portfolio: async (_: any, { id }: { id: string }, context: any) => {
      const useCase = context.container.resolve<GetPortfolioUseCase>('GetPortfolioUseCase')
      return useCase.execute(id)
    },
    
    studentPortfolios: async (_: any, { studentId }: { studentId: string }, context: any) => {
      const useCase = context.container.resolve<GetStudentPortfoliosUseCase>('GetStudentPortfoliosUseCase')
      return useCase.execute(studentId)
    },
  },
  
  Mutation: {
    createPortfolio: async (_: any, { input }: any, context: any) => {
      try {
        const useCase = context.container.resolve<CreatePortfolioUseCase>('CreatePortfolioUseCase')
        const portfolio = await useCase.execute({
          ...input,
          districtId: context.user.districtId,
          schoolId: context.user.schoolId,
        })
        
        return { portfolio, errors: null }
      } catch (error) {
        return {
          portfolio: null,
          errors: [{ message: error.message }],
        }
      }
    },
  },
}
```

---

### Step 11: Register Context in DI Container

**Location**: `infrastructure/di/container.ts`

```typescript
import { Container } from '@/shared/infrastructure/di/Container'
import { PrismaClient } from '@prisma/client'
import { EventBus } from '@/shared/infrastructure/events/EventBus'

// Repositories
import { PrismaPortfolioRepository } from '../persistence/PrismaPortfolioRepository'

// Use Cases
import { CreatePortfolioUseCase } from '../../application/use-cases/CreatePortfolio'
import { GetPortfolioUseCase } from '../../application/use-cases/GetPortfolio'

export function registerPortfolioContext(container: Container) {
  const prisma = container.resolve<PrismaClient>('PrismaClient')
  const eventBus = container.resolve<EventBus>('EventBus')
  
  // Register repositories
  const portfolioRepository = new PrismaPortfolioRepository(prisma)
  container.register('PortfolioRepository', portfolioRepository)
  
  // Register use cases
  container.register(
    'CreatePortfolioUseCase',
    new CreatePortfolioUseCase(portfolioRepository, eventBus)
  )
  container.register(
    'GetPortfolioUseCase',
    new GetPortfolioUseCase(portfolioRepository)
  )
  
  console.log('✓ Portfolio context registered')
}
```

---

### Step 12: Register GraphQL Schema

**Location**: `api/src/graphql/schema.ts`

```typescript
import { portfolioTypeDefs } from '@/contexts/student-portfolio/presentation/graphql/portfolio.schema'
import { portfolioResolvers } from '@/contexts/student-portfolio/presentation/graphql/portfolio.resolvers'

export const typeDefs = `#graphql
  # ... existing schemas
  ${portfolioTypeDefs}
`

export const resolvers = mergeResolvers([
  // ... existing resolvers
  portfolioResolvers,
])
```

---

### Step 13: Write Tests

**Location**: `application/use-cases/__tests__/CreatePortfolio.test.ts`

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { CreatePortfolioUseCase } from '../CreatePortfolio'
import { InMemoryPortfolioRepository } from '../../../infrastructure/persistence/InMemoryPortfolioRepository'
import { InMemoryEventBus } from '@/shared/infrastructure/events/InMemoryEventBus'

describe('CreatePortfolioUseCase', () => {
  let useCase: CreatePortfolioUseCase
  let repository: InMemoryPortfolioRepository
  let eventBus: InMemoryEventBus
  
  beforeEach(() => {
    repository = new InMemoryPortfolioRepository()
    eventBus = new InMemoryEventBus()
    useCase = new CreatePortfolioUseCase(repository, eventBus)
  })
  
  it('should create a portfolio', async () => {
    const input = {
      studentId: 'student-123',
      title: 'My Portfolio',
      description: 'A collection of my work',
      districtId: 'district-456',
      schoolId: 'school-789',
    }
    
    const portfolio = await useCase.execute(input)
    
    expect(portfolio.studentId).toBe('student-123')
    expect(portfolio.title).toBe('My Portfolio')
    expect(portfolio.isPublic).toBe(false)
    expect(portfolio.items).toHaveLength(0)
  })
  
  it('should publish PortfolioCreatedEvent', async () => {
    const input = {
      studentId: 'student-123',
      title: 'My Portfolio',
      districtId: 'district-456',
      schoolId: 'school-789',
    }
    
    await useCase.execute(input)
    
    const events = eventBus.getPublishedEvents()
    expect(events).toHaveLength(1)
    expect(events[0].eventType).toBe('portfolio.created')
  })
})
```

---

## Context Integration Checklist

- [ ] Prisma schema updated
- [ ] Migration created and applied
- [ ] Domain entities defined
- [ ] Value objects created
- [ ] Domain events defined
- [ ] Repository interface created
- [ ] Repository implementation created
- [ ] Use cases implemented
- [ ] GraphQL schema created
- [ ] GraphQL resolvers created
- [ ] DI container configured
- [ ] Event handlers registered
- [ ] Unit tests written
- [ ] Integration tests written
- [ ] Documentation updated

---

## Best Practices

### 1. Domain Purity

Keep domain layer **pure** (no external dependencies):

✅ **Good**:
```typescript
// domain/entities/Portfolio.ts
export class Portfolio extends Entity {
  addItem(item: PortfolioItem): void {
    if (this.props.items.length >= 50) {
      throw new DomainException('Portfolio cannot exceed 50 items')
    }
    this.props.items.push(item)
  }
}
```

❌ **Bad**:
```typescript
// domain/entities/Portfolio.ts
import { prisma } from '@/infrastructure/database'  // NO!

export class Portfolio {
  async addItem(item: PortfolioItem): Promise<void> {
    await prisma.portfolioItem.create({ data: item })  // NO!
  }
}
```

### 2. Aggregate Boundaries

Define clear **aggregate roots**:

```typescript
// Portfolio is aggregate root
// PortfolioItem is part of Portfolio aggregate
// Always access PortfolioItem through Portfolio

✅ Good:
portfolio.addItem(item)

❌ Bad:
portfolioItemRepository.create(item)  // Bypasses aggregate
```

### 3. Event-Driven Communication

Use events for **inter-context communication**:

```typescript
// Portfolio context publishes event
eventBus.publish(new PortfolioCreatedEvent({ portfolioId, studentId }))

// Analytics context subscribes
class PortfolioCreatedHandler {
  async handle(event: PortfolioCreatedEvent) {
    await this.updateStudentPortfolioCount(event.payload.studentId)
  }
}
```

---

## Related Documentation

- [Domain Events](./adding-domain-events.md)
- [DDD Architecture](../architecture/overview.md#ddd-architecture)
- [Testing Guide](./testing.md)

---

**Context Statistics** (as of November 2025):
- **Total Contexts**: 15
- **Average Context Size**: ~2,500 LOC
- **Context Independence**: 95%
- **Event-Driven Integration**: 100%
