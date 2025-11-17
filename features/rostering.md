# Rostering & Class Management

**Last Updated**: November 17, 2025  
**Status**: ✅ Production-Ready

---

## Overview

The Rostering system manages class creation, student enrollment, section assignment, teacher scheduling, and seat management across the educational platform. Supports complex scheduling scenarios including block schedules, co-teaching, and special program placements.

---

## Features

### Class & Section Management

**Class Structure**:
```typescript
interface Class {
  id: string
  courseId: string          // Links to course catalog
  courseName: string        // "Algebra I", "English 9"
  courseCode: string        // "MATH-101"
  sectionNumber: string     // "01", "02"
  academicSessionId: string // Links to term/semester
  gradeLevel: string        // "9", "10", "11", "12"
  creditHours: number       // 1.0, 0.5
  maxSeats: number
  enrolledCount: number
  teacherId: string
  roomNumber: string
  meetingDays: Day[]        // ["Monday", "Wednesday", "Friday"]
  startTime: string         // "08:00"
  endTime: string           // "08:50"
  periodNumber?: number     // For period-based schedules
}
```

### Course Catalog

**Course Library**:
- Pre-defined courses by subject and grade level
- Course descriptions and prerequisites
- State/district course codes
- Credit requirements
- Standards alignment

**Course Categories**:
- Core subjects (Math, ELA, Science, Social Studies)
- Electives (Arts, Music, Technology)
- Advanced courses (AP, IB, Honors)
- CTE (Career and Technical Education)
- Special programs (ESL, Resource, Life Skills)

---

## Class Creation

**View**: `/admin/rostering/classes/new`

### Create Class Workflow

**Step 1: Select Course**:
- Choose from course catalog
- Or create custom course

**Step 2: Class Details**:
- Section number
- Academic session (Fall 2025, Spring 2026)
- Grade level
- Max seats
- Credit hours

**Step 3: Scheduling**:
- Meeting days
- Time (start and end)
- Period number (if applicable)
- Room assignment

**Step 4: Teacher Assignment**:
- Assign primary teacher
- Co-teachers (if applicable)
- Substitute teachers

**Step 5: Enrollment Options**:
- Manual enrollment
- Auto-enrollment (by criteria)
- Import from SIS

---

## Student Enrollment

**View**: `/admin/rostering/enrollment`

### Enrollment Methods

**1. Manual Enrollment**:
```typescript
async function enrollStudent(
  studentId: string,
  classId: string
): Promise<Enrollment> {
  // Check seat availability
  const cls = await db.class.findUnique({ where: { id: classId } })
  if (cls.enrolledCount >= cls.maxSeats) {
    throw new Error('Class is full')
  }

  // Check schedule conflicts
  const conflicts = await checkScheduleConflicts(studentId, classId)
  if (conflicts.length > 0) {
    throw new Error('Schedule conflict detected')
  }

  // Create enrollment
  const enrollment = await db.enrollment.create({
    data: {
      studentId,
      classId,
      enrolledDate: new Date(),
      status: 'ACTIVE'
    }
  })

  // Update class count
  await db.class.update({
    where: { id: classId },
    data: { enrolledCount: { increment: 1 } }
  })

  // Publish event
  await eventBus.publish(new StudentEnrolled({
    enrollmentId: enrollment.id,
    studentId,
    classId
  }))

  return enrollment
}
```

**2. Bulk Enrollment**:
- Upload CSV with student IDs and class codes
- Validation and conflict detection
- Bulk enrollment processing
- Error reporting

**3. Auto-Enrollment Rules**:
```typescript
// Example: Enroll all 9th graders in required courses
const autoEnrollmentRule = {
  criteria: {
    gradeLevel: '9',
    specialProgram: null // General education
  },
  courses: [
    { courseCode: 'ENG-9', required: true },
    { courseCode: 'MATH-9', required: true },
    { courseCode: 'SCI-9', required: true },
    { courseCode: 'HIST-9', required: true }
  ]
}
```

### Enrollment Status

**Status Types**:
- `ACTIVE`: Currently enrolled
- `DROPPED`: Student dropped the class
- `COMPLETED`: Successfully completed
- `WITHDRAWN`: Withdrawn from school
- `TRANSFERRED`: Moved to different section

### Drop/Add Period

**Schedule Changes**:
- Drop class (with reason)
- Add class (seat availability check)
- Swap sections
- Withdraw from all classes

**Grade Impact**:
- Drop before census date: No grade impact
- Drop after census date: "W" on transcript
- Failing drop: "WF" on transcript

---

## Scheduling

### Schedule Types

**1. Traditional 6-Period Schedule**:
```
Period 1: 08:00 - 08:50
Period 2: 09:00 - 09:50
Period 3: 10:00 - 10:50
Lunch:    11:00 - 11:30
Period 4: 11:40 - 12:30
Period 5: 12:40 - 13:30
Period 6: 13:40 - 14:30
```

