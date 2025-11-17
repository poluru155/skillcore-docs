# Scheduled Jobs Architecture

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Technology**: Node-cron + BullMQ Repeat Jobs

---

## Overview

SkillCore implements comprehensive scheduled job processing using **node-cron** for lightweight tasks and **BullMQ repeat jobs** for complex, queued operations. This architecture enables reliable, time-based automation for educational workflows.

---

## Scheduling Technologies

### Node-Cron

**Use Cases**:
- Lightweight, time-based tasks
- System maintenance operations
- Metric aggregations
- Health checks

**Implementation**:
```typescript
import cron from 'node-cron'

// Daily at 2:00 AM - Stream cleanup
cron.schedule('0 2 * * *', async () => {
  await cleanupOldStreams()
})
```

### BullMQ Repeat Jobs

**Use Cases**:
- Heavy, resource-intensive tasks
- Jobs requiring retry logic
- Tasks needing progress tracking
- Jobs with variable execution times

**Implementation**:
```typescript
import { Queue } from 'bullmq'

const queue = new Queue('attendance-aggregation', { connection })

// Add repeatable job - Daily at 6:00 PM
await queue.add(
  'daily-aggregation',
  { date: new Date().toISOString() },
  {
    repeat: {
      pattern: '0 18 * * *',  // Cron expression
      tz: 'America/New_York', // Timezone-aware
    },
  }
)
```

---

## Daily Scheduled Jobs

### 1. Stream Cleanup (2:00 AM)

**Purpose**: Redis EventBus stream retention management

**Technology**: Node-cron

**Schedule**: `0 2 * * *` (Daily at 2:00 AM)

**Implementation** (`src/cron/event-metrics-cron.ts`):
```typescript
import cron from 'node-cron'
import { eventBus } from '../infrastructure/event-bus'

// Stream cleanup cron
cron.schedule('0 2 * * *', async () => {
  const logger = console
  logger.info('Starting stream cleanup cron job')
  
  try {
    const RETENTION_DAYS = 30
    const cutoffDate = new Date()
    cutoffDate.setDate(cutoffDate.getDate() - RETENTION_DAYS)
    
    // Get all event streams
    const streams = await eventBus.getAllStreams()
    
    for (const stream of streams) {
      // Trim stream to last 30 days
      await eventBus.trimStream(stream, cutoffDate)
      logger.info(`Trimmed stream: ${stream}`)
    }
    
    logger.info('Stream cleanup completed successfully')
  } catch (error) {
    logger.error('Stream cleanup failed:', error)
  }
})
```

**Retention Policy**:
- **EventBus Streams**: 30 days
- **EventStore (PostgreSQL)**: 7 years (FERPA requirement)
- **Audit Logs**: 7 years (FERPA requirement)

**Metrics**:
- Streams cleaned per day: ~50
- Events trimmed per day: ~500,000
- Execution time: 2-5 minutes
- Redis memory freed: ~100-200 MB

---

### 2. Attendance Interventions (6:30 PM)

**Purpose**: Detect students with chronic absence and create interventions

**Technology**: BullMQ Repeat Job

**Schedule**: `30 18 * * *` (Daily at 6:30 PM)

**Queue**: `intervention-detection`

**Job Implementation**:
```typescript
// Add repeatable job
await interventionQueue.add(
  'detect-chronic-absence',
  {
    detectionType: 'chronic-absence',
    threshold: 0.10,  // 10% absence rate
  },
  {
    repeat: {
      pattern: '30 18 * * *',
      tz: 'America/New_York',
    },
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,
    },
  }
)

// Job processor
async function detectChronicAbsence(job: Job) {
  const { threshold } = job.data
  
  // Get all active students
  const students = await prisma.student.findMany({
    where: { isActive: true },
    include: { attendanceRecords: true },
  })
  
  const atRiskStudents = []
  
  for (const student of students) {
    const attendanceRate = calculateAttendanceRate(student)
    
    if (attendanceRate < (1 - threshold)) {
      atRiskStudents.push({
        studentId: student.id,
        attendanceRate,
        absenceDays: student.attendanceRecords.filter(r => r.status === 'ABSENT').length,
      })
    }
  }
  
  // Create interventions for at-risk students
  for (const student of atRiskStudents) {
    // Check if intervention already exists
    const existingIntervention = await prisma.intervention.findFirst({
      where: {
        studentId: student.studentId,
        type: 'ATTENDANCE',
        status: 'ACTIVE',
      },
    })
    
    if (!existingIntervention) {
      // Create Tier 1 intervention
      await createIntervention({
        studentId: student.studentId,
        type: 'ATTENDANCE',
        tier: 1,
        reason: `Chronic absence detected: ${student.attendanceRate.toFixed(1)}% attendance rate`,
        recommendedStrategies: [
          'Parent conference',
          'Attendance contract',
          'Transportation support',
        ],
      })
      
      // Notify counselor
      await notificationQueue.add('send-email', {
        to: student.counselor.email,
        subject: 'Attendance Intervention Created',
        templateId: 'intervention-created',
        templateData: {
          studentName: student.name,
          attendanceRate: student.attendanceRate,
          absenceDays: student.absenceDays,
        },
      })
    }
  }
  
  return {
    totalStudents: students.length,
    atRiskStudents: atRiskStudents.length,
    interventionsCreated: atRiskStudents.filter(s => !s.existingIntervention).length,
  }
}
```

