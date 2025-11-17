# Attendance System

**Last Updated**: November 16, 2025  
**Status**: ✅ Production-Ready

---

## Overview

SkillCore WebApp provides a comprehensive attendance tracking system with real-time entry, automated interventions, parent notifications, and compliance reporting for federal and state requirements.

---

## Features

### Attendance Codes

**Standard Codes**:
```typescript
enum AttendanceStatus {
  PRESENT = 'P',
  ABSENT = 'A',
  TARDY = 'T',
  EXCUSED_ABSENT = 'EA',
  EXCUSED_TARDY = 'ET',
  UNEXCUSED_ABSENT = 'UA',
  UNEXCUSED_TARDY = 'UT',
  HEALTH_OFFICE = 'HO',
  FIELD_TRIP = 'FT',
  SUSPENSION = 'S',
  EARLY_DISMISSAL = 'ED'
}
```

**Custom Codes** (district-configurable):
```typescript
interface CustomAttendanceCode {
  id: string
  code: string          // "ISS" (In-School Suspension)
  label: string
  countsAsAbsent: boolean
  countsTowardTruancy: boolean
  requiresNote: boolean
  color: string
}
```

### Entry Methods

**1. Homeroom Attendance** (Daily, morning):
```typescript
// Teacher marks entire homeroom
async function markHomeroomAttendance(
  classId: string,
  date: Date,
  attendanceData: { studentId: string; status: AttendanceStatus }[]
): Promise<void> {
  for (const entry of attendanceData) {
    await db.attendanceRecord.create({
      data: {
        studentId: entry.studentId,
        classId,
        date,
        status: entry.status,
        periodNumber: 0, // Homeroom
        markedBy: currentUser.id,
        markedAt: new Date()
      }
    })

    // Publish event for each absence
    if (entry.status.includes('ABSENT')) {
      await eventBus.publish(new AttendanceMarked({
        studentId: entry.studentId,
        date,
        status: entry.status,
        classId
      }))
    }
  }
}
```

**2. Period-by-Period Attendance**:
```typescript
// Teacher marks attendance for each class period
async function markPeriodAttendance(
  classId: string,
  periodNumber: number,
  date: Date,
  attendanceData: { studentId: string; status: AttendanceStatus }[]
): Promise<void> {
  // Similar to homeroom, but includes periodNumber
}
```

**3. Bulk Import**:
```typescript
// Import from student information system
async function importAttendance(file: File): Promise<void> {
  const rows = await parseCSV(file)
  
  for (const row of rows) {
    await markAttendance({
      studentId: findStudentByNumber(row.studentNumber),
      date: parseDate(row.date),
      status: row.status as AttendanceStatus
    })
  }
}
```

**4. Mobile App Entry**:
```typescript
// Quick entry from iOS/Android app
POST /api/attendance/quick-entry
{
  "classId": "class-123",
  "presentStudents": ["student-1", "student-2"],
  "absentStudents": ["student-3"],
  "tardyStudents": ["student-4"]
}
```

### Real-Time Tracking

**Live Dashboard**:
```typescript
interface AttendanceDashboard {
  totalStudents: number
  presentCount: number
  absentCount: number
  tardyCount: number
  unmarkedCount: number
  attendanceRate: number        // Percentage
  chronicallyAbsentCount: number // 10%+ absences
}

// WebSocket subscription for real-time updates
subscription attendanceUpdates($schoolId: ID!) {
  attendanceDashboardUpdated(schoolId: $schoolId) {
    totalStudents
    presentCount
    absentCount
    attendanceRate
  }
}
```

**Absence Alerts**:
```typescript
// Real-time notifications when student absent
export class AbsentStudentHandler implements IEventHandler<AttendanceMarked> {
  async handle(event: AttendanceMarked): Promise<void> {
    if (event.status.includes('ABSENT')) {
      // Notify counselor if 3+ absences in 7 days
      const recentAbsences = await countRecentAbsences(event.studentId, 7)
      
      if (recentAbsences >= 3) {
        await notificationService.send({
          recipientRole: 'COUNSELOR',
          title: 'Student Absence Alert',
          body: `${event.studentName} has ${recentAbsences} absences in last 7 days`,
          priority: 'HIGH',
          link: `/students/${event.studentId}/attendance`
        })
      }

      // Notify parent
      await sendAbsenceNotificationToParent(event.studentId, event.date)
    }
  }
}
```

### Parent Notifications

