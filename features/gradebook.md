# Gradebook System

**Last Updated**: November 16, 2025  
**Status**: ✅ Production-Ready

---

## Overview

SkillCore WebApp provides a comprehensive gradebook system with automatic calculations, weighted grading, standards-based grading, grade posting workflows, and real-time parent/student access.

---

## Features

### Grading Models

**1. Traditional Percentage-Based Grading**:
- Assignments weighted by category (Tests 40%, Homework 30%, etc.)
- Automatic average calculations
- Letter grade conversion (A, B, C, D, F)
- GPA calculation (4.0 scale)

**2. Standards-Based Grading (SBG)**:
- Proficiency levels (1-4 scale)
- Multiple attempts allowed
- Most recent score or highest score
- Standards mastery tracking

**3. Points-Based Grading**:
- Total points earned / Total points possible
- No weighting, straight percentage
- Popular for middle school classes

**4. Pass/Fail Grading**:
- Binary outcomes
- Used for participation, behavior, electives
- No impact on GPA

### Grade Categories

```typescript
interface GradeCategory {
  id: string
  name: string          // "Tests", "Homework", "Projects"
  weight: number        // 0.40 (40%), 0.30 (30%), etc.
  dropLowest?: number   // Drop 2 lowest homework scores
  classId: string
}
```

**Common Weighting Schemes**:
```typescript
// High School
const hsCategories = [
  { name: 'Tests', weight: 0.40 },
  { name: 'Quizzes', weight: 0.25 },
  { name: 'Homework', weight: 0.20 },
  { name: 'Participation', weight: 0.15 }
]

// Middle School
const msCategories = [
  { name: 'Assessments', weight: 0.50 },
  { name: 'Classwork', weight: 0.30 },
  { name: 'Homework', weight: 0.20 }
]

// Elementary
const elemCategories = [
  { name: 'Formative', weight: 0.40 },
  { name: 'Summative', weight: 0.60 }
]
```

### Assignment Types

```typescript
interface Assignment {
  id: string
  classId: string
  categoryId: string
  title: string
  description?: string
  totalPoints: number
  dueDate: Date
  assignedDate: Date
  allowLateSubmission: boolean
  latePenalty?: number      // -10% per day
  gradingType: 'PERCENTAGE' | 'STANDARDS' | 'POINTS' | 'PASS_FAIL'
  rubricId?: string
  standards?: string[]      // Link to educational standards
}
```

### Grading Workflows

**1. Grade Entry**:
```typescript
// Quick entry mode (keyboard navigation)
async function enterGrade(input: GradeEntryInput): Promise<Grade> {
  const grade = await db.grade.create({
    data: {
      studentId: input.studentId,
      assignmentId: input.assignmentId,
      score: input.score,
      maxScore: assignment.totalPoints,
      percentage: (input.score / assignment.totalPoints) * 100,
      enteredBy: currentUser.id,
      enteredAt: new Date()
    }
  })

  // Trigger grade calculation event
  await eventBus.publish(new GradeUpdated({
    gradeId: grade.id,
    studentId: input.studentId,
    assignmentId: input.assignmentId,
    score: input.score
  }))

  return grade
}
```

**2. Bulk Entry**:
```typescript
// Import from CSV
async function importGrades(file: File, assignmentId: string): Promise<void> {
  const rows = await parseCSV(file)
  
  for (const row of rows) {
    await enterGrade({
      studentId: findStudentByNumber(row.studentNumber),
      assignmentId,
      score: parseFloat(row.score)
    })
  }
}
```

**3. Rubric-Based Grading**:
```typescript
interface Rubric {
  id: string
  name: string
  criteria: RubricCriterion[]
}

interface RubricCriterion {
  id: string
  name: string          // "Thesis Statement"
  description: string
  maxPoints: number
  levels: RubricLevel[]
}

interface RubricLevel {
  id: string
  label: string         // "Exemplary", "Proficient", "Developing"
  points: number
  description: string
}

// Grade with rubric
async function gradeWithRubric(
  studentId: string,
  assignmentId: string,
  criteriaScores: { criterionId: string; levelId: string }[]
): Promise<Grade> {
  const totalPoints = criteriaScores.reduce((sum, cs) => {
    const level = findLevel(cs.criterionId, cs.levelId)
    return sum + level.points
  }, 0)

  return enterGrade({
    studentId,
    assignmentId,
    score: totalPoints,
    rubricData: criteriaScores
  })
}
```

