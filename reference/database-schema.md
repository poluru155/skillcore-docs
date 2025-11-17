# Database Schema Reference

**Last Updated**: November 17, 2025  
**Database**: PostgreSQL 16  
**ORM**: Prisma 5.7  
**Total Tables**: 78  
**Total Relationships**: 156

---

## Overview

This document provides a **complete reference** for the SkillCore database schema, including all tables, relationships, indexes, and multi-tenancy patterns.

---

## Multi-Tenancy Architecture

### Tenant Isolation

Every table (except `Tenant` and `User`) includes:

```prisma
tenantId String @db.Uuid
tenant   Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

@@index([tenantId])
```

**Benefits**:
- Complete data isolation between districts
- Cascade deletion when tenant removed
- Optimized queries with tenantId indexes

---

## Core Schema

### 1. Tenant & Organization

```prisma
model Tenant {
  id          String   @id @default(uuid()) @db.Uuid
  name        String
  subdomain   String   @unique
  domain      String?
  
  // Status
  status      TenantStatus @default(ACTIVE)
  tier        TenantTier   @default(FREE)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations (all cascade delete)
  users       User[]
  schools     School[]
  students    Student[]
  teachers    Teacher[]
  classes     Class[]
  // ... all other tables
  
  @@index([subdomain])
  @@index([status])
}

enum TenantStatus {
  ACTIVE
  SUSPENDED
  TRIAL
  CANCELLED
}

enum TenantTier {
  FREE
  STARTER
  PROFESSIONAL
  ENTERPRISE
}

model School {
  id          String   @id @default(uuid()) @db.Uuid
  sourcedId   String   @unique
  
  // Basic Info
  name        String
  identifier  String?
  type        SchoolType
  
  // Contact
  address     String?
  city        String?
  state       String?
  zip         String?
  phone       String?
  email       String?
  
  // Metadata
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations
  classes     Class[]
  enrollments SchoolEnrollment[]
  
  @@index([tenantId])
  @@index([identifier])
}

enum SchoolType {
  ELEMENTARY
  MIDDLE
  HIGH
  K8
  K12
}
```

---

### 2. User & Authentication

```prisma
model User {
  id            String    @id @default(uuid()) @db.Uuid
  email         String    @unique
  passwordHash  String
  
  // Profile
  givenName     String
  familyName    String
  preferredName String?
  phone         String?
  
  // Role & Status
  role          UserRole
  status        UserStatus @default(ACTIVE)
  
  // Email Verification
  emailVerified Boolean   @default(false)
  verifiedAt    DateTime?
  
  // Password Reset
  resetToken    String?   @unique
  resetExpires  DateTime?
  
  // Sessions
  sessions      Session[]
  
  // Multi-tenant
  tenantId      String    @db.Uuid
  tenant        Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  lastLoginAt   DateTime?
  
  @@index([tenantId])
  @@index([email])
  @@index([role])
}

enum UserRole {
  SUPER_ADMIN
  DISTRICT_ADMIN
  SCHOOL_ADMIN
  TEACHER
  PARENT
  STUDENT
}

enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
}

model Session {
  id           String   @id @default(uuid()) @db.Uuid
  userId       String   @db.Uuid
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  token        String   @unique
  expiresAt    DateTime
  ipAddress    String?
  userAgent    String?
  
  createdAt    DateTime @default(now())
  
  @@index([userId])
  @@index([token])
  @@index([expiresAt])
}
```

---

### 3. Student Demographics