**Daily Absence Notifications**:
```typescript
// Automated at 10:00 AM each school day
async function sendDailyAbsenceNotifications(): Promise<void> {
  const today = new Date()
  const absences = await db.attendanceRecord.findMany({
    where: {
      date: today,
      status: { in: ['A', 'UA', 'EA'] }
    },
    include: { student: { include: { parents: true } } }
  })

  for (const absence of absences) {
    for (const parent of absence.student.parents) {
      // Multi-channel notification
      await notificationQueue.add('absence-notification', {
        parentId: parent.id,
        studentId: absence.studentId,
        date: today,
        status: absence.status,
        channels: ['EMAIL', 'SMS', 'PUSH'] // Based on parent preference
      })
    }
  }
}
```

**SMS Template**:
```
SkillCore Alert: Your child [Student Name] was marked absent today ([Date]). 
If this is an error, please contact the school office. Reply STOP to opt-out.
```

**Email Template**:
```html
<h2>Absence Notification</h2>
<p>Dear [Parent Name],</p>
<p>Your child <strong>[Student Name]</strong> was marked absent on <strong>[Date]</strong>.</p>
<p><strong>Status:</strong> [Excused/Unexcused]</p>
<p>If this absence was excused, please submit documentation to the school office.</p>
<p>Current attendance rate: <strong>[XX%]</strong></p>
<a href="[Portal Link]">View Full Attendance Record</a>
```

### Intervention Triggers

**Automatic Intervention Creation**:
```typescript
export class AttendanceInterventionHandler implements IEventHandler<DailyAttendanceAggregated> {
  async handle(event: DailyAttendanceAggregated): Promise<void> {
    for (const student of event.students) {
      // Tier 1: 3-5 absences (10% of days)
      if (student.absenceCount >= 3 && student.absenceCount <= 5) {
        if (!student.hasTier1Intervention) {
          await createIntervention({
            studentId: student.id,
            type: 'attendance',
            tier: 1,
            strategies: [
              'Parent phone call',
              'Attendance contract',
              'Check-in/check-out system'
            ],
            goal: 'Improve attendance to 95% or higher',
            reviewDate: addDays(new Date(), 30)
          })
        }
      }

      // Tier 2: 6-9 absences (20% of days)
      if (student.absenceCount >= 6 && student.absenceCount <= 9) {
        if (!student.hasTier2Intervention) {
          await createIntervention({
            studentId: student.id,
            type: 'attendance',
            tier: 2,
            strategies: [
              'Parent conference',
              'Mentoring program',
              'Incentive plan',
              'Community agency referral'
            ],
            goal: 'Reduce absences below 10%',
            reviewDate: addDays(new Date(), 21)
          })
        }
      }

      // Tier 3: 10+ absences (truancy threshold)
      if (student.absenceCount >= 10) {
        if (!student.hasTier3Intervention) {
          await createIntervention({
            studentId: student.id,
            type: 'attendance',
            tier: 3,
            strategies: [
              'Truancy petition',
              'Home visit',
              'District attendance officer referral',
              'Court involvement if necessary'
            ],
            goal: 'Immediate attendance improvement',
            reviewDate: addDays(new Date(), 14)
          })

          // Notify district attendance officer
          await notificationService.send({
            recipientRole: 'ATTENDANCE_OFFICER',
            title: 'Truancy Threshold Reached',
            body: `${student.name} has reached truancy threshold (${student.absenceCount} absences)`,
            priority: 'CRITICAL'
          })
        }
      }
    }
  }
}
```

### Compliance Reporting

**Chronic Absenteeism Report** (Federal requirement):
```typescript
interface ChronicAbsenteeismReport {
  schoolYear: string
  totalStudents: number
  chronicallyAbsent: number    // 10%+ absence rate
  rate: number                 // Percentage
  byGrade: { grade: string; count: number; rate: number }[]
  byDemographic: {
    specialEducation: { count: number; rate: number }
    englishLearner: { count: number; rate: number }
    economicallyDisadvantaged: { count: number; rate: number }
  }
}

async function generateChronicAbsenteeismReport(): Promise<ChronicAbsenteeismReport> {
  const schoolYear = getCurrentSchoolYear()
  const students = await getAllActiveStudents()

  const chronicallyAbsent = students.filter(s => {
    const absenceRate = (s.absenceCount / s.enrollmentDays) * 100
    return absenceRate >= 10
  })

  return {
    schoolYear,
    totalStudents: students.length,
    chronicallyAbsent: chronicallyAbsent.length,
    rate: (chronicallyAbsent.length / students.length) * 100,
    // ... calculate breakdowns
  }
}
```

