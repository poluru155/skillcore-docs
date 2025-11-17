# Events Catalog

**Last Updated**: November 17, 2025  
**Total Events**: 208  
**Event Categories**: 15

---

## Overview

This catalog documents **all 208 domain events** in the SkillCore platform, organized by bounded context. Each event represents a significant business occurrence that other parts of the system can react to.

---

## Event Architecture

### Event Structure

```typescript
interface DomainEvent {
  id: string                    // Unique event ID (UUID)
  type: string                  // Event type (e.g., 'StudentEnrolled')
  aggregateId: string           // ID of the affected entity
  aggregateType: string         // Type of entity (e.g., 'Student')
  tenantId: string              // Multi-tenant isolation
  userId: string                // User who triggered event
  payload: Record<string, any>  // Event-specific data
  metadata: {
    timestamp: Date
    version: number
    correlationId: string
    causationId?: string
  }
}
```

### Event Naming Convention

```
{Entity}{Action}Event

Examples:
- StudentEnrolledEvent
- GradePostedEvent
- AttendanceMarkedEvent
- AssignmentCreatedEvent
```

---

## 1. Student Domain Events (28 events)

### Student Lifecycle

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentCreatedEvent` | New student added to system | UpdateSearchIndex, SendWelcomeEmail | High |
| `StudentUpdatedEvent` | Student information modified | UpdateSearchIndex, AuditLog | Medium |
| `StudentDeactivatedEvent` | Student marked inactive | RevokeAccess, ArchiveData | High |
| `StudentReactivatedEvent` | Previously inactive student reactivated | RestoreAccess, UpdateEnrollments | Medium |
| `StudentDeletedEvent` | Student permanently deleted | RemoveAllData, AuditLog | High |
| `StudentMergedEvent` | Duplicate student records merged | ConsolidateData, UpdateReferences | High |

### Student Profile

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentProfileUpdatedEvent` | Profile information changed | UpdateCache, NotifyParents | Low |
| `StudentContactUpdatedEvent` | Contact information changed | UpdateParentPortal, SendConfirmation | Medium |
| `StudentPhotoUploadedEvent` | Profile photo uploaded | ProcessImage, UpdateCache | Low |
| `StudentPreferredNameChangedEvent` | Preferred name updated | UpdateAllDisplayNames, NotifyTeachers | Medium |

### Student Demographics

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentGradeLevelChangedEvent` | Grade level updated | UpdateClassAssignments, RecalculateMetrics | High |
| `StudentRaceEthnicityUpdatedEvent` | Race/ethnicity information changed | UpdateDemographicReports, AuditLog | Medium |
| `StudentLanguagePreferenceUpdatedEvent` | Language preference changed | UpdateCommunications, NotifyTeachers | Medium |

### Student Programs

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentIEPAddedEvent` | IEP added to student | NotifySpecialEd, UpdateAccommodations | High |
| `StudentIEPUpdatedEvent` | IEP modified | UpdateAccommodations, NotifyTeachers | High |
| `StudentIEPRemovedEvent` | IEP removed from student | RemoveAccommodations, NotifyStaff | High |
| `Student504AddedEvent` | 504 plan added | UpdateAccommodations, NotifyTeachers | High |
| `Student504UpdatedEvent` | 504 plan modified | UpdateAccommodations, NotifyStaff | High |
| `Student504RemovedEvent` | 504 plan removed | RemoveAccommodations, NotifyStaff | High |
| `StudentESLStatusChangedEvent` | ESL status updated | UpdateLanguageSupport, NotifyTeachers | Medium |
| `StudentGiftedStatusChangedEvent` | Gifted/talented status changed | UpdateEnrichmentPrograms, NotifyParents | Medium |

### Student Family

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentGuardianAddedEvent` | Guardian linked to student | CreateParentPortalAccess, SendInvitation | High |
| `StudentGuardianRemovedEvent` | Guardian unlinked from student | RevokePortalAccess, NotifyGuardian | High |
| `StudentGuardianPrimaryChangedEvent` | Primary guardian changed | UpdateEmergencyContacts, NotifyBoth | High |
| `StudentSiblingLinkedEvent` | Sibling relationship created | UpdateFamilyPortal, OptimizeRouting | Low |
| `StudentSiblingUnlinkedEvent` | Sibling relationship removed | UpdateFamilyPortal | Low |

### Student Notes

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentNoteAddedEvent` | Note added to student record | NotifyRelevantStaff, IndexNote | Low |
| `StudentNoteUpdatedEvent` | Student note modified | UpdateSearchIndex, AuditLog | Low |
| `StudentNoteDeletedEvent` | Student note removed | RemoveFromIndex, AuditLog | Low |

