# Special Programs Management

**Last Updated**: November 16, 2025  
**Status**: ✅ Production-Ready (100% Backend + UI Complete)

---

## Overview

Comprehensive special education and enrichment program management system supporting IEP, Section 504, ESL, and Gifted programs with IDEA/Section 504 compliance automation and FERPA-compliant workflows.

---

## Supported Programs

### 1. Individualized Education Program (IEP)

**Purpose**: Specialized instruction for students with disabilities

**Features**:
- ✅ IEP plan creation with goals and accommodations
- ✅ Annual review tracking (IDEA compliance)
- ✅ Progress monitoring with measurable objectives
- ✅ Service tracking (speech, OT, PT, counseling)
- ✅ Team member management (parents, teachers, specialists)
- ✅ Accommodation implementation tracking
- ✅ Document management and PDF export
- ✅ Auto-intervention creation (Tier 2)

**IDEA Compliance**:
- 30/60/90 day review reminders
- Audit logging for all modifications
- Parent consent tracking
- Service hour verification
- Compliance reports

**Event Integration** (10 events):
- `IEPCreated`, `IEPUpdated`, `IEPArchived`
- `IEPReviewScheduled` - Automated compliance reminders
- `IEPGoalAdded`, `IEPGoalUpdated`, `IEPGoalArchived`
- `IEPAccommodationModified` - Teacher notification triggered
- `IEPServiceHoursUpdated`
- `IEPTeamMeetingScheduled`

**Handlers** (3 handlers):
1. **CoordinatorNotificationHandler** - Notifies IEP coordinator
2. **TeacherAccommodationHandler** - Alerts teachers of accommodations
3. **IEPInterventionHandler** - Auto-creates Tier 2 interventions

---

### 2. Section 504 Plans

**Purpose**: Accommodations for students with disabilities (not requiring specialized instruction)

**Features**:
- ✅ 504 plan creation with accommodation catalog
- ✅ Multi-category accommodations (classroom, testing, behavioral, physical)
- ✅ Review cycle management (annual/3-year)
- ✅ Team collaboration (counselor, teachers, parents, nurse)
- ✅ Functional limitation tracking
- ✅ Accommodation effectiveness monitoring
- ✅ Compliance documentation

**Common Accommodations**:
- Classroom: Extended time, preferential seating, note-taking support
- Testing: Separate setting, read-aloud, oral responses
- Behavioral: Breaks, sensory tools, behavior plan
- Physical: Accessibility, elevator pass, modified PE

**Section 504 Compliance**:
- Annual review scheduling
- Reevaluation every 3 years
- Parent notification requirements
- Accommodation change tracking
- District-level compliance reporting

**Event Integration** (8 events):
- `Section504PlanCreated`, `Section504PlanUpdated`, `Section504Archived`
- `Section504ReviewScheduled` - Compliance automation
- `Section504AccommodationAdded`, `Section504AccommodationRemoved`
- `Section504AccommodationUpdated` - Teacher alerts
- `Section504TeamMeetingScheduled`

**Handlers** (3 handlers):
1. **CoordinatorNotificationHandler** - Notifies 504 coordinator
2. **TeacherAccommodationHandler** - Alerts teachers
3. **ComplianceReportHandler** - Section 504 audit logging

---

### 3. English as a Second Language (ESL)

**Purpose**: Language support for English language learners

**Features**:
- ✅ ESL enrollment with native language tracking
- ✅ Proficiency level assessment (6 levels: Entering → Reaching)
- ✅ Service model configuration (push-in, pull-out, co-teaching)
- ✅ Language assessment tracking (WIDA, CELDT, ELPA21)
- ✅ Progress monitoring and goal setting
- ✅ Service hours and schedule management
- ✅ Exit criteria tracking
- ✅ Parent communication (multilingual support)

**Proficiency Levels** (WIDA Framework):
1. **Level 1 - Entering**: Minimal English comprehension
2. **Level 2 - Emerging**: Basic phrases and simple sentences
3. **Level 3 - Developing**: Simple conversations with support
4. **Level 4 - Expanding**: Grade-level content with scaffolding
5. **Level 5 - Bridging**: Near grade-level proficiency
6. **Level 6 - Reaching**: Full grade-level proficiency

