# Demographics & User Management

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Bounded Context**: Rostering Domain

---

## Overview

The Demographics feature manages **students**, **teachers**, **parents**, and **staff** across districts and schools. It handles **enrollment**, **family relationships**, **demographic data**, and **user profiles** with full **FERPA compliance** and **multi-tenant isolation**.

---

## Key Features

### 1. Student Management

**Student Profile**:
```typescript
interface StudentProfile {
  // Identity
  id: string
  sourcedId: string           // OneRoster ID
  studentNumber: string       // School-assigned ID
  stateId?: string           // State-assigned ID
  
  // Name
  givenName: string
  familyName: string
  middleName?: string
  preferredName?: string
  
  // Contact
  email?: string
  sms?: string               // Mobile phone
  phone?: string             // Home phone
  
  // Demographics
  dateOfBirth: Date
  gender?: string
  ethnicity?: string[]
  primaryLanguage?: string
  secondaryLanguages?: string[]
  
  // Academic
  gradeLevel: string         // K-12
  currentSchoolId: string
  homeroom?: string
  studentType?: string       // 'full-time' | 'part-time' | 'exchange'
  
  // Enrollment status
  enrollmentStatus: 'active' | 'inactive' | 'graduated' | 'transferred' | 'withdrawn'
  enrollmentDate?: Date
  exitDate?: Date
  exitReason?: string
  
  // Special programs
  hasIEP: boolean
  has504: boolean
  isESL: boolean
  isGifted: boolean
  
  // Multi-tenant
  districtId: string
  schoolId: string
  
  // Audit
  createdAt: Date
  updatedAt: Date
  createdBy: string
  updatedBy: string
}
```

**Student Creation Flow**:
```typescript
// 1. Validate student data
const schema = z.object({
  givenName: z.string().min(1).max(50),
  familyName: z.string().min(1).max(50),
  studentNumber: z.string().regex(/^\d{6,10}$/),
  dateOfBirth: z.date().max(new Date()),
  gradeLevel: z.enum(['K', '1', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11', '12']),
  schoolId: z.string().uuid(),
})

// 2. Create student with auto-generated sourcedId
const student = await prisma.student.create({
  data: {
    sourcedId: generateSourcedId('student'),
    districtId: currentUser.districtId,
    schoolId: input.schoolId,
    ...input,
    enrollmentStatus: 'active',
    enrollmentDate: new Date(),
    createdBy: currentUser.id,
  }
})

// 3. Publish domain event
await eventBus.publish('student.created', {
  studentId: student.id,
  districtId: student.districtId,
  schoolId: student.schoolId,
  gradeLevel: student.gradeLevel,
})

// 4. Create initial analytics record
await prisma.studentAnalytics.create({
  data: {
    studentId: student.id,
    districtId: student.districtId,
    currentGPA: 0,
    attendanceRate: 100,
  }
})
```

**Student Update**:
```typescript
// FERPA-compliant update with audit trail
const updateStudent = async (studentId: string, updates: Partial<StudentProfile>) => {
  // 1. Verify access
  await verifyStudentAccess(studentId, currentUser)
  
  // 2. Get current data for audit
  const current = await prisma.student.findUnique({ where: { id: studentId } })
  
  // 3. Update student
  const updated = await prisma.student.update({
    where: { id: studentId },
    data: {
      ...updates,
      updatedBy: currentUser.id,
    }
  })
  
  // 4. Log change in audit table
  await prisma.auditLog.create({
    data: {
      entityType: 'Student',
      entityId: studentId,
      action: 'UPDATE',
      oldData: current,
      newData: updated,
      changedBy: currentUser.id,
      reason: updates.changeReason,
    }
  })
  
  // 5. Publish event
  await eventBus.publish('student.updated', {
    studentId,
    changes: updates,
  })
  
  return updated
}
```

### 2. Family Relationships

**Parent-Student Linking**:
```prisma
model UserRelationship {
  id            String   @id @default(uuid())
  
  // Parent/Guardian
  userId        String
  user          User     @relation("ParentRelationships", fields: [userId], references: [id])
  
  // Student
  relatedUserId String
  relatedUser   User     @relation("StudentRelationships", fields: [relatedUserId], references: [id])
  
  // Relationship type
  relationshipType String // 'parent', 'guardian', 'emergency_contact', 'sibling'
  
  // Permissions
  canViewGrades     Boolean @default(true)
  canViewAttendance Boolean @default(true)
  canReceiveAlerts  Boolean @default(true)
  isPrimary         Boolean @default(false)
  
  // Multi-tenant
  districtId    String
  
  // Audit
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  
  @@unique([userId, relatedUserId, relationshipType])
  @@index([userId])
  @@index([relatedUserId])
}
```

