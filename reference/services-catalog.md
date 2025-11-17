# Services Catalog

**Last Updated**: November 17, 2025  
**Total Services**: 48  
**Architecture**: DDD + Clean Architecture

---

## Overview

This catalog documents all **services, use cases, and repositories** in the SkillCore platform, organized by bounded context using **Domain-Driven Design** principles.

---

## Service Architecture

### Service Layers

```
┌─────────────────────────────────────┐
│      Presentation Layer             │
│  (GraphQL/REST/WebSocket)           │
└───────────┬─────────────────────────┘
            │
┌───────────▼─────────────────────────┐
│      Application Layer              │
│  (Use Cases / Application Services) │
└───────────┬─────────────────────────┘
            │
┌───────────▼─────────────────────────┐
│        Domain Layer                 │
│  (Entities, Value Objects, Events)  │
└───────────┬─────────────────────────┘
            │
┌───────────▼─────────────────────────┐
│    Infrastructure Layer             │
│  (Repositories, External Services)  │
└─────────────────────────────────────┘
```

---

## 1. Student Domain Services

### Use Cases (8)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `CreateStudentUseCase` | Create new student record | StudentRepository, EventPublisher |
| `UpdateStudentUseCase` | Update student information | StudentRepository, EventPublisher |
| `DeactivateStudentUseCase` | Deactivate student | StudentRepository, EnrollmentService |
| `MergeStudentsUseCase` | Merge duplicate student records | StudentRepository, GradeService |
| `UpdateStudentProgramsUseCase` | Update IEP/504/ESL/Gifted status | StudentRepository, ProgramService |
| `AddStudentGuardianUseCase` | Link guardian to student | StudentRepository, ParentRepository |
| `RemoveStudentGuardianUseCase` | Unlink guardian from student | StudentRepository, AccessService |
| `LinkSiblingsUseCase` | Create sibling relationship | StudentRepository |

### Repositories (1)

```typescript
interface IStudentRepository {
  // Query methods
  findById(id: string): Promise<Student | null>
  findByStudentNumber(studentNumber: string, tenantId: string): Promise<Student | null>
  findByClass(classId: string, tenantId: string): Promise<Student[]>
  findByGradeLevel(gradeLevel: string, tenantId: string): Promise<Student[]>
  
  // Command methods
  save(student: Student): Promise<Student>
  update(student: Student): Promise<Student>
  delete(id: string): Promise<void>
  
  // Specialized queries
  findWithIEP(tenantId: string): Promise<Student[]>
  findWith504(tenantId: string): Promise<Student[]>
  findESLStudents(tenantId: string): Promise<Student[]>
  searchStudents(query: string, tenantId: string): Promise<Student[]>
}
```

---

## 2. Enrollment Domain Services

### Use Cases (6)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `EnrollStudentUseCase` | Enroll student in class | EnrollmentRepository, ClassRepository |
| `UnenrollStudentUseCase` | Remove student from class | EnrollmentRepository, GradeService |
| `TransferEnrollmentUseCase` | Move student to different class | EnrollmentRepository, GradeService |
| `ProcessEnrollmentRequestUseCase` | Handle enrollment request | EnrollmentRepository, NotificationService |
| `ManageWaitlistUseCase` | Manage class waitlist | WaitlistRepository, NotificationService |
| `BulkEnrollStudentsUseCase` | Batch enrollment operation | EnrollmentRepository, EventPublisher |

### Repositories (2)

```typescript
interface IEnrollmentRepository {
  findById(id: string): Promise<Enrollment | null>
  findByStudent(studentId: string, tenantId: string): Promise<Enrollment[]>
  findByClass(classId: string, tenantId: string): Promise<Enrollment[]>
  findActiveEnrollments(studentId: string, tenantId: string): Promise<Enrollment[]>
  save(enrollment: Enrollment): Promise<Enrollment>
  update(enrollment: Enrollment): Promise<Enrollment>
  delete(id: string): Promise<void>
}

interface IWaitlistRepository {
  findByStudent(studentId: string, tenantId: string): Promise<Waitlist[]>
  findByClass(classId: string, tenantId: string): Promise<Waitlist[]>
  getPosition(studentId: string, classId: string): Promise<number>
  save(waitlist: Waitlist): Promise<Waitlist>
  remove(id: string): Promise<void>
}
```

---

## 3. Gradebook Domain Services