**Average Daily Attendance (ADA)** (State funding):
```typescript
// Used for state funding calculations
async function calculateADA(startDate: Date, endDate: Date): Promise<number> {
  const days = eachDayOfInterval({ start: startDate, end: endDate })
  let totalPresent = 0
  let totalDays = 0

  for (const day of days) {
    if (isSchoolDay(day)) {
      const attendance = await getDailyAttendance(day)
      totalPresent += attendance.presentCount
      totalDays += attendance.enrolledCount
    }
  }

  return (totalPresent / totalDays) * 100
}
```

---

## Event System

### Domain Events

```typescript
export class AttendanceMarked extends DomainEvent {
  studentId: string
  classId?: string
  date: Date
  status: AttendanceStatus
  periodNumber?: number
  markedBy: string
}

export class AttendanceUpdated extends DomainEvent {
  recordId: string
  previousStatus: AttendanceStatus
  newStatus: AttendanceStatus
  updatedBy: string
  reason: string
}

export class DailyAttendanceAggregated extends DomainEvent {
  date: Date
  schoolId: string
  totalEnrolled: number
  presentCount: number
  absentCount: number
  tardyCount: number
  attendanceRate: number
  students: {
    id: string
    absenceCount: number
    hasTier1Intervention: boolean
    hasTier2Intervention: boolean
    hasTier3Intervention: boolean
  }[]
}

export class ChronicAbsenteeismDetected extends DomainEvent {
  studentId: string
  absenceCount: number
  enrollmentDays: number
  absenceRate: number
}
```

### Event Handlers

**1. DailyAttendanceAggregationHandler** (runs nightly):
```typescript
export class DailyAttendanceAggregationHandler {
  async handle(): Promise<void> {
    const today = new Date()
    const schools = await getAllSchools()

    for (const school of schools) {
      const students = await getEnrolledStudents(school.id)
      let presentCount = 0
      let absentCount = 0
      let tardyCount = 0

      const studentData = []

      for (const student of students) {
        const record = await getAttendanceRecord(student.id, today)
        
        if (record.status === 'P') presentCount++
        if (record.status.includes('ABSENT')) absentCount++
        if (record.status.includes('TARDY')) tardyCount++

        // Get student's total absences this year
        const absences = await countAbsencesThisYear(student.id)
        const interventions = await getActiveInterventions(student.id, 'attendance')

        studentData.push({
          id: student.id,
          absenceCount: absences,
          hasTier1Intervention: interventions.some(i => i.tier === 1),
          hasTier2Intervention: interventions.some(i => i.tier === 2),
          hasTier3Intervention: interventions.some(i => i.tier === 3)
        })
      }

      // Publish aggregated event
      await eventBus.publish(new DailyAttendanceAggregated({
        date: today,
        schoolId: school.id,
        totalEnrolled: students.length,
        presentCount,
        absentCount,
        tardyCount,
        attendanceRate: (presentCount / students.length) * 100,
        students: studentData
      }))
    }
  }
}
```

**2. ParentNotificationHandler**:
```typescript
export class ParentNotificationHandler implements IEventHandler<AttendanceMarked> {
  async handle(event: AttendanceMarked): Promise<void> {
    if (event.status.includes('ABSENT') || event.status.includes('TARDY')) {
      const parent = await getParentOf(event.studentId)

      if (parent && parent.preferences.attendanceNotifications) {
        // Send multi-channel notification
        await notificationQueue.add('attendance-alert', {
          parentId: parent.id,
          studentId: event.studentId,
          date: event.date,
          status: event.status,
          channels: parent.preferences.notificationChannels // ['EMAIL', 'SMS', 'PUSH']
        })
      }
    }
  }
}
```

**3. TruancyReportHandler**:
```typescript
export class TruancyReportHandler implements IEventHandler<DailyAttendanceAggregated> {
  async handle(event: DailyAttendanceAggregated): Promise<void> {
    const truantStudents = event.students.filter(s => s.absenceCount >= 10)

    if (truantStudents.length > 0) {
      // Generate truancy report
      const report = await generateTruancyReport({
        date: event.date,
        schoolId: event.schoolId,
        students: truantStudents
      })

      // Email to district attendance officer
      await notificationQueue.add('email', {
        to: getAttendanceOfficerEmail(event.schoolId),
        subject: `Daily Truancy Report - ${format(event.date, 'MM/dd/yyyy')}`,
        template: 'truancy-report',
        attachments: [report.pdfUrl]
      })
    }
  }
}
```