**Link Parent to Student**:
```typescript
const linkParentToStudent = async (parentId: string, studentId: string) => {
  // 1. Verify both exist in same district
  const parent = await prisma.user.findUnique({ where: { id: parentId } })
  const student = await prisma.user.findUnique({ where: { id: studentId } })
  
  if (parent.districtId !== student.districtId) {
    throw new Error('Parent and student must be in same district')
  }
  
  // 2. Create relationship
  const relationship = await prisma.userRelationship.create({
    data: {
      userId: parentId,
      relatedUserId: studentId,
      relationshipType: 'parent',
      districtId: parent.districtId,
      canViewGrades: true,
      canViewAttendance: true,
      canReceiveAlerts: true,
      isPrimary: true,
    }
  })
  
  // 3. Publish event
  await eventBus.publish('parent.linked', {
    parentId,
    studentId,
    districtId: parent.districtId,
  })
  
  return relationship
}
```

**Query Students by Parent**:
```typescript
const getStudentsByParent = async (parentId: string) => {
  const students = await prisma.user.findMany({
    where: {
      relatedRelationships: {
        some: {
          userId: parentId,
          relationshipType: { in: ['parent', 'guardian'] },
        }
      },
      role: 'STUDENT',
      deletedAt: null,
    },
    include: {
      currentEnrollments: {
        where: { status: 'active' },
        include: {
          class: {
            include: {
              course: true,
              primaryTeacher: true,
            }
          }
        }
      },
      grades: {
        where: { 
          scoreStatus: 'fully_graded',
          deletedAt: null,
        },
        orderBy: { gradedAt: 'desc' },
        take: 10,
      },
      attendance: {
        where: {
          date: { gte: startOfWeek(new Date()) },
        },
        orderBy: { date: 'desc' },
      }
    }
  })
  
  return students
}
```

### 3. Teacher Management

**Teacher Profile**:
```typescript
interface TeacherProfile {
  // Identity
  id: string
  sourcedId: string
  employeeNumber: string
  
  // Name
  givenName: string
  familyName: string
  title?: string             // Mr., Ms., Dr.
  
  // Contact
  email: string
  phone?: string
  officeLocation?: string
  
  // Professional
  certifications?: string[]
  yearsExperience?: number
  subjects?: string[]        // Math, Science, ELA
  gradeLevels?: string[]     // K-5, 6-8, 9-12
  
  // Assignment
  primarySchoolId: string
  additionalSchools?: string[]
  department?: string
  
  // Teaching
  currentClasses: Class[]
  maxStudents?: number
  
  // Multi-tenant
  districtId: string
  
  // Status
  employmentStatus: 'active' | 'leave' | 'terminated'
  hireDate?: Date
  terminationDate?: Date
}
```

**Assign Teacher to Class**:
```typescript
const assignTeacherToClass = async (teacherId: string, classId: string, role: 'primary' | 'co-teacher') => {
  // 1. Verify teacher and class in same district
  const teacher = await prisma.user.findUnique({ where: { id: teacherId } })
  const classData = await prisma.class.findUnique({ where: { id: classId } })
  
  if (teacher.districtId !== classData.districtId) {
    throw new Error('Teacher and class must be in same district')
  }
  
  // 2. Assign teacher
  if (role === 'primary') {
    await prisma.class.update({
      where: { id: classId },
      data: { primaryTeacherId: teacherId },
    })
  } else {
    await prisma.class.update({
      where: { id: classId },
      data: {
        coTeachers: {
          connect: { id: teacherId },
        }
      }
    })
  }
  
  // 3. Publish event
  await eventBus.publish('teacher.assigned', {
    teacherId,
    classId,
    role,
    districtId: teacher.districtId,
  })
}
```

### 4. Enrollment Management

**Enrollment Model**:
```prisma
model Enrollment {
  id                String      @id @default(uuid())
  
  // OneRoster fields
  sourcedId         String      @unique
  status            EntityStatus @default(active)
  
  // Relationships
  classId           String
  class             Class       @relation(fields: [classId], references: [id])
  studentId         String
  student           Student     @relation(fields: [studentId], references: [id])
  schoolId          String
  school            Org         @relation(fields: [schoolId], references: [sourcedId])
  
  // Enrollment period
  beginDate         DateTime?
  endDate           DateTime?
  
  // Role in class
  role              EnrollmentRole @default(student) // student, aide, proctor
  
  // Academic session
  academicSessionId String
  academicSession   AcademicSession @relation(fields: [academicSessionId], references: [id])
  
  // Multi-tenant
  districtId        String
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt
  
  @@unique([classId, studentId, academicSessionId])
  @@index([studentId])
  @@index([classId])
  @@index([districtId])
}
```