### Grade Calculations

**1. Weighted Average**:
```typescript
async function calculateWeightedAverage(
  studentId: string,
  classId: string
): Promise<number> {
  const categories = await getCategories(classId)
  let totalWeightedScore = 0
  let totalWeight = 0

  for (const category of categories) {
    const assignments = await getAssignmentsByCategory(category.id)
    const grades = await getGradesForStudent(studentId, assignments.map(a => a.id))

    // Drop lowest N scores if configured
    const gradesToUse = dropLowestScores(grades, category.dropLowest)

    // Calculate category average
    const categoryAvg = gradesToUse.reduce((sum, g) => sum + g.percentage, 0) / gradesToUse.length

    totalWeightedScore += categoryAvg * category.weight
    totalWeight += category.weight
  }

  return totalWeightedScore / totalWeight
}
```

**2. Standards-Based Calculation**:
```typescript
async function calculateStandardsMastery(
  studentId: string,
  standardId: string
): Promise<number> {
  const attempts = await db.standardAttempt.findMany({
    where: { studentId, standardId },
    orderBy: { attemptedAt: 'desc' }
  })

  // Use most recent attempt (or highest, based on teacher preference)
  const useHighest = await getTeacherPreference('standards_scoring_method')
  
  if (useHighest) {
    return Math.max(...attempts.map(a => a.score))
  } else {
    return attempts[0].score // Most recent
  }
}
```

**3. Trend Analysis**:
```typescript
interface GradeTrend {
  direction: 'IMPROVING' | 'DECLINING' | 'STABLE'
  slope: number
  recentAverage: number
  overallAverage: number
}

async function calculateGradeTrend(
  studentId: string,
  classId: string
): Promise<GradeTrend> {
  const recentGrades = await getGradesLast30Days(studentId, classId)
  const allGrades = await getAllGrades(studentId, classId)

  // Linear regression
  const slope = linearRegression(recentGrades.map((g, i) => [i, g.percentage]))

  return {
    direction: slope > 0.5 ? 'IMPROVING' : slope < -0.5 ? 'DECLINING' : 'STABLE',
    slope,
    recentAverage: average(recentGrades.map(g => g.percentage)),
    overallAverage: average(allGrades.map(g => g.percentage))
  }
}
```

### Grade Posting

**1. Draft Mode**:
```typescript
// Grades entered but not visible to students/parents
await db.assignment.update({
  where: { id: assignmentId },
  data: { status: 'DRAFT' }
})
```

**2. Posted Mode**:
```typescript
// Grades visible to students/parents
async function postGrades(assignmentId: string): Promise<void> {
  await db.assignment.update({
    where: { id: assignmentId },
    data: {
      status: 'POSTED',
      postedAt: new Date(),
      postedBy: currentUser.id
    }
  })

  // Trigger notifications
  await eventBus.publish(new GradesPosted({
    assignmentId,
    postedBy: currentUser.id
  }))
}
```

**3. Locking**:
```typescript
// Prevent further changes after term ends
await db.class.update({
  where: { id: classId },
  data: {
    gradesLockedAt: new Date(),
    gradesLockedBy: currentUser.id
  }
})
```

---

## Event System

### Domain Events

```typescript
export class GradeUpdated extends DomainEvent {
  gradeId: string
  studentId: string
  assignmentId: string
  previousScore?: number
  newScore: number
  enteredBy: string
}

export class GradesPosted extends DomainEvent {
  assignmentId: string
  studentIds: string[]
  postedBy: string
  postedAt: Date
}

export class GradeDeleted extends DomainEvent {
  gradeId: string
  studentId: string
  assignmentId: string
  deletedBy: string
  reason?: string
}

export class CategoryWeightChanged extends DomainEvent {
  categoryId: string
  classId: string
  previousWeight: number
  newWeight: number
  changedBy: string
}

export class AssignmentCreated extends DomainEvent {
  assignmentId: string
  classId: string
  title: string
  dueDate: Date
}

export class GradebookRecalculated extends DomainEvent {
  classId: string
  studentIds: string[]
  reason: string
}
```

