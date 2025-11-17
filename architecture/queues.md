# Queue Architecture

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Technology**: BullMQ + Redis

---

## Overview

SkillCore implements a robust queue-based background processing system using **BullMQ** with **Redis** as the message broker. This architecture enables scalable, reliable asynchronous task processing for educational workflows.

---

## Queue Infrastructure

### Technology Stack

**BullMQ**:
- Production-grade message queue for Node.js
- Built on top of Redis Streams
- Advanced features: Priority, delays, retries, rate limiting
- Worker pools for horizontal scaling
- Dashboard UI for monitoring

**Redis Configuration**:
- **Version**: Redis 6.2+ (Redis Streams required)
- **Connection**: Shared Redis instance with EventBus
- **Persistence**: AOF (Append-Only File) for durability
- **Memory**: Eviction policy set to `noeviction` for queue data
- **High Availability**: Redis Sentinel for failover (production)

### Architecture Pattern

```
API Request → Queue Producer → Redis Queue → Worker Pool → Job Processing → Completion/Failure
                                    ↓
                              Job Persistence
                              (Redis Streams)
                                    ↓
                              Retry Logic
                              (Exponential Backoff)
                                    ↓
                              Dead Letter Queue
                              (Failed Jobs)
```

---

## Production Queues (7 Total)

### 1. notification-delivery Queue

**Purpose**: Multi-channel notification dispatching (email, SMS, push)

**Job Types**:
- `send-email` - SendGrid email delivery
- `send-sms` - Twilio SMS delivery
- `send-push` - Firebase FCM push notifications
- `batch-email` - Bulk email campaigns
- `batch-sms` - Bulk SMS alerts

**Configuration**:
```typescript
{
  name: 'notification-delivery',
  concurrency: 10,           // Process 10 jobs concurrently
  limiter: {
    max: 100,                // Max 100 jobs per duration
    duration: 1000,          // 1 second (rate limiting)
  },
  defaultJobOptions: {
    attempts: 5,             // Retry 5 times
    backoff: {
      type: 'exponential',
      delay: 2000,           // Start with 2s delay
    },
    removeOnComplete: 100,   // Keep last 100 completed
    removeOnFail: 1000,      // Keep last 1000 failed
  },
}
```

**Job Data Schema**:
```typescript
interface EmailJobData {
  to: string | string[]
  subject: string
  templateId: string
  templateData: Record<string, any>
  priority?: 'high' | 'normal' | 'low'
  tenantId: string
  userId: string
}

interface SMSJobData {
  to: string
  message: string
  priority?: 'high' | 'normal' | 'low'
  tenantId: string
  userId: string
}

interface PushJobData {
  tokens: string[]
  title: string
  body: string
  data?: Record<string, string>
  priority?: 'high' | 'normal' | 'low'
  tenantId: string
  userId: string
}
```

**Priority Levels**:
- **High (1)**: Emergency alerts, attendance emergencies, security
- **Normal (5)**: Grade postings, assignment due reminders, announcements
- **Low (10)**: Newsletters, weekly digests, non-urgent updates

**SLA Targets**:
- High priority: < 30 seconds
- Normal priority: < 5 minutes
- Low priority: < 30 minutes

---

### 2. grade-processing Queue

**Purpose**: Grade calculations, GPA updates, transcript generation

**Job Types**:
- `calculate-grade-average` - Weighted grade calculations
- `update-gpa` - Student GPA recalculation
- `recalculate-class-average` - Class performance metrics
- `generate-transcript` - Official transcript generation
- `update-grade-trends` - Grade trend analysis

**Configuration**:
```typescript
{
  name: 'grade-processing',
  concurrency: 5,            // CPU-intensive, limit concurrency
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
    timeout: 30000,          // 30s timeout for complex calculations
  },
}
```

**Job Data Schema**:
```typescript
interface GradeCalculationJobData {
  studentId: string
  classId: string
  gradingPeriodId: string
  tenantId: string
  triggeredBy: 'grade_update' | 'weight_change' | 'manual'
}

interface GPAUpdateJobData {
  studentId: string
  academicSessionId: string
  tenantId: string
}
```

**Processing Flow**:
1. Fetch all grades for student/class/period
2. Apply weighting rules (category weights, extra credit)
3. Calculate weighted average
4. Convert to letter grade
5. Update database
6. Emit `GradeAverageCalculated` event
7. Trigger parent/student notifications

---

### 3. attendance-aggregation Queue

**Purpose**: Daily attendance rollups, rate calculations, intervention detection