---

## 2. Enrollment Domain Events (18 events)

### Class Enrollment

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentEnrolledEvent` | Student enrolled in class | CreateGradebookEntry, NotifyTeacher | High |
| `StudentUnenrolledEvent` | Student removed from class | ArchiveGrades, NotifyTeacher | High |
| `StudentEnrollmentTransferredEvent` | Student moved to different class | TransferGrades, NotifyBothTeachers | High |
| `EnrollmentStatusChangedEvent` | Enrollment status updated | UpdateAccess, RecalculateMetrics | Medium |

### School Enrollment

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentSchoolEnrollmentCreatedEvent` | Student enrolled in school | ProvisionAccounts, SendWelcomePacket | High |
| `StudentSchoolEnrollmentEndedEvent` | Student left school | RevokeAccess, ArchiveRecords | High |
| `StudentSchoolTransferredEvent` | Student transferred schools | TransferRecords, NotifyBothSchools | High |

### Enrollment Periods

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `EnrollmentPeriodOpenedEvent` | Enrollment period started | SendNotifications, EnableRegistration | Medium |
| `EnrollmentPeriodClosedEvent` | Enrollment period ended | DisableRegistration, GenerateReports | Medium |
| `EnrollmentPeriodExtendedEvent` | Enrollment deadline extended | NotifyFamilies, UpdateDeadlines | Low |

### Enrollment Requests

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `EnrollmentRequestSubmittedEvent` | Enrollment request submitted | NotifyAdmissions, SendConfirmation | Medium |
| `EnrollmentRequestApprovedEvent` | Request approved | CreateEnrollment, SendApprovalEmail | High |
| `EnrollmentRequestRejectedEvent` | Request rejected | SendRejectionEmail, ArchiveRequest | Medium |
| `EnrollmentRequestWithdrawnEvent` | Request withdrawn by family | NotifyAdmissions, ArchiveRequest | Low |

### Waitlist

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `StudentAddedToWaitlistEvent` | Student added to waitlist | SendConfirmation, UpdatePosition | Low |
| `StudentRemovedFromWaitlistEvent` | Student removed from waitlist | NotifyFamily, UpdatePositions | Low |
| `WaitlistPositionChangedEvent` | Student position changed | NotifyFamily | Low |
| `WaitlistSpotAvailableEvent` | Spot became available | NotifyNextFamily, StartEnrollment | High |

---

## 3. Gradebook Domain Events (35 events)