```prisma
model Student {
  id                String    @id @default(uuid()) @db.Uuid
  sourcedId         String    @unique
  
  // Personal Info
  givenName         String
  familyName        String
  middleName        String?
  preferredName     String?
  
  // Identifiers
  studentNumber     String
  stateId           String?
  email             String?
  phone             String?
  
  // Demographics
  birthDate         DateTime?
  gender            Gender?
  gradeLevel        String
  
  // Race & Ethnicity (multi-select)
  raceEthnicity     RaceEthnicity[]
  hispanicLatino    Boolean?
  
  // Language
  primaryLanguage   String?
  homeLanguage      String?
  
  // Programs
  hasIEP            Boolean   @default(false)
  has504            Boolean   @default(false)
  isESL             Boolean   @default(false)
  isGifted          Boolean   @default(false)
  
  // Contact
  address           String?
  city              String?
  state             String?
  zip               String?
  
  // Status
  status            StudentStatus @default(ACTIVE)
  
  // Photo
  photoUrl          String?
  
  // Multi-tenant
  tenantId          String    @db.Uuid
  tenant            Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  
  // Relations
  enrollments       Enrollment[]
  schoolEnrollments SchoolEnrollment[]
  grades            Grade[]
  attendanceRecords AttendanceRecord[]
  guardians         StudentGuardian[]
  siblings          StudentSibling[]  @relation("Student1")
  siblingsOf        StudentSibling[]  @relation("Student2")
  assessmentScores  AssessmentScore[]
  
  @@index([tenantId])
  @@index([studentNumber])
  @@index([gradeLevel])
  @@index([status])
}

enum Gender {
  MALE
  FEMALE
  NON_BINARY
  PREFER_NOT_TO_SAY
}

enum RaceEthnicity {
  AMERICAN_INDIAN_ALASKA_NATIVE
  ASIAN
  BLACK_AFRICAN_AMERICAN
  NATIVE_HAWAIIAN_PACIFIC_ISLANDER
  WHITE
  TWO_OR_MORE_RACES
}

enum StudentStatus {
  ACTIVE
  INACTIVE
  GRADUATED
  TRANSFERRED
  WITHDRAWN
}

model StudentGuardian {
  id              String   @id @default(uuid()) @db.Uuid
  
  studentId       String   @db.Uuid
  student         Student  @relation(fields: [studentId], references: [id], onDelete: Cascade)
  
  guardianId      String   @db.Uuid
  guardian        Parent   @relation(fields: [guardianId], references: [id], onDelete: Cascade)
  
  relationshipType RelationshipType
  isPrimary       Boolean  @default(false)
  hasPickupAuth   Boolean  @default(true)
  hasDataAccess   Boolean  @default(true)
  
  tenantId        String   @db.Uuid
  tenant          Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  createdAt       DateTime @default(now())
  
  @@unique([studentId, guardianId])
  @@index([tenantId])
  @@index([studentId])
  @@index([guardianId])
}

enum RelationshipType {
  MOTHER
  FATHER
  GUARDIAN
  GRANDPARENT
  STEP_PARENT
  FOSTER_PARENT
  OTHER
}

model StudentSibling {
  id         String   @id @default(uuid()) @db.Uuid
  
  student1Id String   @db.Uuid
  student1   Student  @relation("Student1", fields: [student1Id], references: [id], onDelete: Cascade)
  
  student2Id String   @db.Uuid
  student2   Student  @relation("Student2", fields: [student2Id], references: [id], onDelete: Cascade)
  
  tenantId   String   @db.Uuid
  tenant     Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  createdAt  DateTime @default(now())
  
  @@unique([student1Id, student2Id])
  @@index([tenantId])
}
```

---

### 4. Teacher & Staff

```prisma
model Teacher {
  id               String    @id @default(uuid()) @db.Uuid
  sourcedId        String    @unique
  
  // Personal Info
  givenName        String
  familyName       String
  preferredName    String?
  
  // Contact
  email            String
  phone            String?
  
  // Employment
  employeeNumber   String?
  department       String?
  title            String?
  
  // Status
  status           TeacherStatus @default(ACTIVE)
  
  // Photo
  photoUrl         String?
  
  // Multi-tenant
  tenantId         String    @db.Uuid
  tenant           Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt
  
  // Relations
  classes          Class[]
  grades           Grade[]
  attendanceRecords AttendanceRecord[]
  
  @@index([tenantId])
  @@index([email])
  @@index([employeeNumber])
}

enum TeacherStatus {
  ACTIVE
  INACTIVE
  ON_LEAVE
  RETIRED
}
```

---

### 5. Parent/Guardian

```prisma
model Parent {
  id            String    @id @default(uuid()) @db.Uuid
  sourcedId     String    @unique
  
  // Personal Info
  givenName     String
  familyName    String
  preferredName String?
  
  // Contact
  email         String
  phone         String?
  phoneWork     String?
  
  // Address
  address       String?
  city          String?
  state         String?
  zip           String?
  
  // Preferences
  preferredContactMethod ContactMethod @default(EMAIL)
  language      String?
  
  // Multi-tenant
  tenantId      String    @db.Uuid
  tenant        Tenant    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  // Relations
  students      StudentGuardian[]
  
  @@index([tenantId])
  @@index([email])
}

enum ContactMethod {
  EMAIL
  PHONE
  SMS
  APP
}
```

---

### 6. Class & Enrollment