### Use Cases (12)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `CreateAssignmentUseCase` | Create new assignment | AssignmentRepository, EventPublisher |
| `UpdateAssignmentUseCase` | Modify assignment | AssignmentRepository, GradeService |
| `DeleteAssignmentUseCase` | Remove assignment | AssignmentRepository, GradeRepository |
| `EnterGradeUseCase` | Enter/update grade | GradeRepository, CalculationService |
| `PostGradeUseCase` | Make grade visible to students | GradeRepository, NotificationService |
| `CalculateGradeAverageUseCase` | Calculate student average | GradeRepository, GradingPeriodService |
| `CalculateGPAUseCase` | Calculate student GPA | GradeRepository, TranscriptService |
| `ApplyLatePenaltyUseCase` | Apply late work penalty | GradeRepository, CalculationService |
| `ExemptStudentUseCase` | Exempt student from assignment | GradeRepository, CalculationService |
| `CreateGradeCategoryUseCase` | Create grade category | CategoryRepository, ClassRepository |
| `UpdateCategoryWeightUseCase` | Modify category weight | CategoryRepository, GradeService |
| `GenerateReportCardUseCase` | Generate report card | GradeRepository, ReportService |

### Repositories (3)

```typescript
interface IAssignmentRepository {
  findById(id: string): Promise<Assignment | null>
  findByClass(classId: string, tenantId: string): Promise<Assignment[]>
  findByCategory(categoryId: string, tenantId: string): Promise<Assignment[]>
  findPublished(classId: string, tenantId: string): Promise<Assignment[]>
  save(assignment: Assignment): Promise<Assignment>
  update(assignment: Assignment): Promise<Assignment>
  delete(id: string): Promise<void>
}

interface IGradeRepository {
  findById(id: string): Promise<Grade | null>
  findByStudent(studentId: string, tenantId: string): Promise<Grade[]>
  findByAssignment(assignmentId: string, tenantId: string): Promise<Grade[]>
  findByStudentAndAssignment(studentId: string, assignmentId: string): Promise<Grade | null>
  save(grade: Grade): Promise<Grade>
  update(grade: Grade): Promise<Grade>
  delete(id: string): Promise<void>
}

interface IGradeCategoryRepository {
  findById(id: string): Promise<GradeCategory | null>
  findByTenant(tenantId: string): Promise<GradeCategory[]>
  save(category: GradeCategory): Promise<GradeCategory>
  update(category: GradeCategory): Promise<GradeCategory>
  delete(id: string): Promise<void>
}
```

### Domain Services (2)

```typescript
class GradeCalculationService {
  calculateAverage(grades: Grade[], category: GradeCategory): number
  calculateWeightedAverage(grades: Grade[], categories: GradeCategory[]): number
  calculateGPA(grades: Grade[], gradeScale: GradeScale): number
  applyLatePenalty(grade: Grade, penalty: number): Grade
  calculateLetterGrade(percent: number, gradeScale: GradeScale): string
}

class GradingPeriodService {
  getCurrentPeriod(sessionId: string): Promise<GradingPeriod | null>
  getPeriodGrades(studentId: string, periodId: string): Promise<Grade[]>
  closePeriod(periodId: string): Promise<void>
  reopenPeriod(periodId: string): Promise<void>
}
```

---

## 4. Attendance Domain Services

### Use Cases (8)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `MarkAttendanceUseCase` | Record attendance | AttendanceRepository, EventPublisher |
| `UpdateAttendanceUseCase` | Modify attendance record | AttendanceRepository, EventPublisher |
| `ExcuseAbsenceUseCase` | Mark absence as excused | AttendanceRepository, NotificationService |
| `DetectChronicAbsenceUseCase` | Identify chronic absence patterns | AttendanceRepository, InterventionService |
| `CalculateAttendanceRateUseCase` | Calculate attendance rate | AttendanceRepository |
| `CreateInterventionUseCase` | Create attendance intervention | InterventionRepository, EventPublisher |
| `ScheduleAttendanceMeetingUseCase` | Schedule attendance meeting | MeetingRepository, NotificationService |
| `GenerateAttendanceReportUseCase` | Generate attendance report | AttendanceRepository, ReportService |

### Repositories (2)

```typescript
interface IAttendanceRepository {
  findById(id: string): Promise<AttendanceRecord | null>
  findByStudent(studentId: string, tenantId: string): Promise<AttendanceRecord[]>
  findByStudentAndDateRange(studentId: string, start: Date, end: Date): Promise<AttendanceRecord[]>
  findByClass(classId: string, date: Date, tenantId: string): Promise<AttendanceRecord[]>
  save(record: AttendanceRecord): Promise<AttendanceRecord>
  update(record: AttendanceRecord): Promise<AttendanceRecord>
  delete(id: string): Promise<void>
}

interface IAttendanceInterventionRepository {
  findById(id: string): Promise<AttendanceIntervention | null>
  findByStudent(studentId: string, tenantId: string): Promise<AttendanceIntervention[]>
  findActive(tenantId: string): Promise<AttendanceIntervention[]>
  save(intervention: AttendanceIntervention): Promise<AttendanceIntervention>
  update(intervention: AttendanceIntervention): Promise<AttendanceIntervention>
  close(id: string): Promise<void>
}
```

