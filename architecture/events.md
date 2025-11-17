# Event-Driven Architecture

**Last Updated**: November 16, 2025

---

## Overview

SkillCore WebApp implements a comprehensive event-driven architecture using Redis EventBus for distributed event processing, enabling scalable, decoupled, and FERPA-compliant educational workflows.

---

## Event Infrastructure

### Redis EventBus
- **Implementation**: Redis Streams-based event bus
- **Total Events**: 208 domain events emitting
- **Total Handlers**: 79 registered event handlers
- **Persistence**: Dual-write to EventStore (PostgreSQL) + Redis
- **Scalability**: Consumer groups for horizontal scaling
- **Reliability**: Dead Letter Queue (DLQ) for failed events

### Architecture Pattern
```
Event Publisher → Redis Stream → Consumer Groups → Event Handlers → Side Effects
                       ↓
                  EventStore (Postgres)
                  (FERPA Audit Trail)
```

### Key Features
- ✅ **At-least-once delivery** with consumer acknowledgment
- ✅ **Event ordering** within streams
- ✅ **Automatic retry** with exponential backoff (5 attempts)
- ✅ **Dead Letter Queue** for failed events
- ✅ **Consumer groups** for load distribution
- ✅ **Stalled message recovery** (10-minute timeout)
- ✅ **FERPA compliance** via EventStore persistence

---

## Domain Events by Context

### 1. Special Programs (25 Events, 6 Handlers)

**IEP Events (10)**:
- `IEPCreated` - IEP plan created
- `IEPUpdated` - IEP plan modified
- `IEPArchived` - IEP plan archived
- `IEPReviewScheduled` - Annual review scheduled
- `IEPGoalAdded` - New goal added to IEP
- `IEPGoalUpdated` - Goal progress updated
- `IEPGoalArchived` - Goal completed/archived
- `IEPAccommodationModified` - Accommodation changed
- `IEPServiceHoursUpdated` - Service hours modified
- `IEPTeamMeetingScheduled` - Team meeting scheduled

**Section 504 Events (8)**:
- `Section504PlanCreated` - 504 plan created
- `Section504PlanUpdated` - 504 plan modified
- `Section504Archived` - 504 plan archived
- `Section504ReviewScheduled` - Review scheduled
- `Section504AccommodationAdded` - Accommodation added
- `Section504AccommodationRemoved` - Accommodation removed
- `Section504AccommodationUpdated` - Accommodation modified
- `Section504TeamMeetingScheduled` - Team meeting scheduled

**ESL Events (4)**:
- `ESLEnrollmentCreated` - Student enrolled in ESL
- `ESLServicesUpdated` - ESL services modified
- `ESLProgramExited` - Student exited ESL program
- `ESLProficiencyLevelChanged` - Language proficiency updated

**Gifted Events (3)**:
- `GiftedNominationCreated` - Student nominated for gifted
- `GiftedIdentificationComplete` - Identification process finished
- `GiftedEnrollmentCreated` - Student enrolled in gifted program
- `GiftedServicesUpdated` - Gifted services modified
- `GiftedProgramExited` - Student exited gifted program

**Special Programs Handlers**:
1. **CoordinatorNotificationHandler** (7 events) - Notifies program coordinators
2. **TeacherAccommodationHandler** (7 events) - Alerts teachers of accommodations
3. **IEPInterventionHandler** (2 events) - Auto-creates Tier 2 interventions
4. **StudentProfileUpdateHandler** (8 events) - Updates student metadata
5. **ComplianceReportHandler** (2 events) - IDEA/504 compliance logging
6. **ArchiveProgramDataHandler** (4 events) - Audit trail for program exits

---

### 2. Notification System (18 Events, 6 Handlers)

**Email Events (6)**:
- `EmailSent` - Email dispatched to SendGrid
- `EmailDelivered` - SendGrid confirmed delivery
- `EmailBounced` - Email bounced (hard/soft)
- `EmailOpened` - Recipient opened email
- `EmailClicked` - Link clicked in email
- `EmailDeliveryFailed` - Email failed to send

**SMS Events (5)**:
- `SMSSent` - SMS dispatched to Twilio
- `SMSDelivered` - Twilio confirmed delivery
- `SMSFailed` - SMS failed to send
- `SMSDeliveryFailed` - Delivery failure
- `SMSReplied` - Recipient replied to SMS

**Push Notification Events (5)**:
- `PushNotificationSent` - Push sent to Firebase FCM
- `PushNotificationDelivered` - FCM confirmed delivery
- `PushNotificationClicked` - User clicked notification
- `PushNotificationFailed` - Push failed to send
- `PushNotificationDeliveryFailed` - Delivery failure

**Multi-Channel Events (2)**:
- `NotificationDelivered` - Notification delivered on any channel
- `NotificationPartiallyDelivered` - Some channels failed