**Detection Criteria**:
- Attendance rate < 90% over last 30 days
- 10+ absences in current academic session
- 3+ consecutive absences

**Auto-Actions**:
- Create Tier 1 intervention (if none exists)
- Notify counselor via email + push
- Schedule parent conference
- Add to counselor's priority list

---

### 3. IEP Review Reminders (7:00 AM)

**Purpose**: Send reminders for upcoming IEP annual reviews

**Technology**: BullMQ Repeat Job

**Schedule**: `0 7 * * *` (Daily at 7:00 AM)

**Queue**: `notification-delivery`

**Reminder Timelines**:
- **90 days before**: Initial reminder to special ed coordinator
- **60 days before**: Reminder to case manager + principal
- **30 days before**: Urgent reminder to all team members
- **14 days before**: Final reminder + schedule meeting
- **7 days before**: Meeting confirmation
- **Day of review**: Meeting day reminder

**Job Implementation**:
```typescript
async function sendIEPReviewReminders(job: Job) {
  const today = new Date()
  
  // Find IEPs with upcoming annual reviews
  const ieps = await prisma.iEP.findMany({
    where: {
      status: 'ACTIVE',
      nextAnnualReview: {
        gte: today,
        lte: addDays(today, 90),  // Next 90 days
      },
    },
    include: {
      student: true,
      caseManager: true,
      teamMembers: true,
    },
  })
  
  for (const iep of ieps) {
    const daysUntilReview = differenceInDays(iep.nextAnnualReview, today)
    
    if ([90, 60, 30, 14, 7, 0].includes(daysUntilReview)) {
      // Determine recipients based on timeline
      const recipients = getReviewReminderRecipients(daysUntilReview, iep)
      
      for (const recipient of recipients) {
        await notificationQueue.add('send-email', {
          to: recipient.email,
          subject: `IEP Annual Review ${daysUntilReview === 0 ? 'Today' : `in ${daysUntilReview} Days`}`,
          templateId: 'iep-review-reminder',
          templateData: {
            studentName: iep.student.name,
            daysUntilReview,
            reviewDate: iep.nextAnnualReview,
            recipientRole: recipient.role,
          },
          priority: daysUntilReview <= 14 ? 'high' : 'normal',
        })
      }
      
      // Update reminder sent timestamp
      await prisma.iEP.update({
        where: { id: iep.id },
        data: {
          lastReminderSent: new Date(),
          remindersSentCount: { increment: 1 },
        },
      })
    }
  }
  
  return {
    iepsChecked: ieps.length,
    remindersSent: recipients.length,
  }
}
```

**IDEA Compliance**:
- Annual review must occur within 365 days
- Parent must be notified 10 days before meeting
- All team members must be invited
- Meeting minutes must be documented

---

### 4. Grade Posting Digest (8:00 PM)

**Purpose**: Send daily digest of new grades posted to students/parents

**Technology**: BullMQ Repeat Job

**Schedule**: `0 20 * * *` (Daily at 8:00 PM)

**Queue**: `notification-delivery`