**Enroll Student in Class**:
```typescript
const enrollStudent = async (studentId: string, classId: string) => {
  // 1. Verify student and class in same district
  const student = await prisma.student.findUnique({ where: { id: studentId } })
  const classData = await prisma.class.findUnique({ 
    where: { id: classId },
    include: { academicSession: true },
  })
  
  // 2. Check enrollment capacity
  const currentEnrollment = await prisma.enrollment.count({
    where: { 
      classId,
      status: 'active',
      deletedAt: null,
    }
  })
  
  if (currentEnrollment >= classData.maxStudents) {
    throw new Error('Class is at maximum capacity')
  }
  
  // 3. Create enrollment
  const enrollment = await prisma.enrollment.create({
    data: {
      sourcedId: generateSourcedId('enrollment'),
      studentId,
      classId,
      schoolId: classData.schoolId,
      academicSessionId: classData.academicSessionId,
      districtId: student.districtId,
      status: 'active',
      beginDate: new Date(),
      role: 'student',
    }
  })
  
  // 4. Publish event
  await eventBus.publish('student.enrolled', {
    enrollmentId: enrollment.id,
    studentId,
    classId,
    districtId: student.districtId,
  })
  
  return enrollment
}
```

**Bulk Enrollment**:
```typescript
const bulkEnrollStudents = async (studentIds: string[], classId: string) => {
  const enrollments = await Promise.all(
    studentIds.map(studentId => enrollStudent(studentId, classId))
  )
  
  // Publish bulk event
  await eventBus.publish('students.bulk_enrolled', {
    count: enrollments.length,
    classId,
    studentIds,
  })
  
  return enrollments
}
```

### 5. Demographic Reporting

**Demographic Breakdown**:
```typescript
const getDemographicBreakdown = async (districtId: string, schoolId?: string) => {
  const where = {
    districtId,
    ...(schoolId && { schoolId }),
    deletedAt: null,
    enrollmentStatus: 'active',
  }
  
  // Gender breakdown
  const genderBreakdown = await prisma.student.groupBy({
    by: ['gender'],
    where,
    _count: true,
  })
  
  // Ethnicity breakdown
  const ethnicityBreakdown = await prisma.student.groupBy({
    by: ['ethnicity'],
    where,
    _count: true,
  })
  
  // Grade level breakdown
  const gradeLevelBreakdown = await prisma.student.groupBy({
    by: ['gradeLevel'],
    where,
    _count: true,
  })
  
  // Special programs
  const specialPrograms = {
    iep: await prisma.student.count({ where: { ...where, hasIEP: true } }),
    section504: await prisma.student.count({ where: { ...where, has504: true } }),
    esl: await prisma.student.count({ where: { ...where, isESL: true } }),
    gifted: await prisma.student.count({ where: { ...where, isGifted: true } }),
  }
  
  return {
    gender: genderBreakdown,
    ethnicity: ethnicityBreakdown,
    gradeLevel: gradeLevelBreakdown,
    specialPrograms,
    total: await prisma.student.count({ where }),
  }
}
```

---

## GraphQL API

### Queries

**Get Student**:
```graphql
query GetStudent($id: ID!) {
  student(id: $id) {
    id
    sourcedId
    givenName
    familyName
    preferredName
    email
    gradeLevel
    currentSchool {
      id
      name
    }
    enrollments(filter: { status: ACTIVE }) {
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
    hasIEP
    has504
    isESL
  }
}
```

**Get Students by School**:
```graphql
query GetStudentsBySchool($schoolId: ID!, $gradeLevel: String) {
  students(
    filter: {
      schoolId: $schoolId
      gradeLevel: $gradeLevel
      enrollmentStatus: ACTIVE
    }
    orderBy: { field: FAMILY_NAME, direction: ASC }
  ) {
    edges {
      node {
        id
        givenName
        familyName
        gradeLevel
        homeroom
        currentGPA
        attendanceRate
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
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
      relationshipType
      isPrimary
    }
    siblings {
      id
      givenName
      familyName
      gradeLevel
      currentSchool {
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
      givenName
      familyName
      studentNumber
      gradeLevel
    }
    errors {
      field
      message
    }
  }
}
```