**Job Types**:
- `daily-attendance-aggregation` - Daily rollup by student/class
- `calculate-attendance-rate` - YTD attendance percentage
- `detect-chronic-absence` - Identify students missing 10%+ days
- `generate-attendance-report` - Monthly reports for admin

**Configuration**:
```typescript
{
  name: 'attendance-aggregation',
  concurrency: 3,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'fixed',
      delay: 5000,           // 5s fixed delay between retries
    },
  },
}
```

**Job Data Schema**:
```typescript
interface AttendanceAggregationJobData {
  date: string               // ISO date (YYYY-MM-DD)
  schoolId?: string          // Optional: specific school
  tenantId: string
}

interface ChronicAbsenceDetectionJobData {
  schoolId: string
  academicSessionId: string
  threshold: number          // Default: 0.10 (10% absence rate)
  tenantId: string
}
```

**Scheduled Triggers**:
- **Daily at 6:00 PM**: Aggregate attendance for current day
- **Daily at 7:00 PM**: Calculate attendance rates
- **Daily at 8:00 PM**: Detect chronic absence and trigger interventions

---

### 4. report-generation Queue

**Purpose**: Generate PDF/Excel reports for teachers, parents, admins

**Job Types**:
- `generate-progress-report` - Student progress reports (PDF)
- `generate-transcript` - Official transcripts (PDF)
- `generate-grade-book` - Gradebook export (Excel)
- `generate-attendance-report` - Attendance reports (PDF/Excel)
- `generate-iep-report` - IEP documentation (PDF)
- `generate-analytics-dashboard` - Data exports for analytics

**Configuration**:
```typescript
{
  name: 'report-generation',
  concurrency: 2,            // Memory-intensive, limit concurrency
  defaultJobOptions: {
    attempts: 2,             // Fail fast for reports
    timeout: 120000,         // 2 minute timeout
    removeOnComplete: 50,    // Keep fewer completed jobs (large payloads)
  },
}
```

**Job Data Schema**:
```typescript
interface ReportGenerationJobData {
  reportType: 'progress' | 'transcript' | 'gradebook' | 'attendance' | 'iep' | 'analytics'
  format: 'pdf' | 'excel' | 'csv'
  parameters: {
    studentId?: string
    classId?: string
    schoolId?: string
    dateRange?: { start: string; end: string }
    academicSessionId?: string
  }
  requestedBy: string        // User ID
  tenantId: string
  outputUrl?: string         // S3 URL to store generated report
}
```

**Processing Flow**:
1. Fetch data based on report type and parameters
2. Apply business logic (grade calculations, aggregations)
3. Render report using template (Handlebars/React PDF)
4. Generate PDF/Excel file
5. Upload to S3 storage
6. Return signed URL with 24-hour expiration
7. Notify user via email/push notification

**Storage**:
- S3 bucket: `skillcore-reports-{env}`
- Path structure: `{tenantId}/{reportType}/{year}/{month}/{reportId}.pdf`
- Retention: 90 days, then auto-delete

---

### 5. data-archival Queue

**Purpose**: Archive old data, cleanup inactive records, FERPA compliance

**Job Types**:
- `archive-old-grades` - Archive grades older than 7 years
- `archive-old-attendance` - Archive attendance older than 7 years
- `cleanup-inactive-users` - Deactivate users inactive > 2 years
- `cleanup-old-notifications` - Delete read notifications > 90 days
- `export-student-data` - FERPA data export requests
- `delete-student-data` - FERPA right-to-deletion requests

**Configuration**:
```typescript
{
  name: 'data-archival',
  concurrency: 1,            // Sequential processing to avoid conflicts
  defaultJobOptions: {
    attempts: 2,
    timeout: 600000,         // 10 minute timeout for large datasets
  },
}
```

**Job Data Schema**:
```typescript
interface DataArchivalJobData {
  archivalType: 'grades' | 'attendance' | 'users' | 'notifications' | 'student-data'
  operation: 'archive' | 'export' | 'delete'
  parameters: {
    olderThan?: string       // ISO date
    studentId?: string       // For FERPA requests
    schoolId?: string
  }
  requestedBy: string        // User ID (for audit log)
  tenantId: string
}
```

**FERPA Compliance**:
- All archival operations logged in `AuditLog` table
- Student data exports encrypted (AES-256)
- Deletions are soft deletes (data anonymized, not purged)
- Parent/guardian consent required for student data operations

---

### 6. intervention-detection Queue

**Purpose**: Detect at-risk students, trigger interventions, MTSS/RTI workflows