**Job Implementation**:
```typescript
async function sendGradePostingDigest(job: Job) {
  const today = new Date()
  const yesterday = subDays(today, 1)
  
  // Find grades posted in last 24 hours
  const postedGrades = await prisma.grade.findMany({
    where: {
      postedAt: {
        gte: yesterday,
        lt: today,
      },
    },
    include: {
      student: {
        include: {
          parents: true,
        },
      },
      assignment: {
        include: {
          class: true,
        },
      },
    },
  })
  
  // Group by student
  const gradesByStudent = groupBy(postedGrades, 'studentId')
  
  for (const [studentId, grades] of Object.entries(gradesByStudent)) {
    const student = grades[0].student
    
    // Send to student
    await notificationQueue.add('send-email', {
      to: student.email,
      subject: `${grades.length} New Grade${grades.length > 1 ? 's' : ''} Posted`,
      templateId: 'grade-digest-student',
      templateData: {
        studentName: student.name,
        grades: grades.map(g => ({
          className: g.assignment.class.name,
          assignmentName: g.assignment.name,
          score: g.score,
          maxScore: g.assignment.maxScore,
          percentage: (g.score / g.assignment.maxScore) * 100,
        })),
      },
    })
    
    // Send to parents
    for (const parent of student.parents) {
      if (parent.notificationPreferences.gradePosted) {
        await notificationQueue.add('send-email', {
          to: parent.email,
          subject: `New Grades for ${student.name}`,
          templateId: 'grade-digest-parent',
          templateData: {
            studentName: student.name,
            parentName: parent.name,
            grades: grades.map(g => ({
              className: g.assignment.class.name,
              assignmentName: g.assignment.name,
              score: g.score,
              maxScore: g.assignment.maxScore,
              percentage: (g.score / g.assignment.maxScore) * 100,
            })),
          },
        })
      }
    }
  }
  
  return {
    gradesPosted: postedGrades.length,
    studentsNotified: Object.keys(gradesByStudent).length,
    parentsNotified: Object.values(gradesByStudent).flatMap(g => g[0].student.parents).length,
  }
}
```

**Notification Preferences**:
- Parents can opt-out of daily digest
- Parents can choose immediate notification for low grades
- Students can mute notifications for specific classes

---

### 5. Missing Assignment Alerts (7:00 PM)

**Purpose**: Alert students/parents about upcoming assignment due dates

**Technology**: BullMQ Repeat Job

**Schedule**: `0 19 * * *` (Daily at 7:00 PM)

**Queue**: `notification-delivery`

**Alert Timelines**:
- **7 days before**: Reminder for major assignments
- **3 days before**: Reminder for all assignments
- **1 day before**: Urgent reminder
- **Day of**: Final reminder (morning)
- **Overdue**: Daily reminder until submitted

**Job Implementation**:
```typescript
async function sendMissingAssignmentAlerts(job: Job) {
  const today = new Date()
  const oneWeekFromNow = addDays(today, 7)
  
  // Find assignments due in next 7 days or overdue
  const upcomingAssignments = await prisma.assignment.findMany({
    where: {
      dueDate: {
        gte: today,
        lte: oneWeekFromNow,
      },
      status: 'PUBLISHED',
    },
    include: {
      class: {
        include: {
          students: true,
        },
      },
      submissions: true,
    },
  })
  
  for (const assignment of upcomingAssignments) {
    const daysUntilDue = differenceInDays(assignment.dueDate, today)
    
    // Only send reminders at specific intervals
    if (![7, 3, 1, 0].includes(daysUntilDue)) continue
    
    // Find students who haven't submitted
    const submittedStudentIds = assignment.submissions.map(s => s.studentId)
    const missingStudents = assignment.class.students.filter(
      s => !submittedStudentIds.includes(s.id)
    )
    
    for (const student of missingStudents) {
      const priority = daysUntilDue <= 1 ? 'high' : 'normal'
      
      // Send to student
      await notificationQueue.add('send-push', {
        tokens: student.fcmTokens,
        title: `Assignment Due ${daysUntilDue === 0 ? 'Today' : `in ${daysUntilDue} Day${daysUntilDue > 1 ? 's' : ''}`}`,
        body: `${assignment.class.name}: ${assignment.name}`,
        data: {
          type: 'assignment-reminder',
          assignmentId: assignment.id,
          classId: assignment.classId,
        },
        priority,
      })
      
      // Send to parents (if enabled)
      for (const parent of student.parents) {
        if (parent.notificationPreferences.assignmentDue) {
          await notificationQueue.add('send-email', {
            to: parent.email,
            subject: `Assignment Reminder: ${assignment.name}`,
            templateId: 'assignment-reminder-parent',
            templateData: {
              studentName: student.name,
              parentName: parent.name,
              className: assignment.class.name,
              assignmentName: assignment.name,
              dueDate: assignment.dueDate,
              daysUntilDue,
            },
            priority,
          })
        }
      }
    }
  }
  
  return {
    assignmentsChecked: upcomingAssignments.length,
    studentsNotified: missingStudents.length,
  }
}
```

---

## Weekly Scheduled Jobs

### 1. Performance Trend Calculation (Sunday 3:00 AM)

**Purpose**: Calculate weekly performance trends for students/classes

**Technology**: BullMQ Repeat Job

**Schedule**: `0 3 * * 0` (Sunday at 3:00 AM)

**Queue**: `grade-processing`

