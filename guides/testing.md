# Testing Guide

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Testing Framework**: Vitest + React Testing Library + Playwright

---

## Overview

SkillCore uses a **comprehensive testing strategy** with **unit tests**, **integration tests**, and **end-to-end tests** to ensure educational data integrity and user experience quality.

---

## Testing Stack

### Backend (API)

- **Unit/Integration**: Vitest
- **Database**: PostgreSQL Test Instance
- **Mocking**: Vitest mocks
- **Coverage**: c8

### Frontend (WebApp)

- **Unit/Component**: Vitest + React Testing Library
- **E2E**: Playwright
- **Visual Regression**: Playwright Screenshots
- **Accessibility**: axe-core

---

## Backend Testing

### Unit Tests

**Location**: `api/src/**/__tests__/*.test.ts`

**Example**: Testing Domain Entity

```typescript
import { describe, it, expect } from 'vitest'
import { Portfolio } from '../entities/Portfolio'
import { PortfolioItem } from '../entities/PortfolioItem'

describe('Portfolio', () => {
  describe('addItem', () => {
    it('should add item to portfolio', () => {
      const portfolio = Portfolio.create({
        studentId: 'student-123',
        title: 'My Portfolio',
        isPublic: false,
      })
      
      const item = PortfolioItem.create({
        title: 'Math Project',
        itemType: 'project',
      })
      
      portfolio.addItem(item)
      
      expect(portfolio.items).toHaveLength(1)
      expect(portfolio.items[0].title).toBe('Math Project')
    })
    
    it('should throw error when adding more than 50 items', () => {
      const portfolio = Portfolio.create({
        studentId: 'student-123',
        title: 'My Portfolio',
        isPublic: false,
      })
      
      // Add 50 items
      for (let i = 0; i < 50; i++) {
        portfolio.addItem(PortfolioItem.create({
          title: `Item ${i}`,
          itemType: 'work_sample',
        }))
      }
      
      // 51st item should throw
      expect(() => {
        portfolio.addItem(PortfolioItem.create({
          title: 'Item 51',
          itemType: 'work_sample',
        }))
      }).toThrow('Portfolio cannot have more than 50 items')
    })
  })
})
```