**Job Types**:
- `detect-grade-drop` - Identify students with significant grade drops
- `detect-chronic-absence` - Attendance rate < 90%
- `detect-failing-students` - Students with F grades
- `detect-behavioral-issues` - Discipline incident patterns
- `recommend-interventions` - AI-powered intervention recommendations
- `escalate-interventions` - Move students to higher tier

**Configuration**:
```typescript
{
  name: 'intervention-detection',
  concurrency: 3,
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
  },
}
```

**Job Data Schema**:
```typescript
interface InterventionDetectionJobData {
  detectionType: 'grade-drop' | 'chronic-absence' | 'failing' | 'behavioral'
  parameters: {
    schoolId?: string
    gradeLevel?: string
    classId?: string
    threshold?: number       // Custom threshold for detection
  }
  academicSessionId: string
  tenantId: string
}
```

**Detection Rules**:
- **Grade Drop**: Current grade 20+ points below previous period
- **Chronic Absence**: Attendance rate < 90% over 30 days
- **Failing Students**: Grade < 60% in any class
- **Behavioral**: 3+ discipline incidents in 30 days

**Auto-Actions**:
1. Detect at-risk students using rules
2. Check if intervention already exists
3. Create Tier 1 intervention if none exists
4. Notify counselor/teacher via email + push
5. Schedule intervention team meeting
6. Log action in `InterventionLog` table

---

### 7. compliance-reporting Queue

**Purpose**: Generate compliance reports for FERPA, IDEA, Section 504, state regulations

**Job Types**:
- `generate-ferpa-audit-log` - FERPA access audit reports
- `generate-idea-compliance` - Special education compliance
- `generate-504-compliance` - Section 504 compliance
- `generate-state-reporting` - State-mandated reports (varies by state)
- `generate-equity-report` - Discipline/achievement gap analysis

**Configuration**:
```typescript
{
  name: 'compliance-reporting',
  concurrency: 1,            // Sequential to ensure data consistency
  defaultJobOptions: {
    attempts: 2,
    timeout: 300000,         // 5 minute timeout
    priority: 1,             // High priority queue
  },
}
```

**Job Data Schema**:
```typescript
interface ComplianceReportJobData {
  reportType: 'ferpa-audit' | 'idea-compliance' | '504-compliance' | 'state-reporting' | 'equity-analysis'
  parameters: {
    schoolId?: string
    districtId?: string
    dateRange: { start: string; end: string }
    reportingPeriod?: 'monthly' | 'quarterly' | 'annual'
  }
  requestedBy: string        // User ID (must have admin role)
  tenantId: string
  outputFormat: 'pdf' | 'excel' | 'csv'
}
```

**Compliance Standards**:
- **FERPA**: All student data access logged with user, timestamp, purpose
- **IDEA**: Special education service hours, IEP compliance, evaluations
- **Section 504**: Accommodation implementation, review timelines
- **State Reporting**: Varies by state (attendance, discipline, assessment data)
- **Equity**: Disaggregated data by race, gender, socioeconomic status

**Scheduled Reports**:
- **Monthly**: FERPA audit logs, equity dashboards
- **Quarterly**: IDEA/504 compliance summaries
- **Annual**: State reporting data exports, comprehensive compliance audit

---

## Queue Monitoring & Observability

### BullMQ Dashboard

**Access**: `https://admin.skillcore.com/queues` (System Admin role only)

**Features**:
- Real-time queue metrics (waiting, active, completed, failed)
- Job details (data, progress, logs, stacktrace)
- Retry failed jobs manually
- Pause/resume queues
- Clean old jobs
- View worker status

**Metrics**:
- Jobs processed per minute
- Average processing time
- Success rate (%)
- Failure rate (%)
- Queue depth (waiting jobs)
- Worker utilization (%)

### Prometheus Metrics

**Exported Metrics**:
```
# Queue depth
bullmq_queue_waiting_total{queue="notification-delivery"} 23
bullmq_queue_active_total{queue="notification-delivery"} 5
bullmq_queue_completed_total{queue="notification-delivery"} 14532
bullmq_queue_failed_total{queue="notification-delivery"} 42

# Processing time
bullmq_job_duration_seconds{queue="notification-delivery",job="send-email"} 1.234

# Worker metrics
bullmq_worker_active_total{queue="notification-delivery"} 3
bullmq_worker_idle_total{queue="notification-delivery"} 2
```

**Grafana Dashboards**:
- **Queue Overview**: All queues at a glance
- **Queue Detail**: Deep dive into specific queue
- **Worker Performance**: Worker utilization, job throughput
- **SLA Compliance**: Processing time vs. SLA targets

### Alerting Rules