**Metrics Calculated**:
- Grade trends (up/down/stable)
- Attendance trends
- Assignment completion rate trends
- Behavior incident trends
- Intervention effectiveness

**Job Implementation**:
```typescript
async function calculatePerformanceTrends(job: Job) {
  const thisWeek = startOfWeek(new Date())
  const lastWeek = subWeeks(thisWeek, 1)
  const twoWeeksAgo = subWeeks(thisWeek, 2)
  
  // Calculate student trends
  const students = await prisma.student.findMany({
    where: { isActive: true },
    include: {
      grades: {
        where: {
          postedAt: { gte: twoWeeksAgo },
        },
      },
      attendanceRecords: {
        where: {
          date: { gte: twoWeeksAgo },
        },
      },
    },
  })
  
  for (const student of students) {
    // Grade trend
    const lastWeekGrades = student.grades.filter(g => 
      g.postedAt >= lastWeek && g.postedAt < thisWeek
    )
    const thisWeekGrades = student.grades.filter(g => 
      g.postedAt >= thisWeek
    )
    
    const lastWeekAvg = average(lastWeekGrades.map(g => g.score))
    const thisWeekAvg = average(thisWeekGrades.map(g => g.score))
    
    const gradeTrend = thisWeekAvg - lastWeekAvg
    
    // Attendance trend
    const lastWeekAttendance = student.attendanceRecords.filter(r =>
      r.date >= lastWeek && r.date < thisWeek
    )
    const thisWeekAttendance = student.attendanceRecords.filter(r =>
      r.date >= thisWeek
    )
    
    const lastWeekAttendanceRate = calculateRate(lastWeekAttendance)
    const thisWeekAttendanceRate = calculateRate(thisWeekAttendance)
    
    const attendanceTrend = thisWeekAttendanceRate - lastWeekAttendanceRate
    
    // Update student analytics
    await prisma.studentAnalytics.upsert({
      where: { studentId: student.id },
      update: {
        gradeTrend,
        gradeTrendDirection: gradeTrend > 5 ? 'UP' : gradeTrend < -5 ? 'DOWN' : 'STABLE',
        attendanceTrend,
        attendanceTrendDirection: attendanceTrend > 5 ? 'UP' : attendanceTrend < -5 ? 'DOWN' : 'STABLE',
        lastCalculated: new Date(),
      },
      create: {
        studentId: student.id,
        gradeTrend,
        gradeTrendDirection: gradeTrend > 5 ? 'UP' : gradeTrend < -5 ? 'DOWN' : 'STABLE',
        attendanceTrend,
        attendanceTrendDirection: attendanceTrend > 5 ? 'UP' : attendanceTrend < -5 ? 'DOWN' : 'STABLE',
      },
    })
    
    // Alert counselor if significant drop
    if (gradeTrend < -10 || attendanceTrend < -10) {
      await notificationQueue.add('send-email', {
        to: student.counselor.email,
        subject: `Performance Alert: ${student.name}`,
        templateId: 'performance-alert',
        templateData: {
          studentName: student.name,
          gradeTrend,
          attendanceTrend,
        },
        priority: 'high',
      })
    }
  }
  
  return {
    studentsAnalyzed: students.length,
    alertsSent: alertCount,
  }
}
```

---

### 2. Weekly Analytics Report (Sunday 6:00 AM)

**Purpose**: Generate weekly analytics reports for principals/district admins

**Technology**: BullMQ Repeat Job

**Schedule**: `0 6 * * 0` (Sunday at 6:00 AM)

**Queue**: `report-generation`

**Report Includes**:
- School-wide grade averages
- Attendance rates by school/grade level
- Intervention summary (created, closed, escalated)
- Discipline incidents
- Special programs enrollment

---

### 3. Inactive User Cleanup (Sunday 2:00 AM)

**Purpose**: Deactivate users inactive for 2+ years

**Technology**: Node-cron

**Schedule**: `0 2 * * 0` (Sunday at 2:00 AM)

**Retention Policy**:
- Teachers: Inactive 2+ years → Deactivate
- Students: Graduated/transferred 2+ years → Archive
- Parents: All children graduated 2+ years → Deactivate
- Staff: Inactive 1+ year → Flag for review

---

## Monthly Scheduled Jobs

### 1. Compliance Audit Report (1st of Month, 1:00 AM)

**Purpose**: Generate monthly compliance reports for FERPA, IDEA, Section 504

**Technology**: BullMQ Repeat Job

**Schedule**: `0 1 1 * *` (1st of month at 1:00 AM)

**Queue**: `compliance-reporting`