### Domain Services (1)

```typescript
class AttendanceCalculationService {
  calculateRate(records: AttendanceRecord[], startDate: Date, endDate: Date): number
  isChronicAbsent(records: AttendanceRecord[], threshold: number): boolean
  detectTrend(records: AttendanceRecord[]): 'improving' | 'declining' | 'stable'
  calculateDaysAbsent(records: AttendanceRecord[]): number
  calculateDaysTardy(records: AttendanceRecord[]): number
}
```

---

## 5. Assessment Domain Services

### Use Cases (10)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `CreateAssessmentUseCase` | Create new assessment | AssessmentRepository, EventPublisher |
| `ScheduleAssessmentUseCase` | Schedule assessment | AssessmentRepository, CalendarService |
| `RecordAssessmentScoreUseCase` | Record assessment score | ScoreRepository, EventPublisher |
| `AnalyzeAssessmentResultsUseCase` | Analyze assessment data | ScoreRepository, AnalyticsService |
| `CalculateGrowthUseCase` | Calculate student growth | ScoreRepository, GrowthService |
| `GenerateBenchmarkReportUseCase` | Generate benchmark report | ScoreRepository, ReportService |
| `IdentifyInterventionNeedsUseCase` | Identify students needing intervention | ScoreRepository, InterventionService |
| `CreateAssessmentWindowUseCase` | Create testing window | WindowRepository, NotificationService |
| `CloseAssessmentWindowUseCase` | Close testing window | WindowRepository, ReportService |
| `ExportAssessmentDataUseCase` | Export assessment data | ScoreRepository, ExportService |

### Repositories (2)

```typescript
interface IAssessmentRepository {
  findById(id: string): Promise<Assessment | null>
  findByType(type: AssessmentType, tenantId: string): Promise<Assessment[]>
  findBySubject(subject: Subject, tenantId: string): Promise<Assessment[]>
  save(assessment: Assessment): Promise<Assessment>
  update(assessment: Assessment): Promise<Assessment>
  delete(id: string): Promise<void>
}

interface IAssessmentScoreRepository {
  findById(id: string): Promise<AssessmentScore | null>
  findByStudent(studentId: string, tenantId: string): Promise<AssessmentScore[]>
  findByAssessment(assessmentId: string, tenantId: string): Promise<AssessmentScore[]>
  save(score: AssessmentScore): Promise<AssessmentScore>
  update(score: AssessmentScore): Promise<AssessmentScore>
  delete(id: string): Promise<void>
}
```

---

## 6. Communication Domain Services

### Use Cases (10)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `CreateAnnouncementUseCase` | Create announcement | AnnouncementRepository, NotificationService |
| `PublishAnnouncementUseCase` | Publish announcement | AnnouncementRepository, NotificationService |
| `SendMessageUseCase` | Send message | MessageRepository, NotificationService |
| `ScheduleConferenceUseCase` | Schedule parent-teacher conference | ConferenceRepository, CalendarService |
| `SendEmergencyAlertUseCase` | Send emergency alert | AlertService, NotificationService |
| `SendBulkNotificationUseCase` | Send bulk notification | NotificationService, QueueService |
| `TrackNotificationDeliveryUseCase` | Track notification delivery | NotificationRepository, AnalyticsService |
| `ManageNotificationPreferencesUseCase` | Update user notification preferences | PreferenceRepository |
| `GenerateParentReportUseCase` | Generate parent report | ReportService, StudentService |
| `ScheduleAutomatedReminderUseCase` | Schedule automated reminder | ReminderService, QueueService |

### Repositories (3)

```typescript
interface IAnnouncementRepository {
  findById(id: string): Promise<Announcement | null>
  findPublished(tenantId: string): Promise<Announcement[]>
  findByRecipient(userId: string, tenantId: string): Promise<Announcement[]>
  save(announcement: Announcement): Promise<Announcement>
  update(announcement: Announcement): Promise<Announcement>
  delete(id: string): Promise<void>
}

interface IMessageRepository {
  findById(id: string): Promise<Message | null>
  findByUser(userId: string, tenantId: string): Promise<Message[]>
  findConversation(user1Id: string, user2Id: string): Promise<Message[]>
  save(message: Message): Promise<Message>
  markAsRead(id: string): Promise<void>
  delete(id: string): Promise<void>
}

interface IConferenceRepository {
  findById(id: string): Promise<Conference | null>
  findByTeacher(teacherId: string, tenantId: string): Promise<Conference[]>
  findByParent(parentId: string, tenantId: string): Promise<Conference[]>
  findUpcoming(tenantId: string): Promise<Conference[]>
  save(conference: Conference): Promise<Conference>
  update(conference: Conference): Promise<Conference>
  cancel(id: string): Promise<void>
}
```

### Domain Services (3)