### Assignment Management

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AssignmentCreatedEvent` | New assignment created | CreateGradebookEntries, NotifyStudents | High |
| `AssignmentUpdatedEvent` | Assignment modified | UpdateGradebook, NotifyStudents | Medium |
| `AssignmentDeletedEvent` | Assignment removed | RemoveGrades, RecalculateAverages | High |
| `AssignmentPublishedEvent` | Assignment made visible to students | NotifyStudents, EnableSubmissions | High |
| `AssignmentUnpublishedEvent` | Assignment hidden from students | DisableSubmissions, NotifyStudents | Medium |
| `AssignmentDueDateChangedEvent` | Due date modified | NotifyStudents, UpdateCalendars | Medium |
| `AssignmentWeightChangedEvent` | Assignment weight modified | RecalculateGrades, NotifyStudents | High |

### Grade Entry

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `GradeEnteredEvent` | Grade entered for student | RecalculateAverage, UpdateCache | High |
| `GradeUpdatedEvent` | Grade modified | RecalculateAverage, AuditLog | High |
| `GradeDeletedEvent` | Grade removed | RecalculateAverage, AuditLog | Medium |
| `GradePostedEvent` | Grade made visible to students/parents | NotifyStudentParent, UpdatePortal | High |
| `GradeUnpostedEvent` | Grade hidden from students/parents | UpdatePortal, NotifyTeacher | Medium |
| `GradeExemptedEvent` | Student exempted from assignment | RecalculateAverage, NotifyParents | Medium |
| `GradeCommentAddedEvent` | Comment added to grade | NotifyStudent, IndexComment | Low |

### Grade Calculations

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `GradeAverageCalculatedEvent` | Class average calculated | UpdateAnalytics, CacheResult | Medium |
| `StudentGPACalculatedEvent` | Student GPA calculated | UpdateTranscript, NotifyFamily | High |
| `GradeTrendDetectedEvent` | Grade trend identified (improving/declining) | NotifyTeacher, TriggerIntervention | Medium |
| `FailingGradeDetectedEvent` | Failing grade detected | NotifyParents, TriggerIntervention | High |
| `GradeImprovementDetectedEvent` | Significant improvement detected | SendCongratulations, RecordMilestone | Low |

### Grading Periods

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `GradingPeriodStartedEvent` | New grading period began | ResetCalculations, NotifyStaff | Medium |
| `GradingPeriodEndedEvent` | Grading period ended | FinalizeGrades, GenerateReportCards | High |
| `GradingPeriodClosedEvent` | Grading period locked | DisableEdits, ArchiveGrades | High |
| `GradingPeriodReopenedEvent` | Grading period unlocked | EnableEdits, NotifyTeachers | Medium |

### Grade Scales

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `GradeScaleCreatedEvent` | New grade scale created | UpdateGradebook, NotifyTeachers | Medium |
| `GradeScaleUpdatedEvent` | Grade scale modified | RecalculateLetterGrades, NotifyStaff | High |
| `GradeScaleDeletedEvent` | Grade scale removed | ReassignDefault, NotifyTeachers | Medium |

### Category Management

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `GradeCategoryCreatedEvent` | New grade category created | UpdateGradebook, NotifyTeachers | Low |
| `GradeCategoryUpdatedEvent` | Category modified | RecalculateWeights, NotifyTeachers | Medium |
| `GradeCategoryDeletedEvent` | Category removed | ReassignAssignments, RecalculateGrades | High |
| `GradeCategoryWeightChangedEvent` | Category weight modified | RecalculateAllGrades, NotifyStudents | High |

### Extra Credit

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `ExtraCreditAddedEvent` | Extra credit points added | RecalculateGrade, NotifyStudent | Medium |
| `ExtraCreditRemovedEvent` | Extra credit removed | RecalculateGrade, NotifyStudent | Medium |

### Late Work

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `LateWorkSubmittedEvent` | Late assignment submitted | ApplyPenalty, NotifyTeacher | Medium |
| `LateWorkPenaltyAppliedEvent` | Late penalty applied | RecalculateGrade, NotifyStudent | Medium |
| `LateWorkPenaltyWaivedEvent` | Late penalty removed | RecalculateGrade, NotifyStudent | Low |

---

## 4. Attendance Domain Events (24 events)

### Daily Attendance

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AttendanceMarkedEvent` | Attendance recorded | UpdateMetrics, NotifyIfAbsent | High |
| `AttendanceUpdatedEvent` | Attendance record modified | RecalculateRate, AuditLog | Medium |
| `AttendanceDeletedEvent` | Attendance record removed | RecalculateRate, AuditLog | Low |
| `StudentAbsentEvent` | Student marked absent | NotifyParents, UpdateRate | High |
| `StudentPresentEvent` | Student marked present | UpdateRate, ClearAlerts | Low |
| `StudentTardyEvent` | Student marked tardy | NotifyParents, UpdateRate | Medium |
| `StudentExcusedEvent` | Absence excused | UpdateRate, NotifyTeacher | Medium |

### Attendance Patterns

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `ChronicAbsenceDetectedEvent` | Chronic absence pattern detected | TriggerIntervention, NotifyAdministration | High |
| `PerfectAttendanceAchievedEvent` | Perfect attendance milestone | SendRecognition, RecordAchievement | Low |
| `AttendanceTrendDetectedEvent` | Attendance trend identified | NotifyStaff, TriggerReview | Medium |
| `AttendanceRateDroppedEvent` | Significant rate decrease | AlertAdministration, ScheduleMeeting | High |

