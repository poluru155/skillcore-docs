# Adding UI Components

**Last Updated**: November 17, 2025  
**Difficulty**: Beginner-Intermediate  
**Time**: 30-60 minutes

---

## Overview

This guide shows how to create **educational UI components** for SkillCore using our design system: **Radix UI Themes** + **Tailwind CSS** + **shadcn/ui components**.

---

## Component Hierarchy (MANDATORY)

**CRITICAL**: Always follow this priority order:

1. **FIRST**: `/src/components/ui/` folder components (Button, Input, Card, Dialog, etc.)
2. **SECOND**: Radix UI Themes components (Text, Badge, Box, Grid, Flex, etc.)
3. **THIRD**: Educational components (StudentCard, GradeEntry, etc.)
4. **LAST RESORT**: Custom HTML elements (only when neither UI nor Radix can achieve result)

---

## Step-by-Step Guide

### Step 1: Check Existing UI Components

**ALWAYS check** `/src/components/ui/` **FIRST**:

```bash
ls src/components/ui/

# Available components:
# button.tsx, input.tsx, card.tsx, dialog.tsx, badge.tsx
# form.tsx, label.tsx, select.tsx, textarea.tsx, checkbox.tsx
# table.tsx, tabs.tsx, dropdown-menu.tsx, popover.tsx
# toast.tsx, alert.tsx, skeleton.tsx, progress.tsx
```

**Example - Use existing UI components**:
```typescript
✅ CORRECT:
import { Button } from '@/components/ui/button'
import { Card } from '@/components/ui/card'
import { Input } from '@/components/ui/input'

<Card className="p-6">
  <Input placeholder="Student name" />
  <Button variant="primary">Save</Button>
</Card>

❌ WRONG: Creating custom when UI component exists
<div className="rounded-lg border bg-white p-6"> {/* Use Card instead */}
<button className="bg-blue-500 text-white px-4 py-2"> {/* Use Button instead */}
```

---

### Step 2: Use Radix UI for Layout & Typography

**For elements not in `/src/components/ui/`, use Radix UI**:

```typescript
import { Text, Badge, Box, Grid, Flex, Callout } from '@radix-ui/themes'

✅ CORRECT: Radix UI for structure
<Box className="space-y-4">
  <Flex align="center" justify="between">
    <Text size="4" weight="bold">Student Profile</Text>
    <Badge color="green">Active</Badge>
  </Flex>
  
  <Grid columns="3" gap="4">
    <Card>...</Card>
    <Card>...</Card>
    <Card>...</Card>
  </Grid>
</Box>

❌ WRONG: Custom divs instead of Radix
<div className="space-y-4"> {/* Use Box */}
  <div className="flex items-center justify-between"> {/* Use Flex */}
    <span className="text-xl font-bold">Student Profile</span> {/* Use Text */}
  </div>
</div>
```

---

### Step 3: Create Educational Component

**Location**: `src/components/educational/{component-name}.tsx`

**Example**: Student Performance Card