**Notification Handlers**:
1. **NotificationDeliveryAnalyticsHandler** - Tracks delivery across all channels
2. **EmailDeliveryHandler** - Processes SendGrid webhooks
3. **SMSDeliveryHandler** - Processes Twilio webhooks
4. **PushNotificationHandler** - Processes Firebase callbacks
5. **NotificationEngagementHandler** - Tracks opens, clicks, replies
6. **DeliveryFailureHandler** - Retry logic and DLQ management

---

### 3. Communication (15+ Events, 8 Handlers)

**Message Events**:
- `MessageSent` - New message created
- `MessageDelivered` - Message delivered to recipient
- `MessageRead` - Message marked as read
- `MessageEdited` - Message content edited
- `MessageDeleted` - Message deleted

**Announcement Events**:
- `AnnouncementPublished` - Announcement published
- `AnnouncementUnpublished` - Announcement removed
- `AnnouncementEdited` - Announcement content edited

**Conference Events**:
- `ConferenceScheduled` - Conference scheduled
- `ConferenceRescheduled` - Conference time changed
- `ConferenceCancelled` - Conference cancelled

**Other Events**:
- `NotificationPreferenceUpdated` - User changed notification settings
- `ConversationCreated` - New conversation started
- `ParticipantAdded` - Participant added to conversation

**Communication Handlers**:
1. **MessageDeliveryHandler** - Coordinates message delivery
2. **AnnouncementPublishingHandler** - Publishes announcements
3. **ConferenceSchedulingHandler** - Manages conference logistics
4. **ParentTeacherCommunicationHandler** - Routes parent-teacher messages
5. **ReadReceiptHandler** - Processes read receipts
6. **NotificationPreferenceHandler** - Manages user preferences
7. **ConversationHandler** - Manages conversation state
8. **ParticipantHandler** - Manages conversation participants

---

### 4. Gradebook (12 Handlers)

**Grade Events**:
- `GradeCreated` - New grade entry
- `GradeUpdated` - Grade modified
- `GradePosted` - Grade posted to student
- `BulkGradesUpdated` - Multiple grades updated

**Assignment Events**:
- `AssignmentCreated` - New assignment
- `AssignmentUpdated` - Assignment modified
- `AssignmentPublished` - Assignment released
- `AssignmentDeleted` - Assignment removed

**Gradebook Handlers**:
1. **GradeCalculationHandler** - Calculates weighted averages
2. **GPAUpdateHandler** - Updates student GPA
3. **LetterGradeConversionHandler** - Converts to letter grades
4. **GradePostingNotificationHandler** - Notifies students/parents
5. **AssignmentWeightRecalculationHandler** - Recalculates category weights
6. **CategoryAverageHandler** - Updates category averages
7. **ClassPerformanceHandler** - Calculates class metrics
8. **GradeTrendHandler** - Tracks grade trends
9. **MissingAssignmentHandler** - Alerts for missing work
10. **ExtraCreditHandler** - Processes extra credit
11. **GradeOverrideHandler** - Handles manual overrides
12. **GradebookAuditHandler** - Logs all grade changes

---

### 5. Attendance (4 Handlers)

**Attendance Events**:
- `AttendanceMarked` - Attendance recorded
- `AttendanceUpdated` - Attendance modified
- `AbsenceRecorded` - Student absent
- `TardyRecorded` - Student tardy

**Attendance Handlers**:
1. **DailyAttendanceAggregationHandler** - Daily rollups
2. **AbsenceInterventionHandler** - Triggers interventions
3. **AttendanceRateCalculationHandler** - Calculates rates
4. **ParentNotificationHandler** - Notifies parents of absences

---

### 6. Chat (10 Handlers - FERPA Compliant)

**Chat Events**:
- `ChatMessageSent` - New chat message
- `ChatMessageDelivered` - Message delivered
- `ChatMessageRead` - Message read
- `ChatConversationCreated` - New conversation
- `ChatParticipantAdded` - Participant added
- `ChatParticipantRemoved` - Participant removed

**Chat Handlers**:
1. **MessageDeliveryHandler** - Real-time delivery
2. **ReadReceiptHandler** - Read status tracking
3. **ConversationCreationHandler** - Creates conversations
4. **ParticipantManagementHandler** - Manages participants
5. **MessageSearchIndexHandler** - Indexes messages
6. **FERPAComplianceHandler** - Ensures student privacy
7. **ChatAnalyticsHandler** - Tracks engagement
8. **AttachmentProcessingHandler** - Processes file attachments
9. **TypingIndicatorHandler** - Real-time typing status
10. **ChatNotificationHandler** - Push/email notifications

---

### 7. Demographics (6 Handlers)

**User Events**:
- `UserCreated` - New user account
- `UserUpdated` - User profile changed
- `UserDeactivated` - User account disabled

**Enrollment Events**:
- `EnrollmentCreated` - Student enrolled
- `EnrollmentUpdated` - Enrollment modified
- `EnrollmentEnded` - Student unenrolled

**Demographics Handlers**:
1. **UserProfileHandler** - Syncs user profiles
2. **EnrollmentHandler** - Manages class rosters
3. **FamilyRelationshipHandler** - Updates family links
4. **EmergencyContactHandler** - Manages contacts
5. **DemographicDataHandler** - Tracks demographics
6. **UserDeactivationHandler** - Handles account closure

