# GraphQL API Reference

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**GraphQL Version**: 16.8  
**Schema**: Code-first with GraphQL Code Generator

---

## Overview

SkillCore's GraphQL API provides a **type-safe**, **performant**, and **fully-documented** interface for all educational operations. The API supports **127 operations** (queries, mutations, subscriptions) across **15 bounded contexts**.

**Base URL**: `https://api.skillcore.app/graphql`  
**WebSocket URL**: `wss://api.skillcore.app/graphql`

---

## Authentication

All GraphQL requests require **JWT authentication** via the `Authorization` header:

```http
POST /graphql
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "query": "query { ... }",
  "variables": { ... }
}
```

**Token Refresh**:
```graphql
mutation RefreshToken($refreshToken: String!) {
  refreshToken(refreshToken: $refreshToken) {
    accessToken
    refreshToken
    expiresIn
  }
}
```

---

## Schema Organization

### Bounded Contexts

The schema is organized into **15 DDD bounded contexts**:

1. **Authentication** - Login, logout, MFA, sessions
2. **Rostering** - Organizations, academic sessions, courses, classes
3. **Demographics** - Students, teachers, parents, enrollments
4. **Gradebook** - Categories, assignments, grades, score scales
5. **Assessment** - Tests, quizzes, questions, submissions
6. **Attendance** - Daily attendance, absence tracking
7. **Special Programs** - IEP, Section 504, ESL, Gifted
8. **Intervention** - MTSS/RTI, progress monitoring
9. **Communication** - Messages, announcements, conferences
10. **Learning Resources** - Curriculum, materials, standards
11. **Lesson Planning** - Lesson plans, unit plans
12. **Behavior Management** - Behavior incidents, interventions
13. **Analytics** - Student analytics, class performance, trends
14. **Notifications** - Preferences, device tokens, delivery logs
15. **Payments** - Fees, invoices, transactions

---

## Core Types

### Scalars

```graphql
scalar DateTime   # ISO 8601 datetime: "2025-11-17T10:30:00Z"
scalar Date       # ISO 8601 date: "2025-11-17"
scalar JSON       # Arbitrary JSON object
scalar Upload     # File upload (multipart/form-data)
```

### Pagination