```prisma
model Class {
  id          String   @id @default(uuid()) @db.Uuid
  sourcedId   String   @unique
  
  // Basic Info
  title       String
  courseCode  String?
  classCode   String?
  
  // Subject & Grade
  subject     Subject
  gradeLevel  String?
  
  // Schedule
  period      String?
  room        String?
  
  // Academic Session
  sessionId   String   @db.Uuid
  session     AcademicSession @relation(fields: [sessionId], references: [id])
  
  // Teacher
  teacherId   String   @db.Uuid
  teacher     Teacher  @relation(fields: [teacherId], references: [id])
  
  // School
  schoolId    String   @db.Uuid
  school      School   @relation(fields: [schoolId], references: [id])
  
  // Status
  status      ClassStatus @default(ACTIVE)
  
  // Multi-tenant
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations
  enrollments Enrollment[]
  assignments Assignment[]
  attendanceRecords AttendanceRecord[]
  
  @@index([tenantId])
  @@index([teacherId])
  @@index([sessionId])
  @@index([subject])
}

enum Subject {
  ENGLISH
  MATH
  SCIENCE
  SOCIAL_STUDIES
  HISTORY
  GEOGRAPHY
  FOREIGN_LANGUAGE
  COMPUTER_SCIENCE
  PHYSICAL_EDUCATION
  HEALTH
  ART
  MUSIC
  OTHER
}

enum ClassStatus {
  ACTIVE
  INACTIVE
  ARCHIVED
}

model Enrollment {
  id          String   @id @default(uuid()) @db.Uuid
  sourcedId   String   @unique
  
  // Student & Class
  studentId   String   @db.Uuid
  student     Student  @relation(fields: [studentId], references: [id], onDelete: Cascade)
  
  classId     String   @db.Uuid
  class       Class    @relation(fields: [classId], references: [id], onDelete: Cascade)
  
  // Dates
  beginDate   DateTime
  endDate     DateTime?
  
  // Status
  status      EnrollmentStatus @default(ACTIVE)
  
  // Multi-tenant
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@unique([studentId, classId])
  @@index([tenantId])
  @@index([studentId])
  @@index([classId])
  @@index([status])
}

enum EnrollmentStatus {
  ACTIVE
  INACTIVE
  DROPPED
  COMPLETED
}
```

---

### 7. Academic Sessions

```prisma
model AcademicSession {
  id          String   @id @default(uuid()) @db.Uuid
  sourcedId   String   @unique
  
  // Basic Info
  title       String
  type        SessionType
  
  // Dates
  startDate   DateTime
  endDate     DateTime
  
  // Hierarchy
  parentId    String?  @db.Uuid
  parent      AcademicSession? @relation("SessionHierarchy", fields: [parentId], references: [id])
  children    AcademicSession[] @relation("SessionHierarchy")
  
  // School Year
  schoolYear  String
  
  // Status
  status      SessionStatus @default(ACTIVE)
  
  // Multi-tenant
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations
  classes     Class[]
  gradingPeriods GradingPeriod[]
  
  @@index([tenantId])
  @@index([type])
  @@index([schoolYear])
}

enum SessionType {
  SCHOOL_YEAR
  SEMESTER
  TERM
  GRADING_PERIOD
}

enum SessionStatus {
  ACTIVE
  INACTIVE
  ARCHIVED
}

model GradingPeriod {
  id          String   @id @default(uuid()) @db.Uuid
  
  // Basic Info
  name        String
  startDate   DateTime
  endDate     DateTime
  
  // Academic Session
  sessionId   String   @db.Uuid
  session     AcademicSession @relation(fields: [sessionId], references: [id])
  
  // Status
  isActive    Boolean  @default(true)
  isClosed    Boolean  @default(false)
  
  // Multi-tenant
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@index([tenantId])
  @@index([sessionId])
}
```

---

### 8. Gradebook

```prisma
model Assignment {
  id            String   @id @default(uuid()) @db.Uuid
  sourcedId     String   @unique
  
  // Basic Info
  title         String
  description   String?
  
  // Grading
  maxPoints     Decimal  @db.Decimal(10, 2)
  weight        Decimal? @db.Decimal(5, 2)
  
  // Category
  categoryId    String?  @db.Uuid
  category      GradeCategory? @relation(fields: [categoryId], references: [id])
  
  // Dates
  assignedDate  DateTime
  dueDate       DateTime?
  
  // Settings
  isPublished   Boolean  @default(false)
  allowLate     Boolean  @default(true)
  latePenalty   Decimal? @db.Decimal(5, 2)
  
  // Class
  classId       String   @db.Uuid
  class         Class    @relation(fields: [classId], references: [id], onDelete: Cascade)
  
  // Multi-tenant
  tenantId      String   @db.Uuid
  tenant        Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  
  // Relations
  grades        Grade[]
  
  @@index([tenantId])
  @@index([classId])
  @@index([categoryId])
}

model Grade {
  id            String   @id @default(uuid()) @db.Uuid
  sourcedId     String   @unique
  
  // Student & Assignment
  studentId     String   @db.Uuid
  student       Student  @relation(fields: [studentId], references: [id], onDelete: Cascade)
  
  assignmentId  String   @db.Uuid
  assignment    Assignment @relation(fields: [assignmentId], references: [id], onDelete: Cascade)
  
  // Score
  score         Decimal? @db.Decimal(10, 2)
  scorePercent  Decimal? @db.Decimal(5, 2)
  letterGrade   String?
  
  // Status
  scoreStatus   ScoreStatus @default(NOT_SUBMITTED)
  
  // Comments
  comment       String?
  
  // Dates
  gradedAt      DateTime?
  postedAt      DateTime?
  
  // Graded By
  gradedById    String?  @db.Uuid
  gradedBy      Teacher? @relation(fields: [gradedById], references: [id])
  
  // Multi-tenant
  tenantId      String   @db.Uuid
  tenant        Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  
  @@unique([studentId, assignmentId])
  @@index([tenantId])
  @@index([studentId])
  @@index([assignmentId])
  @@index([scoreStatus])
}

enum ScoreStatus {
  NOT_SUBMITTED
  PARTIALLY_GRADED
  FULLY_GRADED
  EXEMPT
}

model GradeCategory {
  id          String   @id @default(uuid()) @db.Uuid
  
  // Basic Info
  name        String
  weight      Decimal  @db.Decimal(5, 2)
  dropLowest  Int      @default(0)
  
  // Multi-tenant
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations
  assignments Assignment[]
  
  @@index([tenantId])
}
```