---

### 8. Rostering (12 Handlers)

**Class Events**:
- `ClassCreated` - New class created
- `ClassUpdated` - Class modified
- `ClassArchived` - Class archived

**Section Events**:
- `SectionCreated` - New section
- `SectionUpdated` - Section modified

**Rostering Handlers**:
1. **ClassCreationHandler** - Initializes classes
2. **ClassUpdateHandler** - Updates class data
3. **EnrollmentSyncHandler** - Syncs student rosters
4. **TeacherAssignmentHandler** - Assigns teachers
5. **SectionManagementHandler** - Manages sections
6. **AcademicSessionHandler** - Manages school years
7. **CourseHandler** - Manages course catalog
8. **ScheduleHandler** - Builds schedules
9. **RoomAssignmentHandler** - Assigns classrooms
10. **BellScheduleHandler** - Manages periods
11. **CalendarHandler** - Academic calendar
12. **RosteringAuditHandler** - Audit logging

---

## Event Worker Architecture

### Worker Deployment

**Standalone Processes**:
```bash
# Start event worker
pnpm worker

# Development mode with hot reload
pnpm worker:dev

# Production deployment
docker-compose up event-worker --scale event-worker=3
```

**Configuration**:
- Separate from API instances (API publishes, workers consume)
- Multiple workers for horizontal scaling
- Consumer groups ensure no duplicate processing
- Graceful shutdown on SIGTERM/SIGINT

### Worker Components

**register-handlers.ts**:
- Centralized handler registration
- 79 handlers registered at startup
- Dependency injection for services
- Error handling setup

**event-worker.ts**:
- Main worker process (120 lines)
- EventBus initialization
- Handler registration
- Health check endpoint
- Graceful shutdown

---

## CQRS Command Handlers

### Command Pattern
```typescript
// Command
interface UpdateGradeCommand {
  gradeId: string
  score: number
  teacherId: string
  tenantId: string
}

// Command Handler
class UpdateGradeCommandHandler {
  async execute(command: UpdateGradeCommand): Promise<void> {
    // 1. Validate command
    // 2. Load aggregate
    // 3. Execute business logic
    // 4. Persist changes
    // 5. Publish domain events
  }
}
```

### Command Handlers (47+ Total)

**Assessment Commands**:
- CreateTestCommand, UpdateTestCommand, PublishTestCommand
- CreateQuestionCommand, UpdateQuestionCommand
- SubmitTestCommand, GradeSubmissionCommand

**Gradebook Commands**:
- RecordGradeCommand, UpdateGradeCommand, BulkUpdateGradesCommand
- CreateAssignmentCommand, PublishAssignmentCommand

**Attendance Commands**:
- MarkAttendanceCommand, BulkMarkAttendanceCommand
- RecordAbsenceCommand, RecordTardyCommand

**Special Programs Commands**:
- CreateIEPCommand, UpdateIEPCommand, AddIEPGoalCommand
- Create504Command, AddAccommodationCommand
- EnrollESLCommand, UpdateProficiencyCommand

---

## Event Monitoring & Observability

### Health Checks
**Endpoint**: `/health/events`

**Metrics**:
- Stream pending count
- Consumer group lag
- Last event timestamp
- DLQ message count
- Active consumers
- Processing rate (events/sec)

### Prometheus Metrics
- `eventbus_published_total` - Events published
- `eventbus_consumed_total` - Events consumed
- `eventbus_failed_total` - Failed events
- `eventbus_retry_total` - Retry attempts
- `eventbus_dlq_total` - DLQ messages
- `eventbus_processing_duration_seconds` - Processing time

### Alerting
- **High DLQ count** → PagerDuty alert
- **Consumer lag** > 1000 → Slack notification
- **Failed events** > 5% → Email alert
- **Processing time** > 5s → Warning log

---

## Best Practices

### Event Design
1. **Immutable**: Events cannot be changed after creation
2. **Past Tense**: Event names describe what happened
3. **Rich Context**: Include all necessary data in event
4. **Versioning**: Support schema evolution
5. **Correlation**: Track event chains with correlation IDs

### Handler Design
1. **Idempotent**: Handlers can process same event multiple times
2. **Side-Effect Only**: No business logic, only reactions
3. **Fast Processing**: < 1s per event
4. **Error Handling**: Retry transient, DLQ permanent failures
5. **FERPA Compliant**: Log all student data access

### Performance Optimization
1. **Batch Processing**: Process related events in batches
2. **Async I/O**: Non-blocking database/API calls
3. **Connection Pooling**: Reuse database connections
4. **Caching**: Cache frequently accessed data
5. **Dead Letter Queue**: Prevent poison messages

---

## Related Documentation

- [Architecture Overview](./overview.md) - System architecture
- [API Integration](./api-integration.md) - API patterns
- [Database Schema](./database.md) - Data models