**PagerDuty Alerts**:
- Queue depth > 1000 for > 5 minutes → **Critical**
- Failure rate > 10% for > 10 minutes → **Warning**
- Processing time > SLA for > 5 minutes → **Warning**
- All workers idle for > 2 minutes → **Critical**
- DLQ jobs > 100 → **Warning**

**Slack Alerts**:
- New DLQ job added → #eng-alerts
- Queue paused/resumed → #eng-alerts
- Worker crash/restart → #eng-alerts

---

## Worker Architecture

### Worker Deployment

**Standalone Processes**:
```bash
# Start all queue workers
pnpm worker:queues

# Start specific queue worker
pnpm worker:notification-delivery
pnpm worker:grade-processing

# Development mode with hot reload
pnpm worker:queues:dev

# Production deployment (Docker)
docker-compose up queue-workers --scale queue-workers=5
```

**Worker Configuration** (`src/workers/queue-workers.ts`):
```typescript
import { Worker } from 'bullmq'
import Redis from 'ioredis'

const connection = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT),
  maxRetriesPerRequest: null,
})

// Notification delivery worker
const notificationWorker = new Worker(
  'notification-delivery',
  async (job) => {
    const { type } = job.data
    
    if (type === 'send-email') {
      return await sendEmailJob(job)
    } else if (type === 'send-sms') {
      return await sendSMSJob(job)
    } else if (type === 'send-push') {
      return await sendPushJob(job)
    }
  },
  {
    connection,
    concurrency: 10,
    limiter: {
      max: 100,
      duration: 1000,
    },
  }
)

// Error handling
notificationWorker.on('failed', (job, err) => {
  logger.error('Job failed', {
    jobId: job?.id,
    queue: 'notification-delivery',
    error: err.message,
    stackTrace: err.stack,
  })
})

// Graceful shutdown
process.on('SIGTERM', async () => {
  await notificationWorker.close()
  process.exit(0)
})
```

### Horizontal Scaling

**Multiple Worker Instances**:
- Each queue can have multiple workers processing concurrently
- BullMQ automatically distributes jobs across workers
- Workers consume from same Redis queue (no duplication)
- Scale workers independently based on queue depth

**Auto-Scaling Rules**:
- If queue depth > 500 → Scale to 5 workers
- If queue depth > 1000 → Scale to 10 workers
- If queue depth < 100 for 10 min → Scale down to 2 workers

**Kubernetes Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: notification-worker
  template:
    metadata:
      labels:
        app: notification-worker
    spec:
      containers:
      - name: worker
        image: skillcore/queue-worker:latest
        env:
        - name: QUEUE_NAME
          value: "notification-delivery"
        - name: REDIS_HOST
          value: "redis-master"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## Retry & Error Handling

### Retry Strategy

**Exponential Backoff**:
```typescript
{
  attempts: 5,               // Max 5 retry attempts
  backoff: {
    type: 'exponential',
    delay: 2000,             // Initial delay: 2 seconds
  },
}

// Retry delays:
// Attempt 1: 2s
// Attempt 2: 4s
// Attempt 3: 8s
// Attempt 4: 16s
// Attempt 5: 32s
// Total max delay: 62s
```

**Fixed Backoff**:
```typescript
{
  attempts: 3,
  backoff: {
    type: 'fixed',
    delay: 5000,             // Fixed 5s delay between retries
  },
}
```

**Custom Backoff**:
```typescript
{
  attempts: 5,
  backoff: {
    type: 'custom',
  },
}

// Custom backoff function
function customBackoff(attemptsMade: number): number {
  // Fibonacci backoff: 1s, 2s, 3s, 5s, 8s
  const fib = [1000, 2000, 3000, 5000, 8000]
  return fib[attemptsMade - 1] || 10000
}
```

### Dead Letter Queue (DLQ)

**Automatic DLQ**:
- Jobs failing all retry attempts are moved to DLQ
- DLQ jobs stored in Redis with failure details
- Manual review required for DLQ jobs
- Options: Retry, Edit & Retry, Discard

**DLQ Management**:
```typescript
// Get DLQ jobs
const dlqJobs = await queue.getFailed(0, 100)

// Retry DLQ job
await queue.retryJob(jobId)

// Remove DLQ job
await queue.removeJob(jobId)

// Bulk retry all DLQ jobs
const failedJobs = await queue.getFailed()
for (const job of failedJobs) {
  await job.retry()
}
```

**DLQ Alerts**:
- Email admin when job enters DLQ
- Slack notification for high-priority job failures
- Daily digest of DLQ jobs requiring attention

### Error Classification

**Retryable Errors** (Transient):
- Network timeout (SendGrid, Twilio API)
- Database connection lost
- Redis connection error
- External service rate limit (429)
- Temporary API errors (500, 502, 503)