**Service Models**:
- **Push-In**: ESL teacher in regular classroom
- **Pull-Out**: Small group instruction outside classroom
- **Co-Teaching**: ESL and content teachers collaborate
- **Sheltered English**: Modified instruction for ELLs

**Event Integration** (4 events):
- `ESLEnrollmentCreated` - Student enrolled in ESL
- `ESLServicesUpdated` - Service model or hours changed
- `ESLProgramExited` - Student meets exit criteria
- `ESLProficiencyLevelChanged` - Level progression

**Handlers** (2 handlers):
1. **CoordinatorNotificationHandler** - Notifies ESL coordinator
2. **StudentProfileUpdateHandler** - Updates student metadata

---

### 4. Gifted & Talented Programs

**Purpose**: Enrichment for academically advanced students

**Features**:
- ✅ Nomination and identification workflow
- ✅ Multi-area giftedness tracking (8 areas)
- ✅ Differentiation plan creation
- ✅ Enrichment activity management
- ✅ Acceleration tracking (grade/subject)
- ✅ Competition and achievement logging
- ✅ Portfolio and evidence collection
- ✅ Parent involvement tracking

**Areas of Giftedness**:
1. **Intellectual** - High cognitive ability
2. **Creative** - Original thinking and problem-solving
3. **Academic** - Specific subject excellence
4. **Leadership** - Natural leadership qualities
5. **Arts** - Visual/performing arts talent
6. **Music** - Musical ability and performance
7. **Psychomotor** - Athletic/physical coordination
8. **Other** - Other exceptional abilities

**Identification Process**:
1. **Nomination** - Teacher/parent/self nomination
2. **Screening** - Standardized tests, grades, work samples
3. **Assessment** - IQ tests, achievement tests, observations
4. **Placement** - Committee review and program assignment

**Enrichment Options**:
- Independent study projects
- Mentorship programs
- Advanced/AP courses
- Academic competitions
- Dual enrollment (college courses)
- Summer programs
- Research opportunities

**Event Integration** (5 events):
- `GiftedNominationCreated` - Nomination submitted
- `GiftedIdentificationComplete` - Placement decision made
- `GiftedEnrollmentCreated` - Student enrolled
- `GiftedServicesUpdated` - Services modified
- `GiftedProgramExited` - Student exited program

**Handlers** (2 handlers):
1. **CoordinatorNotificationHandler** - Notifies gifted coordinator
2. **StudentProfileUpdateHandler** - Updates student profile

---

## Cross-Program Features

### Coordinator Dashboard

**Overview Metrics**:
- Total active programs by type
- Students served across all programs
- Pending reviews and assessments
- Upcoming meetings and deadlines

**Filtering**:
- By program type (IEP, 504, ESL, Gifted)
- By status (active, draft, review, expired)
- By coordinator
- By school/grade level

### Team Collaboration

**Team Members**:
- Program Coordinator (lead)
- Classroom Teachers (implementation)
- Special Education Teachers
- Counselors
- Parents/Guardians
- Support Staff (OT, PT, Speech)
- School Psychologists
- Administrators

**Collaboration Features**:
- Team member assignments
- Meeting scheduling
- Document sharing
- Progress notes
- Communication logs

### Document Management

**Document Types**:
- Initial evaluations
- Assessment reports
- IEP/504 plans
- Progress reports
- Parent consent forms
- Service logs
- Meeting minutes

**Features**:
- PDF generation and export
- Version history
- Secure storage
- Parent portal access
- Compliance documentation

### Progress Monitoring

**Progress Notes**:
- Date-stamped entries
- Goal/objective tracking
- Quantitative data entry
- Narrative observations
- Evidence attachments

**Data Tracking**:
- Goal progress charts
- Service hour logs
- Accommodation usage
- Behavioral data
- Academic performance

---