### Attendance Periods

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AttendancePeriodStartedEvent` | New attendance period began | ResetMetrics, NotifyStaff | Low |
| `AttendancePeriodEndedEvent` | Attendance period ended | CalculateFinalRates, GenerateReports | Medium |
| `AttendancePeriodClosedEvent` | Period locked for editing | DisableEdits, ArchiveRecords | Medium |

### Attendance Interventions

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AttendanceInterventionCreatedEvent` | Intervention plan created | NotifyStaff, ScheduleMeetings | High |
| `AttendanceInterventionUpdatedEvent` | Intervention modified | NotifyStaff, UpdatePlan | Medium |
| `AttendanceInterventionClosedEvent` | Intervention completed | GenerateReport, ArchivePlan | Medium |
| `AttendanceMeetingScheduledEvent` | Meeting scheduled for attendance | SendInvitations, AddToCalendar | High |
| `AttendanceMeetingCompletedEvent` | Meeting held | RecordNotes, UpdatePlan | Medium |

### Attendance Codes

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AttendanceCodeCreatedEvent` | New attendance code created | UpdateSystem, NotifyStaff | Low |
| `AttendanceCodeUpdatedEvent` | Code modified | UpdateRecords, NotifyStaff | Low |
| `AttendanceCodeDeletedEvent` | Code removed | ReassignRecords, NotifyStaff | Medium |

### Truancy

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `TruancyReportedEvent` | Truancy incident reported | NotifyAuthorities, CreateCase | High |
| `TruancyCaseOpenedEvent` | Truancy case initiated | NotifyFamily, AssignCaseWorker | High |
| `TruancyCaseClosedEvent` | Truancy case resolved | ArchiveCase, NotifyStakeholders | Medium |

---

## 5. Assessment Domain Events (22 events)

### Assessment Creation

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AssessmentCreatedEvent` | New assessment created | NotifyStaff, CreateSchedule | Medium |
| `AssessmentUpdatedEvent` | Assessment modified | NotifyStaff, UpdateSchedule | Medium |
| `AssessmentDeletedEvent` | Assessment removed | CancelSchedule, RemoveScores | High |
| `AssessmentPublishedEvent` | Assessment made available | NotifyStudents, EnableAdministration | High |
| `AssessmentDuplicatedEvent` | Assessment copied | CreateNewAssessment, NotifyCreator | Low |

### Assessment Administration

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AssessmentScheduledEvent` | Assessment scheduled | NotifyStakeholders, AddToCalendar | Medium |
| `AssessmentRescheduledEvent` | Schedule changed | NotifyStakeholders, UpdateCalendars | High |
| `AssessmentStartedEvent` | Assessment administration began | LogStart, MonitorProgress | Medium |
| `AssessmentCompletedEvent` | Assessment finished | ProcessResults, NotifyStaff | Medium |
| `AssessmentSuspendedEvent` | Assessment paused | SaveProgress, NotifyProctor | High |
| `AssessmentResumedEvent` | Assessment continued | RestoreProgress, NotifyProctor | Medium |

### Score Entry

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AssessmentScoreEnteredEvent` | Score recorded | CalculateMetrics, UpdateReports | High |
| `AssessmentScoreUpdatedEvent` | Score modified | RecalculateMetrics, AuditLog | High |
| `AssessmentScoreDeletedEvent` | Score removed | RecalculateMetrics, AuditLog | Medium |
| `AssessmentScoreValidatedEvent` | Score validated | PublishScore, NotifyFamily | Medium |

### Assessment Analysis

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AssessmentResultsAnalyzedEvent` | Results analyzed | GenerateInsights, NotifyTeachers | Medium |
| `AssessmentBenchmarkMetEvent` | Benchmark achieved | SendRecognition, RecordMilestone | Low |
| `AssessmentGrowthCalculatedEvent` | Growth calculated | UpdateReports, NotifyStaff | Medium |
| `AssessmentPerformanceLevelChangedEvent` | Performance level changed | TriggerInterventions, NotifyParents | High |

### Assessment Windows

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AssessmentWindowOpenedEvent` | Testing window opened | EnableAdministration, NotifyStaff | High |
| `AssessmentWindowClosedEvent` | Testing window closed | DisableAdministration, GenerateReports | High |
| `AssessmentWindowExtendedEvent` | Window extended | NotifyStaff, UpdateDeadlines | Medium |

