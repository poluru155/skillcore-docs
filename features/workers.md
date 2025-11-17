# Background Workers & Scheduled Jobs

**Last Updated**: November 16, 2025  
**Status**: ✅ Production-Ready

---

## Overview

SkillCore WebApp uses a robust background job processing system with BullMQ queues, standalone event workers, and cron-based scheduled tasks for scalable, reliable background processing.

---

## Event Workers

### Architecture

**Standalone Processes**: Event workers run as separate Node.js processes from the API servers.

```
API Instances (Publish Only)     Event Workers (Consume Only)
        ↓                                ↓
   Redis Stream  ←────────────────→  Consumer Groups
        ↓                                ↓
  EventStore                        Event Handlers
   (Postgres)                       (Side Effects)
```

### Deployment

**Development**:
```bash
# Start event worker with hot reload
pnpm worker:dev

# Start API (publishes events)
pnpm dev
```

**Production**:
```bash
# Docker Compose with scaling
docker-compose up api --scale api=3
docker-compose up event-worker --scale event-worker=5

# Or with Kubernetes
kubectl scale deployment event-worker --replicas=5
```

### Configuration

**Worker Process** (`event-worker.ts`):
```typescript
import { RedisEventBus } from './infrastructure/event-bus/RedisEventBus'
import { registerHandlers } from './register-handlers'

async function main() {
  // Initialize EventBus in CONSUME mode
  const eventBus = new RedisEventBus({
    mode: 'consume',
    consumerGroup: 'event-workers',
    consumerId: process.env.HOSTNAME || `worker-${process.pid}`
  })

  await eventBus.start()

  // Register all 79 handlers
  registerHandlers(eventBus)

  // Health check endpoint
  app.get('/health', () => ({ status: 'ok', handlersRegistered: 79 }))

  // Graceful shutdown
  process.on('SIGTERM', async () => {
    await eventBus.stop()
    process.exit(0)
  })
}

main().catch(console.error)
```

### Handler Registration

**Centralized Registration** (`register-handlers.ts`):
```typescript
export function registerHandlers(eventBus: IEventBus) {
  const serviceRegistry = getServiceRegistry()

  // Special Programs (6 handlers)
  eventBus.subscribe(IEPCreated, new CoordinatorNotificationHandler(...))
  eventBus.subscribe(IEPGoalAdded, new IEPInterventionHandler(...))
  // ... 4 more

  // Notifications (6 handlers)
  eventBus.subscribe(EmailSent, new NotificationDeliveryAnalyticsHandler(...))
  eventBus.subscribe(SMSDelivered, new SMSDeliveryHandler(...))
  // ... 4 more

  // Communication (8 handlers)
  eventBus.subscribe(MessageSent, new MessageDeliveryHandler(...))
  // ... 7 more

  // Gradebook (12 handlers)
  eventBus.subscribe(GradeUpdated, new GradeCalculationHandler(...))
  // ... 11 more

  // Attendance (4 handlers)
  eventBus.subscribe(AttendanceMarked, new DailyAttendanceAggregationHandler(...))
  // ... 3 more

  // Chat (10 handlers)
  eventBus.subscribe(ChatMessageSent, new MessageDeliveryHandler(...))
  // ... 9 more

  // Demographics (6 handlers)
  eventBus.subscribe(UserCreated, new UserProfileHandler(...))
  // ... 5 more

  // Rostering (12 handlers)
  eventBus.subscribe(ClassCreated, new ClassCreationHandler(...))
  // ... 11 more

  // Total: 79 handlers registered
}
```

### Health Monitoring

**Health Check Endpoint**: `/health/events`

```json
{
  "status": "healthy",
  "streams": {
    "domain-events": {
      "pending": 12,
      "consumerGroups": [
        {
          "name": "event-workers",
          "consumers": 5,
          "pending": 12,
          "lag": 0
        }
      ],
      "lastEventTimestamp": "2025-11-16T10:30:00Z"
    }
  },
  "dlq": {
    "messageCount": 2,
    "oldestMessage": "2025-11-15T08:00:00Z"
  },
  "handlersRegistered": 79
}
```