## Event-Driven Automation

### Automatic Interventions

**IEP → Intervention Integration**:
```typescript
// When IEP goal added → Create Tier 2 intervention
IEPGoalAdded → IEPInterventionHandler → Create Intervention
```

**Intervention Details**:
- Tier: Tier 2 (targeted support)
- Type: Academic or Behavioral (based on IEP goal)
- Strategies: Aligned with IEP accommodations
- Frequency: Based on service hours
- Progress monitoring: Weekly/bi-weekly

### Teacher Notifications

**Accommodation Alerts**:
```typescript
// When accommodation added/modified → Notify teachers
Section504AccommodationAdded → TeacherAccommodationHandler
  → Query all enrolled classes
  → Get all teachers for those classes
  → Send accommodation notification to each teacher
```

**Notification Content**:
- Student name
- Accommodation details
- Implementation instructions
- Effective date
- Contact person for questions

### Compliance Automation

**Review Reminders**:
```typescript
// Scheduled job runs daily
Daily Cron → Check IEPs due for review (30/60/90 days)
  → IEPReviewScheduled event
  → CoordinatorNotificationHandler
  → Email + Push notification
```

**Compliance Reports**:
```typescript
// When review scheduled → Audit log entry
IEPReviewScheduled → ComplianceReportHandler
  → Create AuditLog entry
  → Track for district compliance reporting
```

---

## GraphQL Integration

### Queries

**IEP Queries**:
```graphql
query GetIEPs($filters: IEPFilters) {
  ieps(filters: $filters) {
    id
    student { id name gradeLevel }
    status
    disability
    goals { id description progress }
    accommodations { category description }
    team { role member { name email } }
  }
}

query GetIEP($id: ID!) {
  iep(id: $id) {
    id
    student { id name }
    goals { id description measurableObjective progress }
    services { type frequency duration provider }
    documents { id name uploadDate url }
  }
}
```

**Section 504 Queries**:
```graphql
query Get504Plans($filters: Section504Filters) {
  section504Plans(filters: $filters) {
    id
    student { id name }
    status
    disabilityCondition
    accommodations { category description effectiveness }
    reviewDate
    nextReviewDate
  }
}
```

### Mutations

**IEP Mutations**:
```graphql
mutation CreateIEP($input: CreateIEPInput!) {
  createIEP(input: $input) {
    id
    status
    student { id name }
  }
}

mutation AddIEPGoal($iepId: ID!, $goal: IEPGoalInput!) {
  addIEPGoal(iepId: $iepId, goal: $goal) {
    id
    goals { id description }
  }
}

mutation ScheduleIEPReview($iepId: ID!, $date: DateTime!) {
  scheduleIEPReview(iepId: $iepId, date: $date) {
    id
    nextReviewDate
  }
}
```

**Section 504 Mutations**:
```graphql
mutation Create504Plan($input: Create504Input!) {
  create504Plan(input: $input) {
    id
    student { id name }
  }
}

mutation AddAccommodation($planId: ID!, $accommodation: AccommodationInput!) {
  addAccommodation(planId: $planId, accommodation: $accommodation) {
    id
    accommodations { id category description }
  }
}
```

---

## FERPA Compliance

### Access Control
- ✅ Coordinators see all programs they manage
- ✅ Teachers see accommodations for their students only
- ✅ Parents see their child's program only
- ✅ Students 18+ can access their own records
- ✅ Audit logging for all access

### Data Protection
- ✅ Sensitive disability info encrypted
- ✅ Document storage with access controls
- ✅ Secure PDF generation (watermarked)
- ✅ No student info in event payloads
- ✅ 7-year retention for compliance

### Parent Rights
- ✅ Parent consent required for evaluations
- ✅ Parent access to all documents
- ✅ Meeting participation rights
- ✅ Dispute resolution process
- ✅ Privacy notice acknowledgment

---

## Related Documentation

- [Event Architecture](../architecture/events.md) - Event system details
- [Intervention System](./interventions.md) - Counselor interventions
- [Features Index](./README.md) - All features