```typescript
class EmailService {
  sendEmail(to: string, subject: string, body: string): Promise<void>
  sendTemplatedEmail(to: string, template: string, data: any): Promise<void>
  sendBulkEmail(recipients: string[], subject: string, body: string): Promise<void>
}

class SMSService {
  sendSMS(to: string, message: string): Promise<void>
  sendBulkSMS(recipients: string[], message: string): Promise<void>
}

class PushNotificationService {
  sendPush(userId: string, title: string, body: string): Promise<void>
  sendBulkPush(userIds: string[], title: string, body: string): Promise<void>
}
```

---

## 7. Teacher Domain Services

### Use Cases (6)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `CreateTeacherUseCase` | Create new teacher | TeacherRepository, EventPublisher |
| `AssignClassToTeacherUseCase` | Assign class to teacher | TeacherRepository, ClassRepository |
| `UnassignClassFromTeacherUseCase` | Remove class from teacher | TeacherRepository, ClassRepository |
| `RecordProfessionalDevelopmentUseCase` | Record PD completion | PDRepository, CertificationService |
| `ScheduleObservationUseCase` | Schedule teacher observation | ObservationRepository, CalendarService |
| `CompleteEvaluationUseCase` | Complete teacher evaluation | EvaluationRepository, EventPublisher |

### Repositories (2)

```typescript
interface ITeacherRepository {
  findById(id: string): Promise<Teacher | null>
  findByEmail(email: string, tenantId: string): Promise<Teacher | null>
  findBySchool(schoolId: string, tenantId: string): Promise<Teacher[]>
  findActive(tenantId: string): Promise<Teacher[]>
  save(teacher: Teacher): Promise<Teacher>
  update(teacher: Teacher): Promise<Teacher>
  delete(id: string): Promise<void>
}

interface ITeacherPDRepository {
  findByTeacher(teacherId: string, tenantId: string): Promise<ProfessionalDevelopment[]>
  save(pd: ProfessionalDevelopment): Promise<ProfessionalDevelopment>
}
```

---

## 8. Analytics Domain Services

### Use Cases (6)

| Use Case | Description | Dependencies |
|----------|-------------|--------------|
| `GenerateReportUseCase` | Generate custom report | ReportRepository, DataService |
| `ScheduleRecurringReportUseCase` | Schedule recurring report | ReportRepository, QueueService |
| `ExportDataUseCase` | Export data to file | ExportService, StorageService |
| `CalculateMetricsUseCase` | Calculate analytics metrics | MetricsService, DataService |
| `DetectTrendUseCase` | Detect data trends | AnalyticsService, DataService |
| `CreateDashboardUseCase` | Create custom dashboard | DashboardRepository, WidgetService |

### Domain Services (2)

```typescript
class AnalyticsService {
  calculateStudentMetrics(studentId: string): Promise<StudentMetrics>
  calculateClassMetrics(classId: string): Promise<ClassMetrics>
  calculateSchoolMetrics(schoolId: string): Promise<SchoolMetrics>
  detectTrends(data: number[], threshold: number): 'up' | 'down' | 'stable'
  identifyOutliers(data: number[]): number[]
}

class ReportGenerationService {
  generatePDF(data: any, template: string): Promise<Buffer>
  generateExcel(data: any[], headers: string[]): Promise<Buffer>
  generateCSV(data: any[]): Promise<string>
}
```

---

## 9. Infrastructure Services

### External Services (6)

```typescript
class EmailService {
  provider: 'SendGrid'
  sendEmail(to, subject, body): Promise<void>
  sendTemplated(to, template, data): Promise<void>
}

class SMSService {
  provider: 'Twilio'
  sendSMS(to, message): Promise<void>
}

class PushNotificationService {
  provider: 'Firebase Cloud Messaging'
  sendPush(userId, title, body): Promise<void>
}

class StorageService {
  provider: 'AWS S3' | 'Azure Blob'
  uploadFile(file, path): Promise<string>
  downloadFile(path): Promise<Buffer>
  deleteFile(path): Promise<void>
}

class CacheService {
  provider: 'Redis'
  get(key): Promise<any>
  set(key, value, ttl): Promise<void>
  delete(key): Promise<void>
}

class QueueService {
  provider: 'BullMQ'
  addJob(queue, data): Promise<void>
  processJob(queue, handler): void
}
```

---

## Service Statistics

**Production Metrics** (November 2025):
- **Total Services**: 48
- **Use Cases**: 66
- **Repositories**: 20
- **Domain Services**: 8
- **Infrastructure Services**: 6
- **Average Response Time**: 85ms
- **Service Availability**: 99.95%

---

## Related Documentation

- [Architecture Overview](../architecture/overview.md)
- [Adding DDD Contexts](../guides/adding-ddd-context.md)
- [Database Schema](./database-schema.md)
- [Events Catalog](./events-catalog.md)