```typescript
import { Card } from '@/components/ui/card'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Text, Flex, Box } from '@radix-ui/themes'
import { TrendingUp, TrendingDown, Minus } from 'lucide-react'

interface StudentPerformanceCardProps {
  student: {
    id: string
    givenName: string
    familyName: string
    currentGPA: number
    previousGPA: number
    trend: 'improving' | 'declining' | 'stable'
    attendanceRate: number
    gradeLevel: string
  }
  onViewDetails?: () => void
}

export function StudentPerformanceCard({
  student,
  onViewDetails,
}: StudentPerformanceCardProps) {
  const getTrendIcon = () => {
    switch (student.trend) {
      case 'improving':
        return <TrendingUp className="h-4 w-4 text-green-600" />
      case 'declining':
        return <TrendingDown className="h-4 w-4 text-red-600" />
      case 'stable':
        return <Minus className="h-4 w-4 text-gray-400" />
    }
  }
  
  const getTrendColor = () => {
    switch (student.trend) {
      case 'improving': return 'green'
      case 'declining': return 'red'
      case 'stable': return 'gray'
    }
  }
  
  return (
    <Card className="p-6 hover:shadow-md transition-shadow">
      <Flex direction="column" gap="4">
        {/* Header */}
        <Flex align="center" justify="between">
          <Box>
            <Text size="4" weight="bold">
              {student.givenName} {student.familyName}
            </Text>
            <Text size="2" color="gray">
              Grade {student.gradeLevel}
            </Text>
          </Box>
          <Badge color={getTrendColor()}>
            <Flex align="center" gap="1">
              {getTrendIcon()}
              <Text size="1">{student.trend}</Text>
            </Flex>
          </Badge>
        </Flex>

        {/* Metrics */}
        <Grid columns="2" gap="3">
          {/* GPA */}
          <Box>
            <Text size="1" color="gray">Current GPA</Text>
            <Flex align="baseline" gap="2">
              <Text size="5" weight="bold" className="font-mono">
                {student.currentGPA.toFixed(2)}
              </Text>
              {student.previousGPA && (
                <Text size="2" color="gray">
                  ({student.currentGPA > student.previousGPA ? '+' : ''}
                  {(student.currentGPA - student.previousGPA).toFixed(2)})
                </Text>
              )}
            </Flex>
          </Box>

          {/* Attendance */}
          <Box>
            <Text size="1" color="gray">Attendance Rate</Text>
            <Text size="5" weight="bold" className="font-mono">
              {student.attendanceRate.toFixed(1)}%
            </Text>
          </Box>
        </Grid>

        {/* Actions */}
        {onViewDetails && (
          <Button variant="outline" onClick={onViewDetails} className="w-full">
            View Details
          </Button>
        )}
      </Flex>
    </Card>
  )
}
```

---

### Step 4: Create Form Component

**Example**: Grade Entry Form

```typescript
import { useState } from 'react'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Label } from '@/components/ui/label'
import { Textarea } from '@/components/ui/textarea'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import { Text, Box } from '@radix-ui/themes'

// Validation schema
const gradeSchema = z.object({
  score: z.number().min(0).max(100),
  scoreStatus: z.enum(['not_submitted', 'partially_graded', 'fully_graded', 'exempt']),
  comment: z.string().optional(),
})

type GradeFormData = z.infer<typeof gradeSchema>

interface GradeEntryFormProps {
  studentName: string
  assignmentName: string
  maxPoints: number
  onSubmit: (data: GradeFormData) => Promise<void>
  onCancel?: () => void
}

export function GradeEntryForm({
  studentName,
  assignmentName,
  maxPoints,
  onSubmit,
  onCancel,
}: GradeEntryFormProps) {
  const [isSubmitting, setIsSubmitting] = useState(false)
  
  const {
    register,
    handleSubmit,
    formState: { errors },
    setValue,
    watch,
  } = useForm<GradeFormData>({
    resolver: zodResolver(gradeSchema),
    defaultValues: {
      scoreStatus: 'fully_graded',
    },
  })
  
  const score = watch('score')
  const scorePercent = score ? ((score / maxPoints) * 100).toFixed(1) : '0.0'
  
  const handleFormSubmit = async (data: GradeFormData) => {
    setIsSubmitting(true)
    try {
      await onSubmit(data)
    } finally {
      setIsSubmitting(false)
    }
  }
  
  return (
    <form onSubmit={handleSubmit(handleFormSubmit)} className="space-y-6">
      {/* Student & Assignment Info */}
      <Box>
        <Text size="4" weight="bold">{studentName}</Text>
        <Text size="2" color="gray">{assignmentName}</Text>
        <Text size="2" color="gray">Maximum: {maxPoints} points</Text>
      </Box>

      {/* Score Input */}
      <div className="space-y-2">
        <Label htmlFor="score">
          Score (0-{maxPoints})
          <span className="text-red-500" aria-label="required">*</span>
        </Label>
        <Input
          id="score"
          type="number"
          min="0"
          max={maxPoints}
          step="0.1"
          {...register('score', { valueAsNumber: true })}
          aria-describedby="score-help score-error"
        />
        {score !== undefined && (
          <Text id="score-help" size="2" color="gray">
            {scorePercent}% ({getLetterGrade(Number(scorePercent))})
          </Text>
        )}
        {errors.score && (
          <Text id="score-error" size="2" color="red">
            {errors.score.message}
          </Text>
        )}
      </div>

      {/* Status Select */}
      <div className="space-y-2">
        <Label htmlFor="scoreStatus">Status</Label>
        <Select
          onValueChange={(value) => setValue('scoreStatus', value as any)}
          defaultValue="fully_graded"
        >
          <SelectTrigger id="scoreStatus">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="fully_graded">Fully Graded</SelectItem>
            <SelectItem value="partially_graded">Partially Graded</SelectItem>
            <SelectItem value="not_submitted">Not Submitted</SelectItem>
            <SelectItem value="exempt">Exempt</SelectItem>
          </SelectContent>
        </Select>
      </div>

      {/* Comment Textarea */}
      <div className="space-y-2">
        <Label htmlFor="comment">Comment (optional)</Label>
        <Textarea
          id="comment"
          placeholder="Add feedback for student..."
          rows={3}
          {...register('comment')}
        />
      </div>

      {/* Actions */}
      <div className="flex gap-2 justify-end">
        {onCancel && (
          <Button type="button" variant="outline" onClick={onCancel}>
            Cancel
          </Button>
        )}
        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Saving...' : 'Save Grade'}
        </Button>
      </div>
    </form>
  )
}

function getLetterGrade(percent: number): string {
  if (percent >= 90) return 'A'
  if (percent >= 80) return 'B'
  if (percent >= 70) return 'C'
  if (percent >= 60) return 'D'
  return 'F'
}
```