### Event Handlers

**1. GradeCalculationHandler** (triggers on GradeUpdated):
```typescript
export class GradeCalculationHandler implements IEventHandler<GradeUpdated> {
  async handle(event: GradeUpdated): Promise<void> {
    // Recalculate student's class average
    const newAverage = await calculateWeightedAverage(
      event.studentId,
      event.classId
    )

    await db.studentClassEnrollment.update({
      where: {
        studentId_classId: {
          studentId: event.studentId,
          classId: event.classId
        }
      },
      data: {
        currentAverage: newAverage,
        letterGrade: convertToLetterGrade(newAverage),
        gpa: convertToGPA(newAverage),
        lastCalculated: new Date()
      }
    })
  }
}
```

**2. ParentNotificationHandler** (triggers on GradesPosted):
```typescript
export class ParentNotificationHandler implements IEventHandler<GradesPosted> {
  async handle(event: GradesPosted): Promise<void> {
    const assignment = await db.assignment.findUnique({
      where: { id: event.assignmentId },
      include: { class: true }
    })

    for (const studentId of event.studentIds) {
      const parent = await getParentOf(studentId)
      const grade = await getGrade(studentId, event.assignmentId)

      if (parent && parent.preferences.gradeNotifications) {
        await notificationQueue.add('email', {
          to: parent.email,
          subject: `Grade Posted: ${assignment.title}`,
          template: 'grade-posted',
          data: {
            studentName: grade.student.name,
            assignmentTitle: assignment.title,
            className: assignment.class.name,
            score: grade.score,
            maxScore: grade.maxScore,
            percentage: grade.percentage,
            letterGrade: convertToLetterGrade(grade.percentage)
          }
        })
      }
    }
  }
}
```

**3. InterventionTriggerHandler** (triggers on GradeCalculationHandler):
```typescript
export class InterventionTriggerHandler implements IEventHandler<GradebookRecalculated> {
  async handle(event: GradebookRecalculated): Promise<void> {
    for (const studentId of event.studentIds) {
      const enrollment = await db.studentClassEnrollment.findUnique({
        where: {
          studentId_classId: {
            studentId,
            classId: event.classId
          }
        }
      })

      // Trigger intervention if grade < 70%
      if (enrollment.currentAverage < 70 && !enrollment.hasActiveIntervention) {
        await createIntervention({
          studentId,
          type: 'academic',
          tier: enrollment.currentAverage < 60 ? 3 : 2,
          reason: `Current average: ${enrollment.currentAverage}%`,
          strategies: [
            'After-school tutoring',
            'Parent conference',
            'Assignment completion plan'
          ]
        })
      }
    }
  }
}
```

**4. TrendAnalysisHandler** (runs daily):
```typescript
export class TrendAnalysisHandler {
  async handle(): Promise<void> {
    const students = await getAllActiveStudents()

    for (const student of students) {
      const classes = await getStudentClasses(student.id)

      for (const cls of classes) {
        const trend = await calculateGradeTrend(student.id, cls.id)

        await db.gradeTrend.upsert({
          where: {
            studentId_classId: {
              studentId: student.id,
              classId: cls.id
            }
          },
          create: {
            studentId: student.id,
            classId: cls.id,
            direction: trend.direction,
            slope: trend.slope,
            calculatedAt: new Date()
          },
          update: {
            direction: trend.direction,
            slope: trend.slope,
            calculatedAt: new Date()
          }
        })

        // Notify if declining
        if (trend.direction === 'DECLINING') {
          await eventBus.publish(new GradeTrendAlert({
            studentId: student.id,
            classId: cls.id,
            trend: 'DECLINING',
            slope: trend.slope
          }))
        }
      }
    }
  }
}
```

---

## GraphQL API

### Queries

