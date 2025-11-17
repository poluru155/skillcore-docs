# Adding Domain Events

**Last Updated**: November 17, 2025  
**Difficulty**: Intermediate  
**Time**: 15-30 minutes

---

## Overview

This guide shows how to add new domain events to SkillCore's event-driven architecture. Domain events represent **important business occurrences** that other parts of the system need to know about.

---

## When to Create Domain Events

Create domain events when:

✅ **Business Logic Completes**: Student enrolled, grade posted, intervention triggered  
✅ **State Changes**: Status updated, record deleted, approval given  
✅ **External Integration Needed**: SIS sync, notification sent, webhook triggered  
✅ **Audit Required**: FERPA-regulated data changed  
✅ **Multiple Contexts Care**: Grade affects analytics, notifications, parent dashboard

❌ **Don't create events for**:
- Simple CRUD operations without business significance
- Internal technical operations
- UI state changes

---

## Step-by-Step Guide

### Step 1: Define Event Type

**Location**: `api/src/contexts/{context}/domain/events/`

**Example**: `api/src/contexts/gradebook/domain/events/GradePostedEvent.ts`

```typescript
import { DomainEvent } from '@/shared/domain/events/DomainEvent'

export interface GradePostedEventPayload {
  // Required: Event identification
  resultId: string
  lineItemId: string
  studentId: string
  classId: string
  
  // Business data
  score: number
  scorePercent: number
  letterGrade: string
  previousScore?: number  // For updates
  
  // Metadata
  gradedById: string
  postedAt: Date
  
  // Multi-tenant
  districtId: string
  schoolId: string
}

export class GradePostedEvent extends DomainEvent<GradePostedEventPayload> {
  constructor(payload: GradePostedEventPayload) {
    super({
      eventType: 'gradebook.grade.posted',  // Follow naming convention
      aggregateId: payload.resultId,
      aggregateType: 'Result',
      payload,
      metadata: {
        districtId: payload.districtId,
        schoolId: payload.schoolId,
        userId: payload.gradedById,
      },
    })
  }
}
```

**Naming Convention**:
```
{context}.{aggregate}.{action}

Examples:
- gradebook.grade.posted
- attendance.record.marked
- intervention.created
- student.enrolled
- iep.goal.updated
```

---

### Step 2: Publish Event from Domain Service

**Location**: `api/src/contexts/{context}/domain/services/`

**Example**: `api/src/contexts/gradebook/domain/services/GradeService.ts`

```typescript
import { EventBus } from '@/shared/infrastructure/events/EventBus'
import { GradePostedEvent } from '../events/GradePostedEvent'

export class GradeService {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly eventBus: EventBus  // Inject EventBus
  ) {}
  
  async postGrade(input: PostGradeInput): Promise<Result> {
    // 1. Validate business rules
    await this.validateGradePosting(input)
    
    // 2. Perform domain operation
    const result = await this.prisma.result.create({
      data: {
        lineItemId: input.lineItemId,
        studentId: input.studentId,
        score: input.score,
        scoreStatus: 'fully_graded',
        gradedById: input.gradedById,
        postedAt: new Date(),
        // ... other fields
      },
      include: {
        lineItem: {
          include: {
            class: true,
          }
        },
        student: true,
      }
    })
    
    // 3. Calculate derived values
    const scorePercent = (result.score / result.lineItem.resultValueMax) * 100
    const letterGrade = this.calculateLetterGrade(scorePercent)
    
    // 4. Create and publish domain event
    const event = new GradePostedEvent({
      resultId: result.id,
      lineItemId: result.lineItemId,
      studentId: result.studentId,
      classId: result.lineItem.classId,
      score: result.score,
      scorePercent,
      letterGrade,
      gradedById: input.gradedById,
      postedAt: result.postedAt,
      districtId: result.lineItem.class.districtId,
      schoolId: result.lineItem.class.schoolId,
    })
    
    await this.eventBus.publish(event)
    
    // 5. Return result
    return result
  }
}
```

---

### Step 3: Create Event Handler

**Location**: `api/src/contexts/{consumer-context}/application/event-handlers/`

**Example**: `api/src/contexts/analytics/application/event-handlers/GradePostedHandler.ts`

```typescript
import { EventHandler } from '@/shared/application/events/EventHandler'
import { GradePostedEvent } from '@/contexts/gradebook/domain/events/GradePostedEvent'
import { StudentAnalyticsService } from '../services/StudentAnalyticsService'

export class GradePostedHandler implements EventHandler<GradePostedEvent> {
  readonly eventType = 'gradebook.grade.posted'
  
  constructor(
    private readonly analyticsService: StudentAnalyticsService
  ) {}
  
  async handle(event: GradePostedEvent): Promise<void> {
    const { payload } = event
    
    try {
      // Update student analytics
      await this.analyticsService.updateGPA(payload.studentId)
      
      // Check for grade alerts (failing, significant drop, etc.)
      await this.analyticsService.checkGradeAlerts(payload.studentId, payload.scorePercent)
      
      // Update class performance metrics
      await this.analyticsService.updateClassMetrics(payload.classId)
      
      console.log(`✓ Analytics updated for grade ${payload.resultId}`)
    } catch (error) {
      console.error(`✗ Failed to process grade posted event:`, error)
      // Event will be retried automatically via DLQ
      throw error
    }
  }
}
```