---

### Step 5: Create Data Display Component

**Example**: Gradebook Table

```typescript
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Text } from '@radix-ui/themes'
import { Edit } from 'lucide-react'

interface GradebookTableProps {
  students: Array<{
    id: string
    name: string
    grades: Record<string, number | null>
    average: number
  }>
  assignments: Array<{
    id: string
    name: string
    maxPoints: number
  }>
  onEditGrade: (studentId: string, assignmentId: string) => void
}

export function GradebookTable({
  students,
  assignments,
  onEditGrade,
}: GradebookTableProps) {
  return (
    <div className="rounded-md border">
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead className="sticky left-0 bg-background z-10">
              Student
            </TableHead>
            {assignments.map(assignment => (
              <TableHead key={assignment.id} className="text-center">
                <div>
                  <Text size="2" weight="medium">{assignment.name}</Text>
                  <Text size="1" color="gray">/{assignment.maxPoints}</Text>
                </div>
              </TableHead>
            ))}
            <TableHead className="text-center">Average</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {students.map(student => (
            <TableRow key={student.id}>
              {/* Student Name (Sticky) */}
              <TableCell className="sticky left-0 bg-background z-10 font-medium">
                {student.name}
              </TableCell>

              {/* Grades */}
              {assignments.map(assignment => {
                const grade = student.grades[assignment.id]
                const percent = grade !== null
                  ? (grade / assignment.maxPoints) * 100
                  : null
                
                return (
                  <TableCell
                    key={assignment.id}
                    className="text-center cursor-pointer hover:bg-gray-50"
                    onClick={() => onEditGrade(student.id, assignment.id)}
                  >
                    {grade !== null ? (
                      <div className="flex items-center justify-center gap-2">
                        <Text size="2" className="font-mono">{grade}</Text>
                        <Badge
                          variant="secondary"
                          className={getGradeColor(percent!)}
                        >
                          {getLetterGrade(percent!)}
                        </Badge>
                      </div>
                    ) : (
                      <Button variant="ghost" size="sm">
                        <Edit className="h-3 w-3" />
                      </Button>
                    )}
                  </TableCell>
                )
              })}

              {/* Average */}
              <TableCell className="text-center font-bold">
                <Badge className={getGradeColor(student.average)}>
                  {student.average.toFixed(1)}%
                </Badge>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </div>
  )
}

function getLetterGrade(percent: number): string {
  if (percent >= 90) return 'A'
  if (percent >= 80) return 'B'
  if (percent >= 70) return 'C'
  if (percent >= 60) return 'D'
  return 'F'
}

function getGradeColor(percent: number): string {
  if (percent >= 90) return 'bg-green-100 text-green-800'
  if (percent >= 80) return 'bg-blue-100 text-blue-800'
  if (percent >= 70) return 'bg-yellow-100 text-yellow-800'
  if (percent >= 60) return 'bg-orange-100 text-orange-800'
  return 'bg-red-100 text-red-800'
}
```

---

### Step 6: Add TypeScript Types

**Location**: `src/types/educational.ts`