**Update Student**:
```graphql
mutation UpdateStudent($id: ID!, $input: UpdateStudentInput!) {
  updateStudent(id: $id, input: $input) {
    student {
      id
      givenName
      familyName
      email
      gradeLevel
      updatedAt
    }
    errors {
      field
      message
    }
  }
}
```

**Link Parent**:
```graphql
mutation LinkParent($parentId: ID!, $studentId: ID!, $relationshipType: RelationshipType!) {
  linkParent(
    parentId: $parentId
    studentId: $studentId
    relationshipType: $relationshipType
  ) {
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
    }
    errors {
      field
      message
    }
  }
}
```

---

## UI Components

### Student Profile Card

```typescript
import { Card } from '@/components/ui/card'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Text, Flex, Box } from '@radix-ui/themes'
import { User, Mail, Phone } from 'lucide-react'

interface StudentProfileCardProps {
  student: Student
  onEdit?: () => void
}

export function StudentProfileCard({ student, onEdit }: StudentProfileCardProps) {
  return (
    <Card className="p-6">
      <Flex direction="column" gap="4">
        {/* Header */}
        <Flex align="center" justify="between">
          <Flex align="center" gap="3">
            <div className="h-12 w-12 rounded-full bg-blue-100 flex items-center justify-center">
              <User className="h-6 w-6 text-blue-600" />
            </div>
            <Box>
              <Text size="4" weight="bold">
                {student.preferredName || student.givenName} {student.familyName}
              </Text>
              <Text size="2" color="gray">
                Student #{student.studentNumber}
              </Text>
            </Box>
          </Flex>
          {onEdit && (
            <Button variant="outline" onClick={onEdit}>
              Edit Profile
            </Button>
          )}
        </Flex>

        {/* Demographics */}
        <Box>
          <Text size="2" weight="medium" className="mb-2">Demographics</Text>
          <Flex gap="2" wrap="wrap">
            <Badge variant="secondary">Grade {student.gradeLevel}</Badge>
            <Badge variant="secondary">{student.gender}</Badge>
            {student.hasIEP && <Badge color="blue">IEP</Badge>}
            {student.has504 && <Badge color="purple">504</Badge>}
            {student.isESL && <Badge color="green">ESL</Badge>}
            {student.isGifted && <Badge color="orange">Gifted</Badge>}
          </Flex>
        </Box>

        {/* Contact */}
        {(student.email || student.phone) && (
          <Box>
            <Text size="2" weight="medium" className="mb-2">Contact</Text>
            <Flex direction="column" gap="1">
              {student.email && (
                <Flex align="center" gap="2">
                  <Mail className="h-4 w-4 text-gray-400" />
                  <Text size="2">{student.email}</Text>
                </Flex>
              )}
              {student.phone && (
                <Flex align="center" gap="2">
                  <Phone className="h-4 w-4 text-gray-400" />
                  <Text size="2">{student.phone}</Text>
                </Flex>
              )}
            </Flex>
          </Box>
        )}

        {/* Quick Stats */}
        <Flex gap="4">
          <Box>
            <Text size="1" color="gray">Current GPA</Text>
            <Text size="3" weight="bold" className="font-mono">
              {student.currentGPA?.toFixed(2) || 'N/A'}
            </Text>
          </Box>
          <Box>
            <Text size="1" color="gray">Attendance</Text>
            <Text size="3" weight="bold" className="font-mono">
              {student.attendanceRate?.toFixed(1) || 'N/A'}%
            </Text>
          </Box>
        </Flex>
      </Flex>
    </Card>
  )
}
```

### Family Relationships List