**4. AttendanceAnalyticsHandler**:
```typescript
export class AttendanceAnalyticsHandler implements IEventHandler<DailyAttendanceAggregated> {
  async handle(event: DailyAttendanceAggregated): Promise<void> {
    // Store daily metrics for analytics
    await db.attendanceAnalytics.create({
      data: {
        date: event.date,
        schoolId: event.schoolId,
        totalEnrolled: event.totalEnrolled,
        presentCount: event.presentCount,
        absentCount: event.absentCount,
        tardyCount: event.tardyCount,
        attendanceRate: event.attendanceRate,
        chronicAbsentCount: event.students.filter(s => {
          const rate = (s.absenceCount / getEnrollmentDays()) * 100
          return rate >= 10
        }).length
      }
    })
  }
}
```

---

## GraphQL API

### Queries

```graphql
type Query {
  # Get class attendance for date
  classAttendance(classId: ID!, date: Date!): ClassAttendance!

  # Get student's attendance history
  studentAttendance(studentId: ID!, startDate: Date!, endDate: Date!): StudentAttendance!

  # Get school-wide attendance summary
  schoolAttendanceSummary(schoolId: ID!, date: Date!): AttendanceSummary!

  # Get chronic absenteeism report
  chronicAbsenteeismReport(schoolId: ID!, schoolYear: String!): ChronicAbsenteeismReport!

  # Export attendance data
  exportAttendance(input: ExportAttendanceInput!): ExportUrl!
}
```

### Mutations

```graphql
type Mutation {
  # Mark attendance for student
  markAttendance(input: MarkAttendanceInput!): AttendanceRecord!

  # Bulk mark attendance for class
  bulkMarkAttendance(input: BulkMarkAttendanceInput!): [AttendanceRecord!]!

  # Update attendance record
  updateAttendance(recordId: ID!, status: AttendanceStatus!, reason: String): AttendanceRecord!

  # Import attendance from CSV
  importAttendance(file: Upload!): ImportResult!
}
```

### Subscriptions

```graphql
type Subscription {
  # Real-time attendance updates
  attendanceUpdated(schoolId: ID!): AttendanceUpdate!

  # Dashboard metrics
  attendanceDashboardUpdated(schoolId: ID!): AttendanceDashboard!
}
```

---

## UI Features

### Teacher View

**Quick Entry Interface**:
- Student roster with checkboxes
- "Mark All Present" button
- Quick status buttons (Present, Absent, Tardy)
- Keyboard shortcuts (P, A, T keys)
- Auto-save after each change

**Period-by-Period View**:
- Tab interface for each class period
- Copy attendance from previous period
- Bulk actions (Mark all present, Copy from homeroom)

### Admin View

**Real-Time Dashboard**:
- Live attendance rate (updates every 30 seconds)
- Classes with unmarked attendance highlighted
- Absence trends chart (last 30 days)
- Chronic absenteeism count

**Compliance Reports**:
- Average Daily Attendance (ADA)
- Chronic Absenteeism Report
- Truancy Report
- State reporting exports

### Parent View

**Attendance History**:
- Calendar view with color-coded days
- Green (Present), Red (Absent), Yellow (Tardy)
- Detailed view on click
- Download attendance certificate

**Notifications Settings**:
- Configure notification preferences
- Choose channels (Email, SMS, Push)
- Set quiet hours
- Opt-in to weekly summaries

---

## Analytics

### Student-Level Metrics

```typescript
interface StudentAttendanceMetrics {
  totalDays: number
  presentDays: number
  absentDays: number
  tardyDays: number
  attendanceRate: number
  chronicAbsentee: boolean
  consecutiveAbsences: number
  absencePattern: 'RANDOM' | 'MONDAYS' | 'FRIDAYS' | 'SPECIFIC_DAYS'
}
```

### School-Level Metrics

```typescript
interface SchoolAttendanceMetrics {
  averageDailyAttendance: number
  chronicAbsenteeismRate: number
  truancyCount: number
  improvementFromLastYear: number
  comparisonToStateAverage: number
}
```

---

## Related Documentation

- [Interventions](./interventions.md) - Academic/behavioral interventions
- [Notifications](./notifications.md) - Multi-channel notifications
- [Features Index](./README.md) - All features