**Cursor-based Pagination** (Relay spec):
```graphql
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type StudentConnection {
  edges: [StudentEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}

type StudentEdge {
  node: Student!
  cursor: String!
}

# Usage
query GetStudents($first: Int, $after: String) {
  students(first: $first, after: $after) {
    edges {
      node {
        id
        givenName
        familyName
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

---

## Rostering Context

### Queries

**Get Organizations**:
```graphql
query GetOrganizations($filter: OrgFilter) {
  organizations(filter: $filter) {
    edges {
      node {
        id
        sourcedId
        type  # board, district, school, department
        name
        identifier
        parent {
          id
          name
        }
        children {
          id
          name
          type
        }
      }
    }
  }
}
```

**Get Academic Sessions**:
```graphql
query GetAcademicSessions($schoolId: ID!) {
  academicSessions(filter: { schoolId: $schoolId }) {
    edges {
      node {
        id
        sourcedId
        title
        type  # schoolYear, semester, term, gradingPeriod
        startDate
        endDate
        status  # active, inactive
        parent {
          id
          title
        }
        children {
          id
          title
          type
          startDate
          endDate
        }
      }
    }
  }
}
```

**Get Classes**:
```graphql
query GetClasses($filter: ClassFilter, $orderBy: ClassOrderBy) {
  classes(filter: $filter, orderBy: $orderBy) {
    edges {
      node {
        id
        sourcedId
        title
        classCode
        classType  # homeroom, scheduled
        course {
          id
          title
          subject
          gradeLevel
        }
        primaryTeacher {
          id
          givenName
          familyName
          email
        }
        coTeachers {
          id
          givenName
          familyName
        }
        enrollmentCount
        maxStudents
        academicSession {
          id
          title
          startDate
          endDate
        }
        school {
          id
          name
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

### Mutations

**Create Class**:
```graphql
mutation CreateClass($input: CreateClassInput!) {
  createClass(input: $input) {
    class {
      id
      sourcedId
      title
      classCode
      course {
        id
        title
      }
      primaryTeacher {
        id
        givenName
        familyName
      }
    }
    errors {
      field
      message
    }
  }
}

input CreateClassInput {
  title: String!
  classCode: String
  classType: ClassType!
  courseId: ID!
  schoolId: ID!
  academicSessionId: ID!
  primaryTeacherId: ID
  maxStudents: Int
}
```

---

## Demographics Context

### Queries

**Get Students**:
```graphql
query GetStudents($filter: StudentFilter, $first: Int, $after: String) {
  students(filter: $filter, first: $first, after: $after) {
    edges {
      node {
        id
        sourcedId
        studentNumber
        givenName
        familyName
        preferredName
        email
        phone
        gradeLevel
        dateOfBirth
        gender
        ethnicity
        primaryLanguage
        enrollmentStatus  # active, inactive, graduated, transferred, withdrawn
        currentSchool {
          id
          name
        }
        homeroom
        hasIEP
        has504
        isESL
        isGifted
        currentGPA
        attendanceRate
        enrollments(filter: { status: ACTIVE }) {
          edges {
            node {
              id
              class {
                id
                title
                primaryTeacher {
                  id
                  givenName
                  familyName
                }
              }
            }
          }
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}

input StudentFilter {
  schoolId: ID
  gradeLevel: String
  enrollmentStatus: EnrollmentStatus
  hasIEP: Boolean
  has504: Boolean
  isESL: Boolean
  isGifted: Boolean
  search: String  # Search by name or student number
}
```

**Get Family Members**:
```graphql
query GetFamilyMembers($studentId: ID!) {
  student(id: $studentId) {
    id
    givenName
    familyName
    parents {
      id
      givenName
      familyName
      email
      phone
      relationshipType  # parent, guardian, emergency_contact
      isPrimary
      canViewGrades
      canViewAttendance
      canReceiveAlerts
    }
    siblings {
      id
      givenName
      familyName
      gradeLevel
      currentSchool {
        id
        name
      }
    }
  }
}
```

### Mutations

**Create Student**:
```graphql
mutation CreateStudent($input: CreateStudentInput!) {
  createStudent(input: $input) {
    student {
      id
      sourcedId
      studentNumber
      givenName
      familyName
      gradeLevel
      enrollmentStatus
    }
    errors {
      field
      message
    }
  }
}

input CreateStudentInput {
  givenName: String!
  familyName: String!
  middleName: String
  preferredName: String
  studentNumber: String!
  dateOfBirth: Date!
  gender: String
  ethnicity: [String!]
  gradeLevel: String!
  schoolId: ID!
  email: String
  phone: String
}
```

**Link Parent to Student**:
```graphql
mutation LinkParent($input: LinkParentInput!) {
  linkParent(input: $input) {
    relationship {
      id
      parent {
        id
        givenName
        familyName
      }
      student {
        id
        givenName
        familyName
      }
      relationshipType
      isPrimary
      canViewGrades
      canViewAttendance
    }
    errors {
      field
      message
    }
  }
}

input LinkParentInput {
  parentId: ID!
  studentId: ID!
  relationshipType: RelationshipType!  # parent, guardian, emergency_contact
  isPrimary: Boolean
  canViewGrades: Boolean
  canViewAttendance: Boolean
  canReceiveAlerts: Boolean
}
```

---

## Gradebook Context

### Queries

**Get Gradebook**:
```graphql
query GetGradebook($classId: ID!, $academicSessionId: ID!) {
  gradebook(classId: $classId, academicSessionId: $academicSessionId) {
    class {
      id
      title
      course {
        title
        subject
      }
    }
    categories {
      id
      title
      weight
      dropLowest
      lineItems {
        id
        title
        dueDate
        resultValueMax
        status
      }
    }
    students {
      id
      givenName
      familyName
      studentNumber
      results {
        lineItemId
        score
        scorePercent
        letterGrade
        scoreStatus  # not_submitted, partially_graded, fully_graded, exempt
        gradedAt
        postedAt
      }
      categoryAverages {
        categoryId
        average
        weight
      }
      overallAverage
      letterGrade
    }
    academicSession {
      id
      title
      startDate
      endDate
    }
  }
}
```

**Get Student Grades**:
```graphql
query GetStudentGrades($studentId: ID!, $academicSessionId: ID!) {
  studentGrades(studentId: $studentId, academicSessionId: $academicSessionId) {
    student {
      id
      givenName
      familyName
      gradeLevel
    }
    classes {
      class {
        id
        title
        course {
          title
          subject
        }
        primaryTeacher {
          givenName
          familyName
        }
      }
      results {
        id
        lineItem {
          id
          title
          dueDate
          resultValueMax
          category {
            title
          }
        }
        score
        scorePercent
        letterGrade
        scoreStatus
        comment
        gradedAt
        postedAt
      }
      currentAverage
      letterGrade
      trend  # improving, declining, stable
    }
    overallGPA
    academicSession {
      id
      title
    }
  }
}
```

### Mutations

**Create Assignment**:
```graphql
mutation CreateLineItem($input: CreateLineItemInput!) {
  createLineItem(input: $input) {
    lineItem {
      id
      sourcedId
      title
      description
      assignDate
      dueDate
      resultValueMax
      category {
        id
        title
      }
      class {
        id
        title
      }
    }
    errors {
      field
      message
    }
  }
}

input CreateLineItemInput {
  title: String!
  description: String
  classId: ID!
  categoryId: ID!
  assignDate: DateTime
  dueDate: DateTime
  resultValueMax: Float!  # Points possible
}
```

**Submit Grade**:
```graphql
mutation SubmitGrade($input: SubmitGradeInput!) {
  submitGrade(input: $input) {
    result {
      id
      score
      scorePercent
      letterGrade
      scoreStatus
      comment
      gradedAt
      postedAt
      student {
        id
        givenName
        familyName
      }
      lineItem {
        id
        title
      }
    }
    errors {
      field
      message
    }
  }
}

input SubmitGradeInput {
  lineItemId: ID!
  studentId: ID!
  score: Float!
  scoreStatus: ScoreStatus!  # not_submitted, partially_graded, fully_graded, exempt
  comment: String
  postImmediately: Boolean  # If true, grade visible to student immediately
}
```

**Bulk Submit Grades**:
```graphql
mutation BulkSubmitGrades($input: BulkSubmitGradesInput!) {
  bulkSubmitGrades(input: $input) {
    results {
      id
      score
      student {
        id
        givenName
        familyName
      }
    }
    successCount
    errorCount
    errors {
      studentId
      message
    }
  }
}

input BulkSubmitGradesInput {
  lineItemId: ID!
  grades: [GradeInput!]!
  postImmediately: Boolean
}

input GradeInput {
  studentId: ID!
  score: Float!
  scoreStatus: ScoreStatus!
  comment: String
}
```

---

## Assessment Context

### Queries

**Get Assessments**:
```graphql
query GetAssessments($filter: AssessmentFilter) {
  assessments(filter: $filter) {
    edges {
      node {
        id
        title
        description
        assessmentType  # quiz, test, exam, project
        totalPoints
        timeLimit
        allowRetakes
        class {
          id
          title
        }
        questions {
          id
          questionType  # multiple_choice, true_false, short_answer, essay
          questionText
          points
          order
        }
        submissions {
          id
          student {
            id
            givenName
            familyName
          }
          score
          status  # in_progress, submitted, graded
          submittedAt
        }
        dueDate
        availableFrom
        availableUntil
      }
    }
  }
}
```

### Mutations

**Create Assessment**:
```graphql
mutation CreateAssessment($input: CreateAssessmentInput!) {
  createAssessment(input: $input) {
    assessment {
      id
      title
      assessmentType
      totalPoints
      questions {
        id
        questionText
        points
      }
    }
    errors {
      field
      message
    }
  }
}

input CreateAssessmentInput {
  title: String!
  description: String
  classId: ID!
  assessmentType: AssessmentType!
  totalPoints: Float!
  timeLimit: Int  # Minutes
  dueDate: DateTime
  availableFrom: DateTime
  availableUntil: DateTime
  allowRetakes: Boolean
  questions: [QuestionInput!]!
}

input QuestionInput {
  questionType: QuestionType!
  questionText: String!
  points: Float!
  order: Int!
  options: [String!]  # For multiple choice
  correctAnswer: String
}
```

---

## Communication Context

### Queries

**Get Announcements**:
```graphql
query GetAnnouncements($filter: AnnouncementFilter, $first: Int, $after: String) {
  announcements(filter: $filter, first: $first, after: $after) {
    edges {
      node {
        id
        title
        body
        priority  # urgent, high, normal, low
        category  # academic, sports, events, emergency
        author {
          id
          givenName
          familyName
        }
        targetType  # district, school, grade, class, role
        targetIds
        publishAt
        expiresAt
        isPinned
        viewCount
        readCount
        attachments {
          id
          fileName
          fileUrl
          fileSize
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Get Conversations**:
```graphql
query GetConversations($userId: ID!) {
  conversations(userId: $userId) {
    edges {
      node {
        id
        type  # direct, group
        subject
        participants {
          id
          givenName
          familyName
          role
        }
        lastMessage {
          id
          body
          sender {
            id
            givenName
          }
          createdAt
        }
        unreadCount
        lastMessageAt
      }
    }
  }
}
```

### Mutations

**Create Announcement**:
```graphql
mutation CreateAnnouncement($input: CreateAnnouncementInput!) {
  createAnnouncement(input: $input) {
    announcement {
      id
      title
      priority
      targetType
      publishAt
    }
    errors {
      field
      message
    }
  }
}

input CreateAnnouncementInput {
  title: String!
  body: String!
  priority: Priority!
  category: String
  targetType: TargetType!
  targetIds: [ID!]!
  publishAt: DateTime
  expiresAt: DateTime
  isPinned: Boolean
}
```

**Send Message**:
```graphql
mutation SendMessage($input: SendMessageInput!) {
  sendMessage(input: $input) {
    message {
      id
      subject
      body
      conversationId
      sender {
        id
        givenName
        familyName
      }
      createdAt
    }
    errors {
      field
      message
    }
  }
}

input SendMessageInput {
  recipientIds: [ID!]!
  subject: String
  body: String!
  conversationId: ID  # Optional: continue existing conversation
}
```

### Subscriptions

**New Message**:
```graphql
subscription OnNewMessage($userId: ID!) {
  messageReceived(userId: $userId) {
    id
    subject
    body
    sender {
      id
      givenName
      familyName
    }
    conversationId
    createdAt
  }
}
```

**New Announcement**:
```graphql
subscription OnNewAnnouncement($userId: ID!) {
  announcementPublished(userId: $userId) {
    id
    title
    body
    priority
    category
    author {
      givenName
      familyName
    }
    publishAt
  }
}
```

---

## Analytics Context

### Queries

**Get Student Analytics**:
```graphql
query GetStudentAnalytics($studentId: ID!, $academicSessionId: ID!) {
  studentAnalytics(studentId: $studentId, academicSessionId: $academicSessionId) {
    student {
      id
      givenName
      familyName
    }
    
    # Academic performance
    currentGPA
    previousGPA
    gpaChange
    trend  # improving, declining, stable
    
    # Attendance
    attendanceRate
    absenceCount
    tardyCount
    excusedAbsenceCount
    
    # Grades by subject
    gradesBySubject {
      subject
      average
      trend
      letterGrade
    }
    
    # Interventions
    activeInterventions {
      id
      type
      reason
      startDate
    }
    
    # Risk indicators
    riskLevel  # none, low, medium, high
    riskFactors
    
    academicSession {
      id
      title
    }
  }
}
```

**Get Class Performance**:
```graphql
query GetClassPerformance($classId: ID!, $academicSessionId: ID!) {
  classPerformance(classId: $classId, academicSessionId: $academicSessionId) {
    class {
      id
      title
      course {
        title
        subject
      }
    }
    
    # Overall metrics
    classAverage
    medianGrade
    passRate
    attendanceRate
    
    # Grade distribution
    gradeDistribution {
      letterGrade
      count
      percentage
    }
    
    # Assignment performance
    assignmentAverages {
      lineItemId
      title
      average
      submissionRate
      dueDate
    }
    
    # Student risk levels
    riskDistribution {
      level  # none, low, medium, high
      count
    }
    
    # Trends
    performanceTrend {
      date
      average
    }
    attendanceTrend {
      date
      rate
    }
    
    academicSession {
      id
      title
    }
  }
}
```

---

## Error Handling

### Error Response

All mutations return an `errors` field with structured error information:

```graphql
type MutationError {
  field: String      # Field that caused error (e.g., "email")
  message: String!   # Human-readable error message
  code: String       # Error code (e.g., "VALIDATION_ERROR", "UNAUTHORIZED")
}

type CreateStudentPayload {
  student: Student
  errors: [MutationError!]
}
```

**Example Error Response**:
```json
{
  "data": {
    "createStudent": {
      "student": null,
      "errors": [
        {
          "field": "email",
          "message": "Email address is already in use",
          "code": "DUPLICATE_EMAIL"
        },
        {
          "field": "studentNumber",
          "message": "Student number must be 6-10 digits",
          "code": "VALIDATION_ERROR"
        }
      ]
    }
  }
}
```

---

## Code Generation

### TypeScript Client

**Install**:
```bash
pnpm add @graphql-codegen/cli @graphql-codegen/typescript @graphql-codegen/typescript-operations @graphql-codegen/typescript-react-apollo
```

**codegen.yml**:
```yaml
schema: https://api.skillcore.app/graphql
documents: 'src/**/*.graphql'
generates:
  src/graphql/generated.ts:
    plugins:
      - typescript
      - typescript-operations
      - typescript-react-apollo
    config:
      withHooks: true
      withComponent: false
```

**Generated Hooks**:
```typescript
import { useGetStudentsQuery, useCreateStudentMutation } from '@/graphql/generated'

// Query hook
const { data, loading, error } = useGetStudentsQuery({
  variables: {
    filter: { schoolId: 'school-123', gradeLevel: '10' },
    first: 50,
  }
})

// Mutation hook
const [createStudent, { loading, error }] = useCreateStudentMutation({
  onCompleted: (data) => {
    console.log('Student created:', data.createStudent.student)
  }
})
```

---

## Performance & Best Practices

### Query Complexity

Maximum query complexity: **1000 points**

- Field access: **1 point**
- Connection: **10 points**
- Nested connection: **Multiplied**

**Example**:
```graphql
query ExpensiveQuery {
  students(first: 100) {  # 10 points
    edges {
      node {
        id                 # 1 point × 100 = 100 points
        enrollments {      # 10 points × 100 = 1000 points
          edges {
            node {
              class {      # Over limit!
                ...
              }
            }
          }
        }
      }
    }
  }
}
# Total: ~1110 points (REJECTED)
```

### Rate Limiting

- **Queries**: 100 requests/minute
- **Mutations**: 60 requests/minute
- **Subscriptions**: 10 concurrent connections

### Caching

**Apollo Client Configuration**:
```typescript
const cache = new InMemoryCache({
  typePolicies: {
    Query: {
      fields: {
        students: {
          keyArgs: ['filter'],
          merge: (existing, incoming) => ({
            ...incoming,
            edges: [...(existing?.edges || []), ...incoming.edges],
          }),
        },
      },
    },
  },
})
```

---

## Related Documentation

- [REST API](./rest.md) - RESTful endpoints
- [WebSocket API](./websocket.md) - Real-time subscriptions
- [Authentication](../architecture/security.md) - JWT authentication

---

**API Statistics** (as of November 2025):
- **Total Operations**: 127 (65 queries, 52 mutations, 10 subscriptions)
- **Schema Size**: 15,000+ lines
- **Average Response Time**: 120ms
- **Uptime**: 99.99%