**Non-Retryable Errors** (Permanent):
- Invalid job data (schema validation failure)
- Missing required parameters
- Permission denied (FERPA violation)
- Entity not found (student deleted)
- External service rejection (400, 404)

**Error Handling Logic**:
```typescript
try {
  await processJob(job)
} catch (error) {
  if (isRetryableError(error)) {
    throw error              // Trigger retry
  } else {
    // Log error and move to DLQ
    logger.error('Non-retryable error', { error, job })
    await job.moveToFailed(error, true)  // Skip retries
  }
}
```

---

## Job Lifecycle

### Job States

1. **Waiting**: Job added to queue, waiting for worker
2. **Active**: Worker processing job
3. **Completed**: Job finished successfully
4. **Failed**: Job failed (all retries exhausted)
5. **Delayed**: Job scheduled for future processing
6. **Paused**: Queue paused, job waiting

### Job Events

**Job-Level Events**:
```typescript
job.on('progress', (progress) => {
  console.log(`Job ${job.id} progress: ${progress}%`)
})

job.on('completed', (result) => {
  console.log(`Job ${job.id} completed with result:`, result)
})

job.on('failed', (error) => {
  console.error(`Job ${job.id} failed:`, error)
})
```

**Queue-Level Events**:
```typescript
queue.on('waiting', (jobId) => {
  logger.info('Job waiting', { jobId })
})

queue.on('active', (job) => {
  logger.info('Job active', { jobId: job.id, data: job.data })
})

queue.on('completed', (job, result) => {
  logger.info('Job completed', { jobId: job.id, result })
})

queue.on('failed', (job, error) => {
  logger.error('Job failed', { jobId: job?.id, error: error.message })
})

queue.on('stalled', (jobId) => {
  logger.warn('Job stalled', { jobId })  // Worker crashed during processing
})
```

### Job Progress Reporting

**Progress Updates**:
```typescript
async function generateReportJob(job: Job) {
  // Step 1: Fetch data (20%)
  await job.updateProgress(20)
  const data = await fetchReportData(job.data)
  
  // Step 2: Process data (50%)
  await job.updateProgress(50)
  const processedData = await processData(data)
  
  // Step 3: Generate PDF (80%)
  await job.updateProgress(80)
  const pdf = await generatePDF(processedData)
  
  // Step 4: Upload to S3 (100%)
  await job.updateProgress(100)
  const url = await uploadToS3(pdf)
  
  return { url }
}
```

**Progress Tracking in UI**:
- Real-time progress bar for long-running jobs
- WebSocket subscription for progress updates
- Estimated time remaining calculation

---

## Best Practices

### Queue Design
1. **Single Responsibility**: Each queue has one clear purpose
2. **Idempotent Jobs**: Jobs can be retried safely without side effects
3. **Job Data Validation**: Validate job data before queuing
4. **Timeout Configuration**: Set appropriate timeouts for each job type
5. **Cleanup Strategy**: Auto-remove old completed/failed jobs

### Performance Optimization
1. **Concurrency Tuning**: Adjust based on CPU/memory constraints
2. **Rate Limiting**: Prevent overwhelming external services
3. **Batch Processing**: Group related jobs when possible
4. **Connection Pooling**: Reuse database connections
5. **Caching**: Cache frequently accessed data

### Monitoring & Alerting
1. **Queue Metrics**: Track depth, processing time, success/failure rates
2. **SLA Monitoring**: Alert when jobs exceed SLA targets
3. **DLQ Management**: Regular review of failed jobs
4. **Worker Health**: Monitor worker process uptime
5. **Resource Usage**: Track memory/CPU per worker

### FERPA Compliance
1. **Audit Logging**: Log all jobs accessing student data
2. **Data Encryption**: Encrypt sensitive job data at rest
3. **Access Control**: Verify permissions before processing
4. **Data Retention**: Auto-delete old job data per policy
5. **Privacy by Design**: Minimize student data in job payloads

---

## Related Documentation

- [Events Architecture](./events.md) - Event-driven system
- [Scheduled Jobs](./scheduled-jobs.md) - Cron-based scheduling
- [Notifications](./notifications.md) - Multi-channel notification system
- [System Admin Portal](../roles/system-admin.md) - Queue management UI

---

**Production Metrics** (as of November 2025):
- **Total Queues**: 7
- **Daily Jobs Processed**: ~50,000
- **Average Processing Time**: 1.2 seconds
- **Success Rate**: 99.2%
- **DLQ Jobs (30 days)**: 42
- **Worker Instances**: 15 (across all queues)