**Example**: Testing Use Case

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest'
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
  
  it('should create portfolio and publish event', async () => {
    const input = {
      studentId: 'student-123',
      title: 'My Portfolio',
      districtId: 'district-456',
      schoolId: 'school-789',
    }
    
    const portfolio = await useCase.execute(input)
    
    // Verify portfolio created
    expect(portfolio.studentId).toBe('student-123')
    expect(portfolio.title).toBe('My Portfolio')
    
    // Verify saved to repository
    const saved = await repository.findById(portfolio.id)
    expect(saved).toBeDefined()
    
    // Verify event published
    const events = eventBus.getPublishedEvents()
    expect(events).toHaveLength(1)
    expect(events[0].eventType).toBe('portfolio.created')
    expect(events[0].payload.portfolioId).toBe(portfolio.id.toString())
  })
})
```

### Integration Tests

**Location**: `api/src/**/__tests__/*.integration.test.ts`

**Example**: Testing API Endpoint

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import { createTestContext, TestContext } from '@/test/test-context'
import { GraphQLClient } from 'graphql-request'

describe('Portfolio GraphQL API', () => {
  let ctx: TestContext
  let client: GraphQLClient
  
  beforeAll(async () => {
    ctx = await createTestContext()
    client = ctx.graphqlClient
  })
  
  afterAll(async () => {
    await ctx.cleanup()
  })
  
  it('should create portfolio', async () => {
    const mutation = `
      mutation CreatePortfolio($input: CreatePortfolioInput!) {
        createPortfolio(input: $input) {
          portfolio {
            id
            title
            studentId
          }
          errors {
            message
          }
        }
      }
    `
    
    const response = await client.request(mutation, {
      input: {
        studentId: ctx.student.id,
        title: 'My Test Portfolio',
      }
    })
    
    expect(response.createPortfolio.portfolio).toBeDefined()
    expect(response.createPortfolio.portfolio.title).toBe('My Test Portfolio')
    expect(response.createPortfolio.errors).toBeNull()
  })
  
  it('should require authentication', async () => {
    const unauthenticatedClient = new GraphQLClient(ctx.apiUrl)
    
    const mutation = `
      mutation CreatePortfolio($input: CreatePortfolioInput!) {
        createPortfolio(input: $input) {
          portfolio { id }
        }
      }
    `
    
    await expect(
      unauthenticatedClient.request(mutation, {
        input: { studentId: 'student-123', title: 'Test' }
      })
    ).rejects.toThrow('Unauthorized')
  })
})
```

### Database Tests

**Example**: Testing Repository

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { PrismaClient } from '@prisma/client'
import { PrismaPortfolioRepository } from '../PrismaPortfolioRepository'
import { Portfolio } from '../../../domain/entities/Portfolio'
import { createTestDatabase } from '@/test/test-database'

describe('PrismaPortfolioRepository', () => {
  let prisma: PrismaClient
  let repository: PrismaPortfolioRepository
  
  beforeEach(async () => {
    prisma = await createTestDatabase()
    repository = new PrismaPortfolioRepository(prisma)
  })
  
  it('should save and retrieve portfolio', async () => {
    const portfolio = Portfolio.create({
      studentId: 'student-123',
      title: 'Test Portfolio',
      isPublic: false,
    })
    
    await repository.save(portfolio)
    
    const retrieved = await repository.findById(portfolio.id)
    
    expect(retrieved).toBeDefined()
    expect(retrieved!.title).toBe('Test Portfolio')
    expect(retrieved!.studentId).toBe('student-123')
  })
  
  it('should soft delete portfolio', async () => {
    const portfolio = Portfolio.create({
      studentId: 'student-123',
      title: 'Test Portfolio',
      isPublic: false,
    })
    
    await repository.save(portfolio)
    await repository.delete(portfolio.id)
    
    const retrieved = await repository.findById(portfolio.id)
    expect(retrieved).toBeNull()  // Soft-deleted records not returned
  })
})
```

---

## Frontend Testing

### Component Tests

**Location**: `src/components/**/__tests__/*.test.tsx`

**Example**: Testing UI Component

```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { StudentCard } from '../student-card'

describe('StudentCard', () => {
  const mockStudent = {
    id: 'student-123',
    givenName: 'John',
    familyName: 'Doe',
    gradeLevel: '10',
    studentNumber: '123456',
    currentGPA: 3.5,
    attendanceRate: 95.5,
  }
  
  it('should render student information', () => {
    render(<StudentCard student={mockStudent} />)
    
    expect(screen.getByText('John Doe')).toBeInTheDocument()
    expect(screen.getByText('Grade 10')).toBeInTheDocument()
    expect(screen.getByText('3.50')).toBeInTheDocument()
    expect(screen.getByText('95.5%')).toBeInTheDocument()
  })
  
  it('should call onSelect when clicked', async () => {
    const onSelect = vi.fn()
    
    render(<StudentCard student={mockStudent} onSelect={onSelect} />)
    
    const viewButton = screen.getByRole('button', { name: /view details/i })
    fireEvent.click(viewButton)
    
    await waitFor(() => {
      expect(onSelect).toHaveBeenCalledWith(mockStudent)
    })
  })
  
  it('should display badges for special programs', () => {
    const studentWithIEP = {
      ...mockStudent,
      hasIEP: true,
      has504: true,
      isESL: true,
    }
    
    render(<StudentCard student={studentWithIEP} />)
    
    expect(screen.getByText('IEP')).toBeInTheDocument()
    expect(screen.getByText('504')).toBeInTheDocument()
    expect(screen.getByText('ESL')).toBeInTheDocument()
  })
})
```

### Hook Tests

**Example**: Testing Custom Hook

```typescript
import { describe, it, expect } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useStudents } from '../use-students'

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  })
  
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

describe('useStudents', () => {
  it('should fetch students', async () => {
    const { result } = renderHook(
      () => useStudents({ schoolId: 'school-123' }),
      { wrapper: createWrapper() }
    )
    
    expect(result.current.isLoading).toBe(true)
    
    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true)
    })
    
    expect(result.current.data).toBeDefined()
    expect(result.current.data.length).toBeGreaterThan(0)
  })
})
```

### Accessibility Tests

**Example**: Testing Accessibility

```typescript
import { describe, it, expect } from 'vitest'
import { render } from '@testing-library/react'
import { axe, toHaveNoViolations } from 'jest-axe'
import { GradeEntryForm } from '../grade-entry-form'

expect.extend(toHaveNoViolations)

describe('GradeEntryForm Accessibility', () => {
  it('should not have accessibility violations', async () => {
    const { container } = render(
      <GradeEntryForm
        studentId="student-123"
        lineItemId="lineitem-456"
        onSubmit={() => {}}
      />
    )
    
    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })
  
  it('should have proper ARIA labels', () => {
    const { getByLabelText } = render(
      <GradeEntryForm
        studentId="student-123"
        lineItemId="lineitem-456"
        onSubmit={() => {}}
      />
    )
    
    expect(getByLabelText(/grade/i)).toBeInTheDocument()
    expect(getByLabelText(/comment/i)).toBeInTheDocument()
  })
})
```

---

## E2E Testing (Playwright)

**Location**: `tests/e2e/**/*.spec.ts`

**Example**: Testing Teacher Workflow

```typescript
import { test, expect } from '@playwright/test'