```typescript
import { Card } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Text, Flex, Box } from '@radix-ui/themes'
import { UserPlus, Mail, Phone, Star } from 'lucide-react'

interface FamilyRelationshipsListProps {
  relationships: UserRelationship[]
  onAddParent?: () => void
}

export function FamilyRelationshipsList({ relationships, onAddParent }: FamilyRelationshipsListProps) {
  return (
    <Card className="p-6">
      <Flex direction="column" gap="4">
        <Flex align="center" justify="between">
          <Text size="4" weight="bold">Family Contacts</Text>
          {onAddParent && (
            <Button variant="outline" onClick={onAddParent}>
              <UserPlus className="h-4 w-4 mr-2" />
              Add Parent
            </Button>
          )}
        </Flex>

        {relationships.map(rel => (
          <Card key={rel.id} className="p-4 bg-gray-50">
            <Flex align="center" justify="between">
              <Flex direction="column" gap="1">
                <Flex align="center" gap="2">
                  <Text size="3" weight="medium">
                    {rel.user.givenName} {rel.user.familyName}
                  </Text>
                  {rel.isPrimary && (
                    <Star className="h-4 w-4 text-yellow-500 fill-yellow-500" />
                  )}
                </Flex>
                <Text size="2" color="gray" className="capitalize">
                  {rel.relationshipType.replace('_', ' ')}
                </Text>
                <Flex gap="3" className="mt-1">
                  {rel.user.email && (
                    <Flex align="center" gap="1">
                      <Mail className="h-3 w-3 text-gray-400" />
                      <Text size="1">{rel.user.email}</Text>
                    </Flex>
                  )}
                  {rel.user.phone && (
                    <Flex align="center" gap="1">
                      <Phone className="h-3 w-3 text-gray-400" />
                      <Text size="1">{rel.user.phone}</Text>
                    </Flex>
                  )}
                </Flex>
              </Flex>
              <Flex direction="column" gap="1" align="end">
                <Text size="1" color={rel.canViewGrades ? 'green' : 'gray'}>
                  {rel.canViewGrades ? '✓' : '✗'} View Grades
                </Text>
                <Text size="1" color={rel.canViewAttendance ? 'green' : 'gray'}>
                  {rel.canViewAttendance ? '✓' : '✗'} View Attendance
                </Text>
                <Text size="1" color={rel.canReceiveAlerts ? 'green' : 'gray'}>
                  {rel.canReceiveAlerts ? '✓' : '✗'} Receive Alerts
                </Text>
              </Flex>
            </Flex>
          </Card>
        ))}
      </Flex>
    </Card>
  )
}
```

---

## Domain Events

### Published Events

```typescript
// Student lifecycle
'student.created'           // New student added
'student.updated'           // Student info changed
'student.deleted'           // Student soft-deleted
'student.enrolled'          // Student enrolled in class
'student.withdrawn'         // Student withdrawn from class
'student.transferred'       // Student transferred to new school
'student.graduated'         // Student graduated

// Family relationships
'parent.linked'             // Parent linked to student
'parent.unlinked'           // Parent unlinked from student
'guardian.assigned'         // Guardian assigned

// Teacher assignments
'teacher.assigned'          // Teacher assigned to class
'teacher.removed'           // Teacher removed from class

// Enrollment
'enrollment.created'        // Student enrolled
'enrollment.updated'        // Enrollment changed
'enrollment.ended'          // Enrollment ended
```

---

## Security & Compliance

### FERPA Compliance

**Data Access Control**:
```typescript
// Only authorized users can access student data
const verifyStudentAccess = async (studentId: string, userId: string, userRole: UserRole) => {
  // System admins can access all students
  if (userRole === 'SYSTEM_ADMIN') return true
  
  // District admins can access students in their district
  if (userRole === 'DISTRICT_ADMIN') {
    const student = await prisma.student.findUnique({ where: { id: studentId } })
    const user = await prisma.user.findUnique({ where: { id: userId } })
    return student.districtId === user.districtId
  }
  
  // Teachers can access students in their classes
  if (userRole === 'TEACHER') {
    const enrollment = await prisma.enrollment.findFirst({
      where: {
        studentId,
        class: {
          OR: [
            { primaryTeacherId: userId },
            { coTeachers: { some: { id: userId } } },
          ],
        },
        status: 'active',
      },
    })
    return !!enrollment
  }
  
  // Parents can access their own children
  if (userRole === 'PARENT') {
    const relationship = await prisma.userRelationship.findFirst({
      where: {
        userId,
        relatedUserId: studentId,
        relationshipType: { in: ['parent', 'guardian'] },
      },
    })
    return !!relationship
  }
  
  // Students can access their own data
  if (userRole === 'STUDENT') {
    return studentId === userId
  }
  
  return false
}
```

---

## Performance Optimization

### Indexing Strategy

```prisma
// Student indexes
@@index([districtId])
@@index([schoolId])
@@index([studentNumber])
@@index([gradeLevel])
@@index([enrollmentStatus])
@@index([deletedAt])

// Enrollment indexes
@@index([studentId, classId])
@@index([classId])
@@index([academicSessionId])
@@index([status])

// Relationship indexes
@@index([userId])
@@index([relatedUserId])
@@index([relationshipType])
```

---

## Related Documentation

- [Rostering API](../api/graphql.md#rostering) - GraphQL API for demographics
- [Authentication](../architecture/security.md) - User authentication
- [Multi-Tenancy](../architecture/database.md#multi-tenancy) - Tenant isolation

---

**Student Statistics** (as of November 2025):
- **Total Students**: 1.2M
- **Active Enrollments**: 8.5M
- **Parent Relationships**: 2.8M
- **Query Performance (P95)**: 85ms