---

## BullMQ Queues

### Queue Architecture

**7 Production Queues**:
1. `notification-delivery` - Email, SMS, Push dispatching
2. `grade-processing` - Grade calculation and posting
3. `attendance-aggregation` - Daily attendance rollups
4. `report-generation` - PDF/Excel report creation
5. `data-archival` - Old data cleanup and archiving
6. `intervention-detection` - At-risk student identification
7. `compliance-reporting` - FERPA audit log generation

### Queue Configuration

```typescript
import { Queue, Worker } from 'bullmq'

// Notification Delivery Queue
const notificationQueue = new Queue('notification-delivery', {
  connection: redisConnection,
  defaultJobOptions: {
    attempts: 5,
    backoff: {
      type: 'exponential',
      delay: 1000 // 1s, 2s, 4s, 8s, 16s
    },
    removeOnComplete: {
      age: 24 * 3600, // Keep for 24 hours
      count: 1000     // Keep last 1000 jobs
    },
    removeOnFail: {
      age: 7 * 24 * 3600 // Keep failures for 7 days
    }
  }
})

// Worker Process
const notificationWorker = new Worker(
  'notification-delivery',
  async (job) => {
    const { channel, recipient, content } = job.data

    if (channel === 'email') {
      await sendGridService.send(recipient, content)
    } else if (channel === 'sms') {
      await twilioService.send(recipient, content)
    } else if (channel === 'push') {
      await fcmService.send(recipient, content)
    }

    return { sent: true, timestamp: new Date() }
  },
  {
    connection: redisConnection,
    concurrency: 10 // Process 10 jobs in parallel
  }
)
```

### Priority Queues

**Job Priorities**:
```typescript
// High priority (emergency notifications)
await notificationQueue.add('emergency-alert', data, {
  priority: 1
})

// Normal priority (grade notifications)
await notificationQueue.add('grade-posted', data, {
  priority: 5
})

// Low priority (digest emails)
await notificationQueue.add('weekly-digest', data, {
  priority: 10
})
```

### Queue Metrics

**Prometheus Metrics**:
```typescript
bullmq_queue_waiting_count{queue="notification-delivery"} 45
bullmq_queue_active_count{queue="notification-delivery"} 10
bullmq_queue_completed_count{queue="notification-delivery"} 15234
bullmq_queue_failed_count{queue="notification-delivery"} 23
bullmq_processing_duration_seconds{queue="notification-delivery"} 0.45
```

---

## Scheduled Jobs

### Daily Jobs (Runs at 2:00 AM)

**1. Stream Cleanup**:
```typescript
// Cron: 0 2 * * *
async function streamCleanup() {
  const redis = getRedisClient()
  const streams = ['domain-events', 'chat-events', 'notification-events']

  for (const stream of streams) {
    // Keep last 30 days
    const cutoff = Date.now() - (30 * 24 * 60 * 60 * 1000)
    await redis.xtrim(stream, 'MINID', cutoff)
  }
}
```

**2. Attendance Intervention Detection**:
```typescript
// Cron: 0 2 * * *
async function detectAttendanceInterventions() {
  const students = await getStudentsWithPoorAttendance()

  for (const student of students) {
    if (student.absenceCount >= 3 && !student.hasIntervention) {
      await createIntervention({
        studentId: student.id,
        type: 'attendance',
        tier: 2,
        reason: `${student.absenceCount} absences in last 7 days`,
        strategies: ['Daily check-in', 'Parent contact', 'Attendance contract']
      })
    }
  }
}
```

**3. IEP Review Reminders**:
```typescript
// Cron: 0 8 * * *
async function iepReviewReminders() {
  const today = new Date()

  // 30-day reminder
  const ieps30Days = await getIEPsForReview(30)
  for (const iep of ieps30Days) {
    await eventBus.publish(new IEPReviewScheduled({
      iepId: iep.id,
      daysUntilDue: 30
    }))
  }

  // 60-day reminder
  const ieps60Days = await getIEPsForReview(60)
  // ... similar logic

  // 90-day reminder
  const ieps90Days = await getIEPsForReview(90)
  // ... similar logic
}
```