test.describe('Teacher Gradebook', () => {
  test.beforeEach(async ({ page }) => {
    // Login as teacher
    await page.goto('/login')
    await page.fill('[name="email"]', 'teacher@test.com')
    await page.fill('[name="password"]', 'password123')
    await page.click('button[type="submit"]')
    
    await page.waitForURL('/dashboard')
  })
  
  test('should post grade for student', async ({ page }) => {
    // Navigate to gradebook
    await page.click('text=Gradebook')
    await page.click('text=Algebra I - Period 1')
    
    // Find student row and grade cell
    const studentRow = page.locator('tr:has-text("John Doe")')
    const gradeCell = studentRow.locator('td[data-column="assignment-1"]')
    
    // Enter grade
    await gradeCell.click()
    await gradeCell.fill('85')
    await gradeCell.press('Enter')
    
    // Verify grade saved
    await expect(gradeCell).toHaveText('85')
    await expect(page.locator('.toast-success')).toContainText('Grade saved')
  })
  
  test('should validate grade input', async ({ page }) => {
    await page.goto('/gradebook/class-123')
    
    const gradeCell = page.locator('[data-testid="grade-cell-student-1"]')
    
    // Try invalid grade (over 100)
    await gradeCell.click()
    await gradeCell.fill('150')
    await gradeCell.press('Enter')
    
    // Verify error message
    await expect(page.locator('.error-message')).toContainText(
      'Grade must be between 0 and 100'
    )
  })
  
  test('should be accessible with keyboard', async ({ page }) => {
    await page.goto('/gradebook/class-123')
    
    // Tab to first grade cell
    await page.keyboard.press('Tab')
    await page.keyboard.press('Tab')
    
    // Enter grade with keyboard
    await page.keyboard.type('92')
    await page.keyboard.press('Enter')
    
    // Verify grade saved
    const cell = page.locator(':focus')
    await expect(cell).toHaveValue('92')
  })
})
```

**Example**: Testing Parent View

```typescript
import { test, expect } from '@playwright/test'

test.describe('Parent Dashboard', () => {
  test('should view student grades', async ({ page }) => {
    // Login as parent
    await page.goto('/login')
    await page.fill('[name="email"]', 'parent@test.com')
    await page.fill('[name="password"]', 'password123')
    await page.click('button[type="submit"]')
    
    // View student grades
    await page.click('text=View Grades')
    
    // Verify grades displayed
    await expect(page.locator('h1')).toContainText('Grades')
    await expect(page.locator('[data-testid="class-grade"]')).toHaveCount(5)
    
    // Check specific grade
    const algebraGrade = page.locator('text=Algebra I').locator('..')
    await expect(algebraGrade).toContainText('A')
    await expect(algebraGrade).toContainText('92%')
  })
})
```

---

## Test Utilities

### Test Context Factory

**Location**: `api/test/test-context.ts`

```typescript
import { PrismaClient } from '@prisma/client'
import { GraphQLClient } from 'graphql-request'
import { startServer } from '@/server'