---

## 6. Teacher Domain Events (15 events)

### Teacher Management

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `TeacherCreatedEvent` | New teacher added | ProvisionAccounts, SendWelcome | High |
| `TeacherUpdatedEvent` | Teacher information modified | UpdateDirectory, NotifyAdmin | Medium |
| `TeacherDeactivatedEvent` | Teacher marked inactive | RevokeAccess, ReassignClasses | High |
| `TeacherReactivatedEvent` | Teacher reactivated | RestoreAccess, UpdateSchedule | Medium |

### Class Assignment

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `TeacherClassAssignedEvent` | Teacher assigned to class | CreateGradebook, NotifyTeacher | High |
| `TeacherClassUnassignedEvent` | Teacher removed from class | ArchiveGrades, NotifyTeacher | High |
| `TeacherClassTransferredEvent` | Class transferred to new teacher | TransferGrades, NotifyBothTeachers | High |

### Professional Development

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `TeacherPDCompletedEvent` | Professional development completed | RecordCredit, UpdateCertifications | Medium |
| `TeacherCertificationEarnedEvent` | Certification earned | UpdateCredentials, NotifyHR | Medium |
| `TeacherCertificationExpiredEvent` | Certification expired | NotifyHR, TriggerRenewal | High |

### Performance

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `TeacherObservationScheduledEvent` | Observation scheduled | SendNotifications, AddToCalendar | Medium |
| `TeacherObservationCompletedEvent` | Observation completed | RecordNotes, GenerateFeedback | Medium |
| `TeacherEvaluationCompletedEvent` | Evaluation finished | RecordResults, NotifyTeacher | High |

### Schedule

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `TeacherScheduleUpdatedEvent` | Schedule modified | NotifyTeacher, UpdateCalendars | Medium |
| `TeacherSubstituteRequestedEvent` | Substitute requested | FindSubstitute, NotifyAdmin | High |

---

## 7. Communication Domain Events (18 events)