---

### 9. Attendance

```prisma
model AttendanceRecord {
  id          String   @id @default(uuid()) @db.Uuid
  
  // Student, Class, Teacher
  studentId   String   @db.Uuid
  student     Student  @relation(fields: [studentId], references: [id], onDelete: Cascade)
  
  classId     String?  @db.Uuid
  class       Class?   @relation(fields: [classId], references: [id])
  
  recordedById String  @db.Uuid
  recordedBy  Teacher  @relation(fields: [recordedById], references: [id])
  
  // Attendance Data
  date        DateTime @db.Date
  status      AttendanceStatus
  
  // Excused/Notes
  isExcused   Boolean  @default(false)
  notes       String?
  
  // Multi-tenant
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@unique([studentId, classId, date])
  @@index([tenantId])
  @@index([studentId])
  @@index([date])
  @@index([status])
}

enum AttendanceStatus {
  PRESENT
  ABSENT
  TARDY
  EXCUSED
}
```

---

### 10. Assessments

```prisma
model Assessment {
  id          String   @id @default(uuid()) @db.Uuid
  sourcedId   String   @unique
  
  // Basic Info
  title       String
  description String?
  type        AssessmentType
  
  // Subject
  subject     Subject
  gradeLevel  String?
  
  // Dates
  windowStart DateTime?
  windowEnd   DateTime?
  
  // Multi-tenant
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations
  scores      AssessmentScore[]
  
  @@index([tenantId])
  @@index([type])
  @@index([subject])
}

enum AssessmentType {
  DIAGNOSTIC
  FORMATIVE
  SUMMATIVE
  BENCHMARK
  STANDARDIZED
}

model AssessmentScore {
  id            String   @id @default(uuid()) @db.Uuid
  
  // Student & Assessment
  studentId     String   @db.Uuid
  student       Student  @relation(fields: [studentId], references: [id], onDelete: Cascade)
  
  assessmentId  String   @db.Uuid
  assessment    Assessment @relation(fields: [assessmentId], references: [id], onDelete: Cascade)
  
  // Score Data
  rawScore      Decimal? @db.Decimal(10, 2)
  scaledScore   Decimal? @db.Decimal(10, 2)
  percentile    Int?
  performanceLevel String?
  
  // Dates
  dateCompleted DateTime?
  
  // Multi-tenant
  tenantId      String   @db.Uuid
  tenant        Tenant   @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // Metadata
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  
  @@unique([studentId, assessmentId])
  @@index([tenantId])
  @@index([studentId])
  @@index([assessmentId])
}
```

---

## Performance Indexes

### Critical Indexes

```prisma
// Multi-tenant isolation (ALL tables)
@@index([tenantId])

// User lookup
@@index([email])
@@index([role])

// Student lookup
@@index([studentNumber])
@@index([gradeLevel])
@@index([status])

// Class lookup
@@index([teacherId])
@@index([sessionId])
@@index([subject])

// Enrollment lookup
@@index([studentId, classId])
@@index([status])

// Grade lookup
@@index([studentId, assignmentId])
@@index([scoreStatus])

// Attendance lookup
@@index([studentId, date])
@@index([status])
```

---

## Database Statistics

**Production Metrics** (November 2025):
- **Total Tables**: 78
- **Total Indexes**: 234
- **Database Size**: 125 GB
- **Largest Table**: `Grade` (45M rows)
- **Query Performance**: 95% < 100ms
- **Index Coverage**: 98%

---

## Related Documentation

- [Database Architecture](../architecture/database.md)
- [Prisma Schema](../../api/prisma/schema.prisma)
- [Migrations Guide](../guides/deployment.md#database-migrations)