export interface TestContext {
  prisma: PrismaClient
  graphqlClient: GraphQLClient
  apiUrl: string
  teacher: any
  student: any
  parent: any
  cleanup: () => Promise<void>
}

export async function createTestContext(): Promise<TestContext> {
  // Start test server
  const server = await startServer({ port: 0 })
  const apiUrl = `http://localhost:${server.address().port}`
  
  // Create test database
  const prisma = new PrismaClient({
    datasources: {
      db: { url: process.env.TEST_DATABASE_URL }
    }
  })
  
  // Seed test data
  const teacher = await prisma.user.create({
    data: {
      email: 'teacher@test.com',
      role: 'TEACHER',
      givenName: 'Test',
      familyName: 'Teacher',
      districtId: 'district-test',
    }
  })
  
  const student = await prisma.student.create({
    data: {
      studentNumber: '123456',
      givenName: 'Test',
      familyName: 'Student',
      gradeLevel: '10',
      districtId: 'district-test',
      schoolId: 'school-test',
    }
  })
  
  // Create authenticated GraphQL client
  const authToken = await generateTestToken(teacher.id)
  const graphqlClient = new GraphQLClient(`${apiUrl}/graphql`, {
    headers: { Authorization: `Bearer ${authToken}` }
  })
  
  return {
    prisma,
    graphqlClient,
    apiUrl,
    teacher,
    student,
    parent: null,
    cleanup: async () => {
      await prisma.$disconnect()
      await server.close()
    }
  }
}
```

---

## Running Tests

### Backend Tests

```bash
# Run all tests
cd api
pnpm test

# Run specific test file
pnpm test src/contexts/gradebook/__tests__/GradeService.test.ts

# Run with coverage
pnpm test:coverage

# Watch mode
pnpm test:watch
```

### Frontend Tests

```bash
# Run all tests
pnpm test

# Run specific test
pnpm test src/components/educational/__tests__/student-card.test.tsx

# Run with UI
pnpm test:ui

# Coverage
pnpm test:coverage
```

### E2E Tests

```bash
# Run all E2E tests
pnpm test:e2e

# Run specific test
pnpm test:e2e tests/e2e/gradebook.spec.ts

# Run with UI
pnpm test:e2e --ui

# Run in specific browser
pnpm test:e2e --project=chromium
pnpm test:e2e --project=firefox
pnpm test:e2e --project=webkit
```

---

## Coverage Requirements

- **Unit Tests**: 80% minimum
- **Integration Tests**: 60% minimum
- **E2E Tests**: Critical paths covered

**Check Coverage**:
```bash
cd api
pnpm test:coverage

# Output
File                  | % Stmts | % Branch | % Funcs | % Lines
----------------------|---------|----------|---------|--------
All files             |   85.2  |   78.5   |   82.1  |   85.8
 domain/entities      |   92.1  |   88.3   |   90.5  |   92.4
 application/use-cases|   78.5  |   72.1   |   75.8  |   79.2
```

---

## Best Practices

### 1. Test Organization

```
Follow AAA pattern:
- Arrange: Set up test data
- Act: Execute code under test
- Assert: Verify results
```

### 2. Test Isolation

```typescript
✅ Good: Each test independent
beforeEach(async () => {
  await cleanDatabase()
  testData = await seedTestData()
})

❌ Bad: Tests depend on each other
test('create user', () => { /* creates user */ })
test('update user', () => { /* assumes user exists */ })
```

### 3. Descriptive Names

```typescript
✅ Good:
it('should throw error when posting grade over maximum points', ...)

❌ Bad:
it('grade test', ...)
```

---

## Related Documentation

- [CI/CD](./deployment.md#cicd-pipeline)
- [Development Setup](./getting-started.md)

---

**Testing Statistics** (as of November 2025):
- **Total Tests**: 2,845
- **Unit Tests**: 1,920
- **Integration Tests**: 725
- **E2E Tests**: 200
- **Test Coverage**: 85.2%
- **Test Execution Time**: 4.5 minutes