**2. Block Schedule (A/B Days)**:
```
A Day:
  Block 1: 08:00 - 09:30 (Period 1)
  Block 2: 09:40 - 11:10 (Period 3)
  Lunch:   11:10 - 11:40
  Block 3: 11:50 - 13:20 (Period 5)
  Block 4: 13:30 - 15:00 (Period 7)

B Day:
  Block 1: 08:00 - 09:30 (Period 2)
  Block 2: 09:40 - 11:10 (Period 4)
  Lunch:   11:10 - 11:40
  Block 3: 11:50 - 13:20 (Period 6)
  Block 4: 13:30 - 15:00 (Period 8)
```

**3. Rotating Schedule**:
- Days 1-6 cycle
- Period order changes daily

**4. Flexible/Modular Schedule**:
- Various block lengths
- Student-specific schedules

### Schedule Conflicts

**Conflict Detection**:
```typescript
interface ScheduleConflict {
  type: 'TIME_OVERLAP' | 'DUPLICATE_COURSE' | 'PREREQUISITE_MISSING' | 'MAX_CREDITS_EXCEEDED'
  message: string
  conflictingClasses: Class[]
}

async function checkScheduleConflicts(
  studentId: string,
  newClassId: string
): Promise<ScheduleConflict[]> {
  const existingClasses = await getStudentClasses(studentId)
  const newClass = await db.class.findUnique({ where: { id: newClassId } })
  
  const conflicts: ScheduleConflict[] = []

  // Check time overlap
  for (const cls of existingClasses) {
    if (hasTimeOverlap(cls, newClass)) {
      conflicts.push({
        type: 'TIME_OVERLAP',
        message: `Time conflict with ${cls.courseName}`,
        conflictingClasses: [cls, newClass]
      })
    }
  }

  // Check duplicate course
  if (existingClasses.some(cls => cls.courseCode === newClass.courseCode)) {
    conflicts.push({
      type: 'DUPLICATE_COURSE',
      message: `Already enrolled in ${newClass.courseName}`,
      conflictingClasses: [newClass]
    })
  }

  // Check prerequisites
  const prerequisites = await getCoursePrerequisites(newClass.courseId)
  const completedCourses = await getCompletedCourses(studentId)
  const missingPrereqs = prerequisites.filter(
    prereq => !completedCourses.includes(prereq)
  )
  
  if (missingPrereqs.length > 0) {
    conflicts.push({
      type: 'PREREQUISITE_MISSING',
      message: `Missing prerequisites: ${missingPrereqs.join(', ')}`,
      conflictingClasses: [newClass]
    })
  }

  return conflicts
}
```

---

## Teacher Assignment

### Primary Teacher

**Assignment**:
- One primary teacher per class
- Teacher certification verification
- Load balancing (max classes per teacher)
- Department alignment

**Teacher Schedules**:
- View teacher's full schedule
- Planning periods
- Duty assignments
- Department meetings

### Co-Teaching

**Co-Teacher Models**:
- **Team Teaching**: Both teachers teach all students
- **Station Teaching**: Teachers rotate stations
- **Parallel Teaching**: Split class, same content
- **Alternative Teaching**: One teacher with small group
- **One Teach, One Assist**: One leads, one supports

**Co-Teacher Assignment**:
```typescript
interface CoTeacher {
  teacherId: string
  role: 'LEAD' | 'SUPPORT' | 'EQUAL'
  percentage: number  // 50%, 100% for grading responsibility
}
```

---

## Seat Management

### Capacity Tracking

**Seat Limits**:
- Set max seats per class
- Real-time enrollment count
- Waitlist management
- Overload approval process

**Seat Analytics**:
```typescript
interface SeatAnalytics {
  classId: string
  maxSeats: number
  enrolledCount: number
  availableSeats: number
  utilizationRate: number
  waitlistCount: number
}
```

### Waitlist Management

**Waitlist Process**:
1. Student requests full class
2. Added to waitlist
3. Notified when seat opens
4. Auto-enroll if configured
5. Waitlist expiration (after X days)

---

## Special Scheduling

### Special Program Placements

**IEP/504 Accommodations**:
- Specific teacher assignments
- Resource room periods
- Push-in/pull-out services
- Modified schedules

**ESL Scheduling**:
- ESL class periods
- Sheltered content classes
- Co-taught classes
- Language support blocks

**Gifted Scheduling**:
- Honors/AP classes
- Seminar periods
- Independent study
- Enrichment activities

### Alternative Schedules

**Homebound Instruction**:
- Virtual class enrollment
- Modified schedule
- Teacher assignments

**Credit Recovery**:
- Online courses
- Summer school
- After-school programs
- Saturday school

---

## Room Assignment

### Room Management

**Room Database**:
```typescript
interface Room {
  id: string
  number: string        // "101", "Gym", "Cafeteria"
  building: string
  capacity: number
  type: 'CLASSROOM' | 'LAB' | 'GYM' | 'LIBRARY' | 'AUDITORIUM'
  equipment: string[]   // ["Smartboard", "Projector", "Science equipment"]
  accessibility: boolean
}
```

**Room Scheduling**:
- View room usage
- Conflict detection
- Room availability search
- Special room requests