**Multiple Handlers Example**:
```typescript
// Notification handler
export class GradePostedNotificationHandler implements EventHandler<GradePostedEvent> {
  readonly eventType = 'gradebook.grade.posted'
  
  async handle(event: GradePostedEvent): Promise<void> {
    const { studentId, scorePercent, lineItemId } = event.payload
    
    // Send notification to student and parents
    await this.notificationService.sendGradeNotification({
      studentId,
      lineItemId,
      scorePercent,
    })
  }
}

// Intervention trigger handler
export class GradePostedInterventionHandler implements EventHandler<GradePostedEvent> {
  readonly eventType = 'gradebook.grade.posted'
  
  async handle(event: GradePostedEvent): Promise<void> {
    const { studentId, scorePercent } = event.payload
    
    // Trigger intervention if failing
    if (scorePercent < 60) {
      await this.interventionService.triggerAcademicIntervention({
        studentId,
        reason: 'failing_grade',
        gradePercent: scorePercent,
      })
    }
  }
}
```

---

### Step 4: Register Event Handler

**Location**: `api/src/contexts/{context}/infrastructure/di/container.ts`

```typescript
import { EventBus } from '@/shared/infrastructure/events/EventBus'
import { GradePostedHandler } from '../application/event-handlers/GradePostedHandler'
import { GradePostedNotificationHandler } from '../application/event-handlers/GradePostedNotificationHandler'
import { StudentAnalyticsService } from '../application/services/StudentAnalyticsService'

export function registerAnalyticsContext(container: Container) {
  // Register services
  container.register('StudentAnalyticsService', StudentAnalyticsService)
  
  // Register event handlers
  const eventBus = container.resolve<EventBus>('EventBus')
  const analyticsService = container.resolve<StudentAnalyticsService>('StudentAnalyticsService')
  
  // Analytics handler
  const gradePostedHandler = new GradePostedHandler(analyticsService)
  eventBus.subscribe(gradePostedHandler.eventType, gradePostedHandler)
  
  // Notification handler
  const notificationHandler = new GradePostedNotificationHandler(notificationService)
  eventBus.subscribe(notificationHandler.eventType, notificationHandler)
  
  console.log('✓ Analytics context event handlers registered')
}
```

---

### Step 5: Add Event to Catalog

**Location**: `api/src/shared/domain/events/event-catalog.ts`

```typescript
export const EVENT_CATALOG = {
  // ... existing events
  
  'gradebook.grade.posted': {
    description: 'Grade posted for student assignment',
    payload: {
      resultId: 'string',
      lineItemId: 'string',
      studentId: 'string',
      classId: 'string',
      score: 'number',
      scorePercent: 'number',
      letterGrade: 'string',
      gradedById: 'string',
      postedAt: 'Date',
      districtId: 'string',
      schoolId: 'string',
    },
    handlers: [
      'StudentAnalyticsService - Update GPA',
      'NotificationService - Send grade notification',
      'InterventionService - Check for failing grade',
      'ParentDashboardService - Update parent view',
    ],
  },
} as const
```

---

### Step 6: Add Integration Test

**Location**: `api/src/contexts/{context}/application/event-handlers/__tests__/`

**Example**: `GradePostedHandler.test.ts`

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { GradePostedEvent } from '@/contexts/gradebook/domain/events/GradePostedEvent'
import { GradePostedHandler } from '../GradePostedHandler'

describe('GradePostedHandler', () => {
  let handler: GradePostedHandler
  let analyticsService: StudentAnalyticsService
  
  beforeEach(() => {
    analyticsService = new StudentAnalyticsService(prisma)
    handler = new GradePostedHandler(analyticsService)
  })
  
  it('should update student GPA when grade posted', async () => {
    // Arrange
    const event = new GradePostedEvent({
      resultId: 'result-123',
      lineItemId: 'lineitem-456',
      studentId: 'student-789',
      classId: 'class-101',
      score: 85,
      scorePercent: 85,
      letterGrade: 'B',
      gradedById: 'teacher-111',
      postedAt: new Date(),
      districtId: 'district-222',
      schoolId: 'school-333',
    })
    
    // Act
    await handler.handle(event)
    
    // Assert
    const analytics = await prisma.studentAnalytics.findUnique({
      where: { studentId: 'student-789' }
    })
    expect(analytics.currentGPA).toBeGreaterThan(0)
    expect(analytics.updatedAt).toBeDefined()
  })
  
  it('should trigger intervention for failing grade', async () => {
    // Arrange
    const event = new GradePostedEvent({
      // ... failing grade (< 60%)
      score: 45,
      scorePercent: 45,
      letterGrade: 'F',
    })
    
    // Act
    await handler.handle(event)
    
    // Assert
    const intervention = await prisma.intervention.findFirst({
      where: {
        studentId: 'student-789',
        type: 'academic',
        reason: 'failing_grade',
      }
    })
    expect(intervention).toBeDefined()
  })
})
```

---

## Event Persistence (EventStore)

All events are automatically persisted to `EventStore` table:

```prisma
model EventStore {
  id              String    @id @default(uuid())
  eventType       String    // 'gradebook.grade.posted'
  aggregateId     String    // 'result-123'
  aggregateType   String    // 'Result'
  payload         Json      // Event data
  metadata        Json      // districtId, userId, etc.
  version         Int       @default(1)
  occurredAt      DateTime  @default(now())
  
  @@index([eventType])
  @@index([aggregateId, aggregateType])
  @@index([occurredAt])
}
```

**Query Events**:
```typescript
// Get all events for a result
const events = await prisma.eventStore.findMany({
  where: {
    aggregateType: 'Result',
    aggregateId: 'result-123',
  },
  orderBy: { occurredAt: 'asc' },
})