**4. Grade Posting Digest**:
```typescript
// Cron: 0 7 * * 1-5 (Weekdays at 7 AM)
async function gradePostingDigest() {
  const yesterday = new Date(Date.now() - 24 * 60 * 60 * 1000)
  const newGrades = await getGradesPostedSince(yesterday)

  // Group by student
  const gradesByStudent = groupBy(newGrades, 'studentId')

  for (const [studentId, grades] of gradesByStudent) {
    await notificationQueue.add('daily-grade-digest', {
      studentId,
      grades,
      date: yesterday
    })
  }
}
```

### Weekly Jobs (Runs Sundays at 3:00 AM)

**1. Performance Trend Calculations**:
```typescript
// Cron: 0 3 * * 0
async function calculatePerformanceTrends() {
  const students = await getAllActiveStudents()

  for (const student of students) {
    const grades = await getRecentGrades(student.id, 30) // Last 30 days
    const trend = calculateTrend(grades)

    await updateStudentMetrics(student.id, {
      trendDirection: trend.direction, // 'up', 'down', 'stable'
      trendSlope: trend.slope,
      lastCalculated: new Date()
    })
  }
}
```

**2. Resource Usage Analytics**:
```typescript
// Cron: 0 3 * * 0
async function resourceUsageAnalytics() {
  const resources = await getAllResources()

  for (const resource of resources) {
    const downloads = await getDownloadCount(resource.id, 7) // Last 7 days
    const views = await getViewCount(resource.id, 7)
    const ratings = await getAverageRating(resource.id)

    await updateResourceMetrics(resource.id, {
      weeklyDownloads: downloads,
      weeklyViews: views,
      averageRating: ratings,
      lastUpdated: new Date()
    })
  }
}
```

**3. Inactive User Notifications**:
```typescript
// Cron: 0 9 * * 0 (Sundays at 9 AM)
async function inactiveUserNotifications() {
  const inactiveUsers = await getUsersInactiveSince(14) // 14 days

  for (const user of inactiveUsers) {
    if (user.role === 'STUDENT') {
      await notificationQueue.add('inactive-student', {
        userId: user.id,
        lastLogin: user.lastLoginAt,
        missedAssignments: await getMissedAssignments(user.id)
      })
    }
  }
}
```

### Monthly Jobs (Runs 1st of Month at 4:00 AM)

**1. Compliance Audit Reports**:
```typescript
// Cron: 0 4 1 * *
async function complianceAuditReports() {
  const lastMonth = getLastMonthDateRange()

  const report = {
    iepReviews: await getIEPReviewsCompleted(lastMonth),
    section504Reviews: await get504ReviewsCompleted(lastMonth),
    dataAccessLogs: await getFERPAAccessLogs(lastMonth),
    consentForms: await getConsentFormsObtained(lastMonth)
  }

  await generateComplianceReport(report)
  await notifyDistrictAdmin(report)
}
```

**2. Data Archival to Cold Storage**:
```typescript
// Cron: 0 4 1 * *
async function archiveOldData() {
  const cutoffDate = new Date(Date.now() - (365 * 24 * 60 * 60 * 1000)) // 1 year ago

  // Archive old messages
  const oldMessages = await getMessagesOlderThan(cutoffDate)
  await archiveToS3(oldMessages, 'messages')
  await deleteFromDatabase(oldMessages)

  // Archive old notifications
  const oldNotifications = await getNotificationsOlderThan(cutoffDate)
  await archiveToS3(oldNotifications, 'notifications')
  await deleteFromDatabase(oldNotifications)

  // Keep audit logs for 7 years (FERPA requirement)
  const veryOldAuditLogs = await getAuditLogsOlderThan(7 * 365)
  await archiveToS3(veryOldAuditLogs, 'audit-logs')
}
```

**3. System Health Summary**:
```typescript
// Cron: 0 5 1 * *
async function systemHealthSummary() {
  const metrics = {
    apiUptime: await getAPIUptime(),
    averageResponseTime: await getAverageResponseTime(),
    errorRate: await getErrorRate(),
    eventProcessingLag: await getEventLag(),
    queueBacklog: await getQueueBacklog(),
    storageUsage: await getStorageUsage(),
    activeSessions: await getActiveSessionCount()
  }

  await generateHealthReport(metrics)
  await notifySystemAdmins(metrics)
}
```