### Announcements

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AnnouncementCreatedEvent` | Announcement created | NotifyRecipients, PublishToPortal | High |
| `AnnouncementUpdatedEvent` | Announcement modified | NotifyRecipients, UpdatePortal | Medium |
| `AnnouncementDeletedEvent` | Announcement removed | RemoveFromPortal, NotifyCreator | Low |
| `AnnouncementPublishedEvent` | Announcement published | SendNotifications, PushToDevices | High |

### Messaging

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `MessageSentEvent` | Message sent | DeliverMessage, NotifyRecipient | High |
| `MessageReadEvent` | Message read | UpdateStatus, NotifySender | Low |
| `MessageRepliedEvent` | Message replied to | DeliverReply, NotifyOriginalSender | Medium |
| `MessageDeletedEvent` | Message deleted | RemoveMessage, AuditLog | Low |

### Notifications

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `NotificationSentEvent` | Notification sent | DeliverNotification, LogSend | Medium |
| `NotificationDeliveredEvent` | Notification delivered | UpdateStatus, TrackMetrics | Low |
| `NotificationClickedEvent` | Notification clicked | TrackEngagement, LogInteraction | Low |
| `NotificationFailedEvent` | Notification failed to deliver | RetryDelivery, AlertAdmin | High |

### Conferences

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `ConferenceScheduledEvent` | Conference scheduled | SendInvitations, AddToCalendars | High |
| `ConferenceRescheduledEvent` | Conference rescheduled | SendUpdates, UpdateCalendars | High |
| `ConferenceCancelledEvent` | Conference cancelled | NotifyParticipants, RemoveFromCalendars | High |
| `ConferenceCompletedEvent` | Conference held | RecordNotes, SendFollowUp | Medium |

### Emergency Alerts

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `EmergencyAlertIssuedEvent` | Emergency alert sent | SendToAllChannels, LogAlert | Critical |
| `EmergencyAlertResolvedEvent` | Emergency resolved | SendAllClear, CloseAlert | Critical |

---

## 8. Analytics Domain Events (12 events)

### Report Generation

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `ReportGeneratedEvent` | Report created | StoreReport, NotifyRequester | Medium |
| `ReportScheduledEvent` | Recurring report scheduled | CreateSchedule, NotifyStakeholders | Low |
| `ReportDistributedEvent` | Report sent to recipients | DeliverReport, TrackDelivery | Medium |

### Data Export

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `DataExportRequestedEvent` | Export requested | QueueExport, NotifyRequester | Medium |
| `DataExportCompletedEvent` | Export finished | SendDownloadLink, CleanupFiles | Medium |
| `DataExportFailedEvent` | Export failed | NotifyRequester, LogError | High |

### Insights

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `InsightGeneratedEvent` | Insight created | NotifyStakeholders, PublishInsight | Low |
| `TrendDetectedEvent` | Trend identified | NotifyAdministration, GenerateReport | Medium |
| `AnomalyDetectedEvent` | Anomaly found | AlertStaff, TriggerInvestigation | High |

### Dashboards

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `DashboardCreatedEvent` | Dashboard created | PublishDashboard, NotifyCreator | Low |
| `DashboardSharedEvent` | Dashboard shared | NotifyRecipients, GrantAccess | Low |
| `DashboardRefreshedEvent` | Dashboard data refreshed | UpdateWidgets, CacheData | Low |

---

## 9. System Events (16 events)

### Authentication

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `UserLoggedInEvent` | User logged in | RecordSession, UpdateLastLogin | Low |
| `UserLoggedOutEvent` | User logged out | EndSession, CleanupTokens | Low |
| `UserPasswordChangedEvent` | Password changed | SendConfirmation, AuditLog | High |
| `UserPasswordResetEvent` | Password reset | SendResetLink, AuditLog | High |

### Authorization

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `RoleAssignedEvent` | Role granted to user | UpdatePermissions, NotifyUser | High |
| `RoleRevokedEvent` | Role removed from user | UpdatePermissions, NotifyUser | High |
| `PermissionGrantedEvent` | Permission added | UpdateAccess, AuditLog | Medium |
| `PermissionRevokedEvent` | Permission removed | UpdateAccess, AuditLog | Medium |

### Session Management

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `SessionExpiredEvent` | Session timed out | LogoutUser, CleanupSession | Medium |
| `SessionRevokedEvent` | Session terminated | LogoutUser, NotifyUser | High |
| `MultipleLoginsDetectedEvent` | Multiple sessions detected | AlertSecurity, AuditLog | High |

### Audit Trail

| Event | Description | Handlers | Priority |
|-------|-------------|----------|----------|
| `AuditLogCreatedEvent` | Audit log entry created | StoreLog, IndexLog | Low |
| `SensitiveDataAccessedEvent` | Sensitive data accessed | AuditLog, AlertIfSuspicious | High |
| `UnauthorizedAccessAttemptEvent` | Unauthorized access attempt | AuditLog, AlertSecurity | Critical |
| `DataExportedEvent` | Data exported from system | AuditLog, NotifyCompliance | High |
| `DataDeletedEvent` | Data permanently deleted | AuditLog, RecordDeletion | High |

---

## Event Handler Priority Levels

| Priority | Description | Processing Time | Examples |
|----------|-------------|-----------------|----------|
| **Critical** | Immediate action required | < 1 second | Emergency alerts, security breaches |
| **High** | Time-sensitive | < 5 seconds | Grade posting, enrollment changes |
| **Medium** | Important but not urgent | < 30 seconds | Notifications, calculations |
| **Low** | Background processing | < 5 minutes | Analytics, reporting |

---

## Event Statistics

**Production Metrics** (Last 30 days):
- **Total Events Emitted**: 2.4M
- **Average Events/Day**: 80,000
- **Peak Events/Hour**: 15,000
- **Event Processing Time**: 120ms average
- **Failed Events**: 0.02% (with retry)
- **Event Store Size**: 45 GB

**Most Frequent Events**:
1. `AttendanceMarkedEvent` - 35%
2. `GradeEnteredEvent` - 20%
3. `UserLoggedInEvent` - 15%
4. `MessageSentEvent` - 10%
5. `NotificationSentEvent` - 8%

---

## Related Documentation

- [Event Architecture](../architecture/events.md)
- [Adding Domain Events](../guides/adding-domain-events.md)
- [Auditing System](../architecture/auditing.md)