```typescript
export interface Student {
  id: string
  sourcedId: string
  givenName: string
  familyName: string
  preferredName?: string
  studentNumber: string
  email?: string
  gradeLevel: string
  currentGPA?: number
  attendanceRate?: number
  hasIEP: boolean
  has504: boolean
  isESL: boolean
}

export interface Grade {
  id: string
  score: number
  scorePercent: number
  letterGrade: string
  scoreStatus: 'not_submitted' | 'partially_graded' | 'fully_graded' | 'exempt'
  comment?: string
  gradedAt?: Date
  postedAt?: Date
}

export type GradeTrend = 'improving' | 'declining' | 'stable'
```

---

### Step 7: Write Component Tests

**Location**: `src/components/educational/__tests__/student-performance-card.test.tsx`

```typescript
import { describe, it, expect, vi } from 'vitest'
import { render, screen, fireEvent } from '@testing-library/react'
import { StudentPerformanceCard } from '../student-performance-card'

describe('StudentPerformanceCard', () => {
  const mockStudent = {
    id: 'student-123',
    givenName: 'John',
    familyName: 'Doe',
    currentGPA: 3.5,
    previousGPA: 3.2,
    trend: 'improving' as const,
    attendanceRate: 95.5,
    gradeLevel: '10',
  }
  
  it('should render student information', () => {
    render(<StudentPerformanceCard student={mockStudent} />)
    
    expect(screen.getByText('John Doe')).toBeInTheDocument()
    expect(screen.getByText('Grade 10')).toBeInTheDocument()
    expect(screen.getByText('3.50')).toBeInTheDocument()
    expect(screen.getByText('95.5%')).toBeInTheDocument()
  })
  
  it('should show GPA change', () => {
    render(<StudentPerformanceCard student={mockStudent} />)
    
    expect(screen.getByText(/\+0.30/)).toBeInTheDocument()
  })
  
  it('should call onViewDetails when button clicked', () => {
    const onViewDetails = vi.fn()
    
    render(
      <StudentPerformanceCard
        student={mockStudent}
        onViewDetails={onViewDetails}
      />
    )
    
    fireEvent.click(screen.getByRole('button', { name: /view details/i }))
    
    expect(onViewDetails).toHaveBeenCalledTimes(1)
  })
})
```

---

## Component Best Practices

### 1. Props Interface

```typescript
✅ Good: Explicit interface
interface StudentCardProps {
  student: Student
  showGrades?: boolean
  onSelect?: (student: Student) => void
}

❌ Bad: No interface
function StudentCard({ student, showGrades, onSelect }) {
  // Missing type safety
}
```

### 2. Educational Context

```typescript
✅ Good: Educational naming
<GradeEntryForm />
<AttendanceMarker />
<IEPGoalTracker />

❌ Bad: Generic naming
<Form />
<Marker />
<Tracker />
```

### 3. Accessibility

```typescript
✅ Good: ARIA labels and semantic HTML
<Label htmlFor="grade-input">
  Grade (0-100)
  <span className="text-red-500" aria-label="required">*</span>
</Label>
<Input
  id="grade-input"
  type="number"
  aria-describedby="grade-help"
  {...register('grade')}
/>

❌ Bad: Missing accessibility
<div>Grade</div>
<input type="number" />
```

### 4. Responsive Design

```typescript
✅ Good: Mobile-first
<div className="
  grid
  grid-cols-1
  md:grid-cols-2
  lg:grid-cols-3
  gap-4
">

❌ Bad: Desktop-only
<div className="grid grid-cols-3 gap-4">
```

---

## Component Checklist

- [ ] Uses UI components from `/src/components/ui/` first
- [ ] Uses Radix UI Themes for layout/typography
- [ ] Has TypeScript interface for props
- [ ] Includes educational context in naming
- [ ] Implements proper accessibility (ARIA, semantic HTML)
- [ ] Responsive design (mobile-first)
- [ ] Component tests written
- [ ] Documented with JSDoc comments

---

## Related Documentation

- [UI Component Library](../features/ui-components.md)
- [State Management](./state-management.md)
- [Testing Guide](./testing.md)

---

**Component Statistics** (as of November 2025):
- **Total Components**: 180+
- **UI Base Components**: 25
- **Educational Components**: 85
- **Page Components**: 70
- **Accessibility Score**: 98%