---

## Retry & Error Handling

### Exponential Backoff

```typescript
const retryConfig = {
  attempts: 5,
  backoff: {
    type: 'exponential',
    delay: 1000, // Initial delay: 1 second
    // Retry schedule: 1s, 2s, 4s, 8s, 16s
  }
}
```

### Dead Letter Queue

**DLQ Processing**:
```typescript
// Failed jobs after 5 attempts → DLQ
const dlqWorker = new Worker('notification-delivery-dlq', async (job) => {
  // Log to monitoring system
  await logFailedJob(job)

  // Notify ops team for manual review
  await notifyOpsTeam({
    queue: job.queueName,
    jobId: job.id,
    data: job.data,
    failureReason: job.failedReason,
    attempts: job.attemptsMade
  })

  // Archive for analysis
  await archiveFailedJob(job)
})
```

### Circuit Breaker

**External Service Protection**:
```typescript
import CircuitBreaker from 'opossum'

const sendGridBreaker = new CircuitBreaker(sendGridService.send, {
  timeout: 5000,          // 5 second timeout
  errorThresholdPercentage: 50, // Open circuit at 50% failure
  resetTimeout: 30000     // Try again after 30 seconds
})

sendGridBreaker.on('open', () => {
  logger.error('SendGrid circuit breaker opened')
  // Fall back to alternate channel (SMS)
})
```

---

## Monitoring & Alerts

### Prometheus Metrics

**Queue Metrics**:
- `bullmq_queue_waiting_count` - Jobs waiting in queue
- `bullmq_queue_active_count` - Jobs currently processing
- `bullmq_queue_completed_count` - Successfully completed jobs
- `bullmq_queue_failed_count` - Failed jobs
- `bullmq_processing_duration_seconds` - Job processing time

**Worker Metrics**:
- `event_worker_count` - Active event workers
- `event_processing_duration_seconds` - Event handler duration
- `event_handler_errors_total` - Handler errors
- `dlq_message_count` - Messages in dead letter queue

### Alerts

**PagerDuty Integration**:
```typescript
// Queue backlog > 1000
if (queueWaiting > 1000) {
  await pagerduty.trigger({
    severity: 'warning',
    summary: `High queue backlog: ${queueWaiting} jobs waiting`,
    service: 'background-workers'
  })
}

// DLQ messages > 50
if (dlqCount > 50) {
  await pagerduty.trigger({
    severity: 'critical',
    summary: `High DLQ count: ${dlqCount} failed messages`,
    service: 'event-workers'
  })
}

// Event processing lag > 5 minutes
if (lagSeconds > 300) {
  await pagerduty.trigger({
    severity: 'critical',
    summary: `Event processing lag: ${lagSeconds}s behind`,
    service: 'event-workers'
  })
}
```

---

## Scaling Strategy

### Horizontal Scaling

**Event Workers**:
```bash
# Scale to 10 workers
kubectl scale deployment event-worker --replicas=10

# Auto-scaling based on queue depth
kubectl autoscale deployment event-worker \
  --min=3 --max=10 \
  --cpu-percent=70
```

**BullMQ Workers**:
```typescript
// Increase concurrency per worker
const worker = new Worker('queue-name', processor, {
  concurrency: 20 // Process 20 jobs in parallel
})

// Or deploy more worker instances
docker-compose up notification-worker --scale=5
```

### Queue Optimization

**Batching**:
```typescript
// Instead of 100 individual jobs
for (const user of users) {
  await queue.add('send-email', { userId: user.id })
}

// Batch into single job
await queue.add('send-bulk-email', {
  userIds: users.map(u => u.id),
  batchSize: 100
})
```

---

## Related Documentation

- [Event Architecture](../architecture/events.md) - Event system
- [Multi-Channel Notifications](./notifications.md) - Notification system
- [Features Index](./README.md) - All features