---

## Event System

### Domain Events

```typescript
export class StudentEnrolled extends DomainEvent {
  enrollmentId: string
  studentId: string
  classId: string
  enrolledBy: string
}

export class StudentDropped extends DomainEvent {
  enrollmentId: string
  studentId: string
  classId: string
  dropReason: string
  droppedBy: string
}

export class ClassCreated extends DomainEvent {
  classId: string
  courseCode: string
  teacherId: string
  academicSessionId: string
  maxSeats: number
}

export class TeacherAssigned extends DomainEvent {
  classId: string
  teacherId: string
  assignmentType: 'PRIMARY' | 'CO_TEACHER'
}

export class ClassCapacityChanged extends DomainEvent {
  classId: string
  previousCapacity: number
  newCapacity: number
}

export class ScheduleConflictDetected extends DomainEvent {
  studentId: string
  conflictType: string
  conflictingClasses: string[]
}
```

### Event Handlers

**1. EnrollmentNotificationHandler**:
```typescript
export class EnrollmentNotificationHandler implements IEventHandler<StudentEnrolled> {
  async handle(event: StudentEnrolled): Promise<void> {
    const student = await getStudent(event.studentId)
    const cls = await getClass(event.classId)

    // Notify student
    await notificationQueue.add('email', {
      to: student.email,
      subject: `Enrolled in ${cls.courseName}`,
      template: 'class-enrollment',
      data: {
        studentName: student.name,
        className: cls.courseName,
        teacher: cls.teacher.name,
        schedule: formatSchedule(cls)
      }
    })

    // Notify parent
    const parent = await getParentOf(event.studentId)
    if (parent) {
      await notificationQueue.add('email', {
        to: parent.email,
        subject: `${student.name} enrolled in ${cls.courseName}`,
        template: 'parent-enrollment-notification'
      })
    }
  }
}
```

**2. GradebookSetupHandler**:
```typescript
export class GradebookSetupHandler implements IEventHandler<StudentEnrolled> {
  async handle(event: StudentEnrolled): Promise<void> {
    // Create gradebook entry for student
    await db.studentClassEnrollment.create({
      data: {
        studentId: event.studentId,
        classId: event.classId,
        currentAverage: null,
        letterGrade: null
      }
    })

    // Initialize grade records for existing assignments
    const assignments = await getClassAssignments(event.classId)
    for (const assignment of assignments) {
      await db.grade.create({
        data: {
          studentId: event.studentId,
          assignmentId: assignment.id,
          score: null,
          status: 'NOT_SUBMITTED'
        }
      })
    }
  }
}
```

---

## GraphQL API

### Queries

```graphql
type Query {
  # Get all classes
  classes(
    academicSessionId: ID
    teacherId: ID
    gradeLevel: String
  ): [Class!]!

  # Get single class
  class(id: ID!): Class!

  # Get class roster
  classRoster(classId: ID!): [Enrollment!]!

  # Get student schedule
  studentSchedule(studentId: ID!, academicSessionId: ID!): [Class!]!

  # Get teacher schedule
  teacherSchedule(teacherId: ID!, academicSessionId: ID!): [Class!]!

  # Check schedule conflicts
  checkScheduleConflicts(studentId: ID!, classId: ID!): [ScheduleConflict!]!

  # Search available classes
  searchClasses(
    courseCode: String
    gradeLevel: String
    hasSeats: Boolean
  ): [Class!]!
}
```

### Mutations

```graphql
type Mutation {
  # Create class
  createClass(input: CreateClassInput!): Class!

  # Update class
  updateClass(classId: ID!, input: UpdateClassInput!): Class!

  # Delete class
  deleteClass(classId: ID!): Boolean!

  # Enroll student
  enrollStudent(studentId: ID!, classId: ID!): Enrollment!

  # Drop student
  dropStudent(enrollmentId: ID!, reason: String!): Boolean!

  # Swap student sections
  swapSections(studentId: ID!, fromClassId: ID!, toClassId: ID!): Enrollment!

  # Assign teacher
  assignTeacher(classId: ID!, teacherId: ID!, role: TeacherRole!): Class!

  # Change class capacity
  changeCapacity(classId: ID!, newCapacity: Int!): Class!

  # Bulk enrollment
  bulkEnroll(input: BulkEnrollInput!): BulkEnrollResult!
}
```

---

## Reporting

### Enrollment Reports

**Class Rosters**:
- Printable class lists
- Student photos
- Contact information
- Special accommodations

**Master Schedule**:
- All classes by period
- Room assignments
- Teacher assignments
- Enrollment counts

**Load Reports**:
- Teacher loads (classes per teacher)
- Room utilization
- Period balancing
- Department staffing

### Compliance Reports

**State Reporting**:
- Course enrollment by subject
- Student-teacher ratios
- Highly qualified teacher compliance

---

## Related Documentation

- [Demographics](./demographics.md)
- [Gradebook](./gradebook.md)
- [Attendance](./attendance.md)
- [Special Programs](./special-programs.md)