**Reports Generated**:
- FERPA access audit log
- IDEA service hour tracking
- Section 504 accommodation compliance
- State reporting data exports

---

### 2. Data Archival (1st of Month, 3:00 AM)

**Purpose**: Archive old data to cold storage, cleanup inactive records

**Technology**: BullMQ Repeat Job

**Schedule**: `0 3 1 * *` (1st of month at 3:00 AM)

**Queue**: `data-archival`

**Archival Rules**:
- Grades older than 7 years → Archive to S3
- Attendance older than 7 years → Archive to S3
- Read notifications older than 90 days → Delete
- Completed interventions older than 3 years → Archive
- Chat messages older than 2 years → Archive

---

### 3. System Health Summary (1st of Month, 8:00 AM)

**Purpose**: Monthly system health report for IT/admins

**Technology**: BullMQ Repeat Job

**Schedule**: `0 8 1 * *` (1st of month at 8:00 AM)

**Metrics Included**:
- API uptime percentage
- Average response times
- Error rates by endpoint
- Queue processing metrics
- Database query performance
- EventBus health
- External service status (SendGrid, Twilio, Firebase)

---

## Ad-Hoc Scheduled Jobs

### 1. End-of-Year Rollover (Configurable)

**Purpose**: Academic year transition processing

**Trigger**: Manually scheduled by district admin

**Tasks**:
- Archive previous year data
- Create new academic session
- Roll over student enrollments
- Update grade levels
- Archive graduated students
- Reset attendance counters
- Close all active interventions

---

### 2. Emergency Notification Test (Quarterly)

**Purpose**: Test emergency notification system

**Schedule**: Configurable (e.g., first Monday of quarter)

**Test Scenarios**:
- School closure notification
- Emergency lockdown alert
- Weather delay announcement
- System-wide broadcast

---

## Job Monitoring & Management

### System Admin Dashboard

**Access**: `/admin/scheduled-jobs` (System Admin role only)

**Features**:
- View all scheduled jobs (cron + repeat jobs)
- Manually trigger job execution
- Pause/resume scheduled jobs
- View job execution history
- View job logs and errors
- Update cron schedules
- Delete scheduled jobs

**Metrics Displayed**:
- Next execution time
- Last execution time
- Last execution status (success/failure)
- Average execution time
- Execution count (last 30 days)

### Prometheus Metrics

```
# Scheduled job executions
scheduled_job_executions_total{job="stream-cleanup"} 30
scheduled_job_failures_total{job="stream-cleanup"} 0
scheduled_job_duration_seconds{job="stream-cleanup"} 180

# Repeat job metrics (BullMQ)
bullmq_repeat_job_executions_total{job="attendance-interventions"} 30
bullmq_repeat_job_next_execution_seconds{job="attendance-interventions"} 3600
```

### Alerting

**PagerDuty Alerts**:
- Critical job failure (e.g., stream cleanup, compliance reporting)
- Job execution timeout
- Job skipped (scheduler not running)

**Slack Alerts**:
- Job execution started/completed
- Job warnings (non-critical failures)
- Long-running job (> expected time)

---

## Best Practices

### Job Design
1. **Idempotent**: Jobs can run multiple times safely
2. **Resumable**: Jobs can recover from crashes
3. **Timed Appropriately**: Run during low-traffic hours
4. **Logged Comprehensively**: All actions logged for audit
5. **Error-Tolerant**: Handle failures gracefully

### Timezone Management
1. **Consistent Timezone**: All crons use school/district timezone
2. **DST Awareness**: Handle daylight saving time transitions
3. **Multi-Timezone**: Support districts across timezones

### Performance
1. **Batch Processing**: Process data in batches to avoid memory issues
2. **Database Optimization**: Use indexes, avoid N+1 queries
3. **Connection Pooling**: Reuse database connections
4. **Progress Tracking**: Log progress for long-running jobs

### FERPA Compliance
1. **Audit Logging**: Log all data access in scheduled jobs
2. **Data Minimization**: Only access necessary student data
3. **Secure Processing**: Encrypt data at rest and in transit
4. **Retention Compliance**: Follow data retention policies

---

## Related Documentation

- [Queue Architecture](./queues.md) - BullMQ queue system
- [Events Architecture](./events.md) - Event-driven system
- [System Admin Portal](../roles/system-admin.md) - Job management UI

---

**Production Jobs** (as of November 2025):
- **Total Daily Jobs**: 5
- **Total Weekly Jobs**: 3
- **Total Monthly Jobs**: 3
- **Average Daily Executions**: 15
- **Success Rate**: 99.8%
- **Average Execution Time**: 3.2 minutes