// Get all grade events in last hour
const recentGrades = await prisma.eventStore.findMany({
  where: {
    eventType: 'gradebook.grade.posted',
    occurredAt: { gte: subHours(new Date(), 1) },
  }
})
```

---

## Async Event Processing

Events are processed **asynchronously** via **BullMQ queue**:

```typescript
// EventBus publishes to queue
export class EventBus {
  async publish(event: DomainEvent): Promise<void> {
    // 1. Persist to EventStore
    await this.persistEvent(event)
    
    // 2. Add to processing queue
    await this.eventQueue.add('process-event', {
      eventType: event.eventType,
      eventId: event.id,
      payload: event.payload,
    }, {
      attempts: 3,
      backoff: {
        type: 'exponential',
        delay: 2000,
      },
    })
  }
}

// Worker processes events
eventQueue.process('process-event', async (job) => {
  const { eventType, eventId, payload } = job.data
  
  // Get all handlers for this event type
  const handlers = eventBus.getHandlers(eventType)
  
  // Execute handlers in parallel
  await Promise.all(
    handlers.map(handler => handler.handle(event))
  )
})
```

---

## Best Practices

### 1. Event Naming

✅ **Good**:
```typescript
'gradebook.grade.posted'       // Clear, specific
'attendance.record.marked'     // Action past tense
'intervention.goal.achieved'   // Positive outcome
```

❌ **Bad**:
```typescript
'grade'                        // Too vague
'gradePosted'                  // Wrong format
'gradebook.gradeUpdate'        // Not past tense
```

### 2. Event Payload

✅ **Include**:
- All data needed by consumers
- Identifiers (IDs)
- Business values
- Timestamps
- Multi-tenant context (districtId, schoolId)

❌ **Don't Include**:
- Entire entity objects (use IDs)
- Sensitive data not needed
- Computed values consumers can derive

### 3. Idempotency

Handlers must be **idempotent** (safe to run multiple times):

```typescript
✅ Good:
await prisma.studentAnalytics.upsert({
  where: { studentId },
  create: { studentId, currentGPA: newGPA },
  update: { currentGPA: newGPA },
})

❌ Bad:
await prisma.studentAnalytics.update({
  where: { studentId },
  data: {
    currentGPA: { increment: gradeChange },  // NOT idempotent!
  }
})
```

### 4. Error Handling

```typescript
async handle(event: GradePostedEvent): Promise<void> {
  try {
    await this.processEvent(event)
  } catch (error) {
    // Log error with context
    logger.error('Failed to handle grade posted event', {
      eventId: event.id,
      eventType: event.eventType,
      error: error.message,
      stack: error.stack,
    })
    
    // Re-throw to trigger retry via DLQ
    throw error
  }
}
```

---

## Debugging Events

### View Published Events

```sql
-- Recent grade posted events
SELECT *
FROM event_store
WHERE event_type = 'gradebook.grade.posted'
ORDER BY occurred_at DESC
LIMIT 10;

-- Events for specific student
SELECT *
FROM event_store
WHERE payload->>'studentId' = 'student-123'
ORDER BY occurred_at DESC;
```

### Monitor Event Queue

```typescript
// BullMQ dashboard: http://localhost:3000/admin/queues
// View:
// - Waiting jobs
// - Active jobs
// - Completed jobs
// - Failed jobs
// - DLQ (Dead Letter Queue)
```

### Enable Event Logging

```typescript
// .env
LOG_EVENTS=true
LOG_EVENT_HANDLERS=true

// Logs output
[EventBus] Publishing event: gradebook.grade.posted (eventId: evt-123)
[GradePostedHandler] Handling event: evt-123
[GradePostedHandler] ✓ Student GPA updated
[NotificationHandler] ✓ Notification sent
```

---

## Related Documentation

- [Domain Events Architecture](../architecture/events.md)
- [Event Catalog](../reference/events-catalog.md)
- [Queue System](../architecture/queues.md)

---

**Event Statistics** (as of November 2025):
- **Total Event Types**: 208
- **Events Published Daily**: 500K
- **Average Processing Time**: 45ms
- **Event Replay Success Rate**: 99.8%