```graphql
type Query {
  # Get gradebook for class
  classGradebook(classId: ID!): Gradebook!

  # Get student's grades in class
  studentGrades(studentId: ID!, classId: ID!): StudentGrades!

  # Get grade history for assignment
  assignmentGrades(assignmentId: ID!): [Grade!]!

  # Get grade categories for class
  gradeCategories(classId: ID!): [GradeCategory!]!

  # Get grade trend
  gradeTrend(studentId: ID!, classId: ID!): GradeTrend!

  # Export gradebook
  exportGradebook(classId: ID!, format: ExportFormat!): ExportUrl!
}
```

### Mutations

```graphql
type Mutation {
  # Create assignment
  createAssignment(input: CreateAssignmentInput!): Assignment!

  # Enter grade
  enterGrade(input: EnterGradeInput!): Grade!

  # Bulk enter grades
  bulkEnterGrades(input: BulkEnterGradesInput!): [Grade!]!

  # Update grade
  updateGrade(gradeId: ID!, score: Float!): Grade!

  # Delete grade
  deleteGrade(gradeId: ID!, reason: String): Boolean!

  # Post grades to students/parents
  postGrades(assignmentId: ID!): Assignment!

  # Create grade category
  createCategory(input: CreateCategoryInput!): GradeCategory!

  # Update category weight
  updateCategoryWeight(categoryId: ID!, weight: Float!): GradeCategory!

  # Lock gradebook
  lockGradebook(classId: ID!): Class!
}
```

### Subscriptions

```graphql
type Subscription {
  # Grade updated in real-time
  gradeUpdated(classId: ID!): Grade!

  # Gradebook calculation in progress
  gradebookCalculating(classId: ID!): CalculationProgress!
}
```

---

## UI Features

### Teacher Gradebook

**1. Spreadsheet View**:
- Students in rows, assignments in columns
- Keyboard navigation (Tab, Arrow keys, Enter)
- Inline editing
- Color-coded cells (green >90%, yellow 70-89%, red <70%)
- Frozen student name column

**2. Quick Entry Mode**:
- Focus on one assignment column
- Enter scores sequentially
- Auto-advance to next student
- Keyboard shortcuts (Ctrl+S to save, Esc to cancel)

**3. Filters & Sorting**:
- Filter by: All students, Missing grades, Failing students
- Sort by: Name, Current average, Recent trend
- Search students by name

**4. Grade Distribution Chart**:
- Histogram showing A's, B's, C's, D's, F's
- Box plot showing median, quartiles
- Identifies outliers

### Student Portal

**1. Current Grades View**:
- List of all classes with current averages
- Color-coded performance indicators
- Trend arrows (↗ improving, → stable, ↘ declining)

**2. Assignment List**:
- Upcoming assignments
- Recent grades with feedback
- Missing assignments highlighted

**3. Progress Charts**:
- Line chart showing grade over time
- Category breakdown (pie chart)
- Standards mastery radar chart

### Parent Portal

**1. Multi-Student Dashboard**:
- All children's grades in one view
- Alert indicators for concerning trends
- Summary notifications

**2. Detailed Views**:
- Click into specific class for detailed breakdown
- View individual assignment scores
- Read teacher comments

**3. Communication**:
- "Contact Teacher" button on every grade
- Pre-filled message with context
- Direct link to schedule parent-teacher conference

---

## Analytics

### Class-Level Metrics

```typescript
interface ClassGradeAnalytics {
  averageGrade: number
  medianGrade: number
  distribution: {
    A: number  // Count or percentage
    B: number
    C: number
    D: number
    F: number
  }
  trendDirection: 'IMPROVING' | 'DECLINING' | 'STABLE'
  assignmentCompletion: number  // Percentage
  missingGrades: number
}
```

### District-Level Metrics

```typescript
interface DistrictGradeAnalytics {
  schoolAverages: { schoolId: string; average: number }[]
  gradeDistribution: Record<string, number>
  failureRate: number
  interventionRate: number
  improvementRate: number
}
```

---

## Related Documentation

- [Event Architecture](../architecture/events.md) - Event system
- [Interventions](./interventions.md) - Academic interventions
- [Features Index](./README.md) - All features
