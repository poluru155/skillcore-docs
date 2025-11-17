# State Management Guide

**Last Updated**: November 17, 2025  
**Difficulty**: Intermediate  
**Time**: 45-90 minutes

---

## Overview

SkillCore uses **three state management solutions** based on state type:

1. **TanStack Query** (React Query) - Server state (API data, caching)
2. **Zustand** - Client state (UI preferences, global app state)
3. **React Hook Form + Zod** - Form state (validation, submission)

---

## Architecture Principles

### State Classification

```typescript
// ✅ Server State → TanStack Query
- Student data
- Grade data
- Attendance records
- Class rosters
- Assignment data

// ✅ Client State → Zustand
- Theme preference
- Sidebar collapsed/expanded
- Selected academic session
- UI notifications
- Filter preferences

// ✅ Form State → React Hook Form
- Grade entry forms
- Student profile forms
- Assignment creation
- User preferences
```

---

## 1. TanStack Query (Server State)

### Installation & Setup

**Location**: `src/lib/query-client.ts`

```typescript
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Educational data stale times
      staleTime: 2 * 60 * 1000, // 2 minutes (grade data)
      gcTime: 5 * 60 * 1000, // 5 minutes (cache time)
      retry: 1,
      refetchOnWindowFocus: false,
    },
    mutations: {
      retry: 0,
    },
  },
})
```

**Provider Setup**: `app/layout.tsx`

```typescript
import { QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { queryClient } from '@/lib/query-client'

export default function RootLayout({ children }: Props) {
  return (
    <html lang="en">
      <body>
        <QueryClientProvider client={queryClient}>
          {children}
          <ReactQueryDevtools initialIsOpen={false} />
        </QueryClientProvider>
      </body>
    </html>
  )
}
```

---

### Query Hooks Pattern

**Location**: `src/hooks/queries/use-students.ts`

```typescript
import { useQuery, useSuspenseQuery } from '@tanstack/react-query'
import { apiService } from '@/services/apiService'
import type { Student } from '@/types/graphql-types'

// Query key factory for type safety
export const studentKeys = {
  all: ['students'] as const,
  lists: () => [...studentKeys.all, 'list'] as const,
  list: (filters: StudentFilters) => [...studentKeys.lists(), filters] as const,
  details: () => [...studentKeys.all, 'detail'] as const,
  detail: (id: string) => [...studentKeys.details(), id] as const,
}

interface StudentFilters {
  classId?: string
  gradeLevel?: string
  hasIEP?: boolean
  has504?: boolean
}

// Query hook for student list
export function useStudents(filters: StudentFilters = {}) {
  return useQuery({
    queryKey: studentKeys.list(filters),
    queryFn: () => apiService.students.getAll(filters),
    staleTime: 2 * 60 * 1000, // 2 minutes
  })
}

// Query hook for single student
export function useStudent(studentId: string) {
  return useQuery({
    queryKey: studentKeys.detail(studentId),
    queryFn: () => apiService.students.getById(studentId),
    staleTime: 2 * 60 * 1000,
    enabled: !!studentId, // Only fetch if ID exists
  })
}

// Suspense query for server components
export function useStudentSuspense(studentId: string) {
  return useSuspenseQuery({
    queryKey: studentKeys.detail(studentId),
    queryFn: () => apiService.students.getById(studentId),
  })
}

// Prefetch utility
export async function prefetchStudent(studentId: string) {
  await queryClient.prefetchQuery({
    queryKey: studentKeys.detail(studentId),
    queryFn: () => apiService.students.getById(studentId),
  })
}
```

---

### Mutation Hooks Pattern

**Location**: `src/hooks/mutations/use-update-grade.ts`

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { apiService } from '@/services/apiService'
import { useToast } from '@/hooks/use-toast'
import { gradeKeys } from '@/hooks/queries/use-grades'

interface UpdateGradeInput {
  gradeId: string
  score: number
  scoreStatus: 'not_submitted' | 'partially_graded' | 'fully_graded' | 'exempt'
  comment?: string
}

export function useUpdateGrade() {
  const queryClient = useQueryClient()
  const { toast } = useToast()
  
  return useMutation({
    mutationFn: (input: UpdateGradeInput) =>
      apiService.grades.update(input.gradeId, input),
    
    // Optimistic update
    onMutate: async (input) => {
      // Cancel outgoing queries
      await queryClient.cancelQueries({ queryKey: gradeKeys.detail(input.gradeId) })
      
      // Snapshot previous value
      const previousGrade = queryClient.getQueryData(gradeKeys.detail(input.gradeId))
      
      // Optimistically update cache
      queryClient.setQueryData(gradeKeys.detail(input.gradeId), (old: any) => ({
        ...old,
        score: input.score,
        scoreStatus: input.scoreStatus,
        comment: input.comment,
      }))
      
      return { previousGrade }
    },
    
    // Rollback on error
    onError: (error, input, context) => {
      queryClient.setQueryData(
        gradeKeys.detail(input.gradeId),
        context?.previousGrade
      )
      
      toast({
        variant: 'destructive',
        title: 'Failed to update grade',
        description: error.message,
      })
    },
    
    // Refetch on success
    onSuccess: (data, input) => {
      queryClient.invalidateQueries({ queryKey: gradeKeys.lists() })
      queryClient.invalidateQueries({ queryKey: gradeKeys.detail(input.gradeId) })
      
      toast({
        title: 'Grade updated successfully',
        description: `Score: ${data.score} (${data.scorePercent}%)`,
      })
    },
  })
}
```

---

### Usage in Components

```typescript
'use client'

import { useStudents } from '@/hooks/queries/use-students'
import { useUpdateGrade } from '@/hooks/mutations/use-update-grade'
import { StudentList } from '@/components/educational/student-list'
import { GradeEntryDialog } from '@/components/educational/grade-entry-dialog'

export function GradebookView({ classId }: Props) {
  // Query - fetch students
  const { data: students, isLoading, error } = useStudents({ classId })
  
  // Mutation - update grade
  const updateGrade = useUpdateGrade()
  
  const handleGradeSubmit = async (gradeId: string, data: GradeFormData) => {
    await updateGrade.mutateAsync({
      gradeId,
      score: data.score,
      scoreStatus: data.scoreStatus,
      comment: data.comment,
    })
  }
  
  if (isLoading) return <GradebookSkeleton />
  if (error) return <ErrorAlert error={error} />
  
  return (
    <div>
      <StudentList students={students} />
      <GradeEntryDialog onSubmit={handleGradeSubmit} />
    </div>
  )
}
```

---

### Prefetching Pattern

```typescript
'use client'

import { useEffect } from 'react'
import { useQueryClient } from '@tanstack/react-query'
import { studentKeys, prefetchStudent } from '@/hooks/queries/use-students'

export function StudentCard({ student }: Props) {
  const queryClient = useQueryClient()
  
  // Prefetch student details on hover
  const handleMouseEnter = () => {
    queryClient.prefetchQuery({
      queryKey: studentKeys.detail(student.id),
      queryFn: () => apiService.students.getById(student.id),
    })
  }
  
  return (
    <div onMouseEnter={handleMouseEnter}>
      <h3>{student.name}</h3>
      <p>Grade: {student.gradeLevel}</p>
    </div>
  )
}
```

---

## 2. Zustand (Client State)

### Store Setup

**Location**: `src/stores/app-store.ts`

```typescript
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

interface AppState {
  // Theme
  theme: 'light' | 'dark' | 'system'
  setTheme: (theme: 'light' | 'dark' | 'system') => void
  
  // Sidebar
  sidebarCollapsed: boolean
  toggleSidebar: () => void
  
  // Academic Session
  selectedSessionId: string | null
  setSelectedSession: (sessionId: string) => void
  
  // Notifications
  notifications: Notification[]
  addNotification: (notification: Omit<Notification, 'id'>) => void
  removeNotification: (id: string) => void
  clearNotifications: () => void
}

interface Notification {
  id: string
  type: 'info' | 'success' | 'warning' | 'error'
  title: string
  message: string
  timestamp: Date
}

export const useAppStore = create<AppState>()(
  persist(
    (set) => ({
      // Theme
      theme: 'system',
      setTheme: (theme) => set({ theme }),
      
      // Sidebar
      sidebarCollapsed: false,
      toggleSidebar: () => set((state) => ({ sidebarCollapsed: !state.sidebarCollapsed })),
      
      // Academic Session
      selectedSessionId: null,
      setSelectedSession: (sessionId) => set({ selectedSessionId: sessionId }),
      
      // Notifications
      notifications: [],
      addNotification: (notification) => set((state) => ({
        notifications: [
          ...state.notifications,
          { ...notification, id: crypto.randomUUID() },
        ],
      })),
      removeNotification: (id) => set((state) => ({
        notifications: state.notifications.filter((n) => n.id !== id),
      })),
      clearNotifications: () => set({ notifications: [] }),
    }),
    {
      name: 'app-storage', // localStorage key
      storage: createJSONStorage(() => localStorage),
      // Only persist certain fields
      partialize: (state) => ({
        theme: state.theme,
        sidebarCollapsed: state.sidebarCollapsed,
        selectedSessionId: state.selectedSessionId,
      }),
    }
  )
)
```

---

### Usage in Components

```typescript
'use client'

import { useAppStore } from '@/stores/app-store'
import { Button } from '@/components/ui/button'
import { Moon, Sun } from 'lucide-react'

export function ThemeToggle() {
  const theme = useAppStore((state) => state.theme)
  const setTheme = useAppStore((state) => state.setTheme)
  
  return (
    <Button
      variant="ghost"
      size="icon"
      onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}
    >
      {theme === 'light' ? <Moon /> : <Sun />}
    </Button>
  )
}
```

---

### Selectors for Performance

```typescript
'use client'

import { useAppStore } from '@/stores/app-store'
import { shallow } from 'zustand/shallow'

// ❌ Bad: Re-renders on ANY state change
function Component() {
  const state = useAppStore()
  return <div>{state.theme}</div>
}

// ✅ Good: Only re-renders when theme changes
function Component() {
  const theme = useAppStore((state) => state.theme)
  return <div>{theme}</div>
}

// ✅ Good: Select multiple values with shallow comparison
function Component() {
  const { theme, sidebarCollapsed } = useAppStore(
    (state) => ({
      theme: state.theme,
      sidebarCollapsed: state.sidebarCollapsed,
    }),
    shallow
  )
  
  return <div>{theme}</div>
}
```

---

### Derived State Pattern

```typescript
import { create } from 'zustand'

interface GradebookState {
  grades: Record<string, number>
  filters: {
    minGrade?: number
    maxGrade?: number
    status?: 'passing' | 'failing'
  }
  
  // Actions
  setGrade: (studentId: string, grade: number) => void
  setFilters: (filters: Partial<GradebookState['filters']>) => void
  
  // Derived selectors
  getFilteredGrades: () => Record<string, number>
  getClassAverage: () => number
}

export const useGradebookStore = create<GradebookState>((set, get) => ({
  grades: {},
  filters: {},
  
  setGrade: (studentId, grade) =>
    set((state) => ({
      grades: { ...state.grades, [studentId]: grade },
    })),
  
  setFilters: (filters) =>
    set((state) => ({
      filters: { ...state.filters, ...filters },
    })),
  
  // Derived selector
  getFilteredGrades: () => {
    const { grades, filters } = get()
    
    return Object.fromEntries(
      Object.entries(grades).filter(([_, grade]) => {
        if (filters.minGrade !== undefined && grade < filters.minGrade) return false
        if (filters.maxGrade !== undefined && grade > filters.maxGrade) return false
        if (filters.status === 'passing' && grade < 60) return false
        if (filters.status === 'failing' && grade >= 60) return false
        return true
      })
    )
  },
  
  // Derived selector
  getClassAverage: () => {
    const grades = Object.values(get().grades)
    if (grades.length === 0) return 0
    return grades.reduce((sum, grade) => sum + grade, 0) / grades.length
  },
}))
```

---

## 3. React Hook Form + Zod (Form State)

### Form Schema Definition

**Location**: `src/lib/validations/grade.ts`

```typescript
import { z } from 'zod'

export const gradeSchema = z.object({
  score: z
    .number({ required_error: 'Score is required' })
    .min(0, 'Score must be at least 0')
    .max(100, 'Score cannot exceed 100'),
  
  scoreStatus: z.enum(['not_submitted', 'partially_graded', 'fully_graded', 'exempt'], {
    required_error: 'Status is required',
  }),
  
  comment: z.string().max(500, 'Comment cannot exceed 500 characters').optional(),
  
  dateGraded: z.date().optional(),
})

export type GradeFormData = z.infer<typeof gradeSchema>
```

---

### Form Component

```typescript
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { gradeSchema, type GradeFormData } from '@/lib/validations/grade'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form'

interface GradeFormProps {
  defaultValues?: Partial<GradeFormData>
  onSubmit: (data: GradeFormData) => Promise<void>
}

export function GradeForm({ defaultValues, onSubmit }: GradeFormProps) {
  const form = useForm<GradeFormData>({
    resolver: zodResolver(gradeSchema),
    defaultValues: {
      scoreStatus: 'fully_graded',
      ...defaultValues,
    },
  })
  
  const handleSubmit = async (data: GradeFormData) => {
    try {
      await onSubmit(data)
      form.reset()
    } catch (error) {
      form.setError('root', {
        message: 'Failed to save grade. Please try again.',
      })
    }
  }
  
  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(handleSubmit)} className="space-y-6">
        {/* Score Field */}
        <FormField
          control={form.control}
          name="score"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Score (0-100)</FormLabel>
              <FormControl>
                <Input
                  type="number"
                  min="0"
                  max="100"
                  step="0.1"
                  {...field}
                  onChange={(e) => field.onChange(parseFloat(e.target.value))}
                />
              </FormControl>
              <FormDescription>
                Enter the student's score for this assignment
              </FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />
        
        {/* Status Field */}
        <FormField
          control={form.control}
          name="scoreStatus"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Status</FormLabel>
              <FormControl>
                <Select onValueChange={field.onChange} defaultValue={field.value}>
                  <SelectTrigger>
                    <SelectValue />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="fully_graded">Fully Graded</SelectItem>
                    <SelectItem value="partially_graded">Partially Graded</SelectItem>
                    <SelectItem value="not_submitted">Not Submitted</SelectItem>
                    <SelectItem value="exempt">Exempt</SelectItem>
                  </SelectContent>
                </Select>
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        
        {/* Comment Field */}
        <FormField
          control={form.control}
          name="comment"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Comment (optional)</FormLabel>
              <FormControl>
                <Textarea {...field} rows={3} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        
        {/* Submit Button */}
        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? 'Saving...' : 'Save Grade'}
        </Button>
        
        {/* Root Error */}
        {form.formState.errors.root && (
          <p className="text-sm text-red-500">
            {form.formState.errors.root.message}
          </p>
        )}
      </form>
    </Form>
  )
}
```

---

## State Management Best Practices

### 1. State Colocation

```typescript
✅ Good: Keep state close to where it's used
function StudentCard({ student }: Props) {
  const [isExpanded, setIsExpanded] = useState(false) // Local state
  
  return (
    <Card>
      <Button onClick={() => setIsExpanded(!isExpanded)}>
        {isExpanded ? 'Collapse' : 'Expand'}
      </Button>
      {isExpanded && <StudentDetails student={student} />}
    </Card>
  )
}

❌ Bad: Global state for local UI
const useAppStore = create((set) => ({
  studentCardExpanded: {},
  toggleStudentCard: (id) => { /* ... */ },
}))
```

### 2. Cache Invalidation

```typescript
✅ Good: Invalidate related queries
onSuccess: (data) => {
  queryClient.invalidateQueries({ queryKey: gradeKeys.lists() })
  queryClient.invalidateQueries({ queryKey: studentKeys.detail(data.studentId) })
}

❌ Bad: Invalidate everything
onSuccess: () => {
  queryClient.invalidateQueries() // Too broad!
}
```

### 3. Error Handling

```typescript
✅ Good: User-friendly error messages
onError: (error) => {
  toast({
    variant: 'destructive',
    title: 'Failed to save grade',
    description: 'Please check your internet connection and try again.',
  })
}

❌ Bad: Raw error messages
onError: (error) => {
  alert(error.message) // Technical jargon
}
```

---

## Testing State Management

### Testing TanStack Query Hooks

```typescript
import { describe, it, expect, vi } from 'vitest'
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useStudents } from '@/hooks/queries/use-students'
import { apiService } from '@/services/apiService'

vi.mock('@/services/apiService')

describe('useStudents', () => {
  it('should fetch students successfully', async () => {
    const mockStudents = [
      { id: '1', name: 'John Doe' },
      { id: '2', name: 'Jane Smith' },
    ]
    
    vi.mocked(apiService.students.getAll).mockResolvedValue(mockStudents)
    
    const queryClient = new QueryClient()
    const wrapper = ({ children }) => (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    )
    
    const { result } = renderHook(() => useStudents(), { wrapper })
    
    await waitFor(() => expect(result.current.isSuccess).toBe(true))
    
    expect(result.current.data).toEqual(mockStudents)
  })
})
```

### Testing Zustand Stores

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { useAppStore } from '@/stores/app-store'

describe('AppStore', () => {
  beforeEach(() => {
    useAppStore.setState({
      theme: 'system',
      sidebarCollapsed: false,
    })
  })
  
  it('should toggle sidebar', () => {
    expect(useAppStore.getState().sidebarCollapsed).toBe(false)
    
    useAppStore.getState().toggleSidebar()
    
    expect(useAppStore.getState().sidebarCollapsed).toBe(true)
  })
  
  it('should set theme', () => {
    useAppStore.getState().setTheme('dark')
    
    expect(useAppStore.getState().theme).toBe('dark')
  })
})
```

---

## State Management Checklist

- [ ] Server state uses TanStack Query
- [ ] Client state uses Zustand
- [ ] Form state uses React Hook Form + Zod
- [ ] Query keys use factory pattern
- [ ] Mutations invalidate related queries
- [ ] Optimistic updates for better UX
- [ ] Error handling with user-friendly messages
- [ ] Zustand selectors for performance
- [ ] State persisted when needed (localStorage)
- [ ] Tests written for hooks and stores

---

## Related Documentation

- [Adding Components](./adding-components.md)
- [Testing Guide](./testing.md)
- [GraphQL API](../api/graphql.md)

---

**State Management Statistics** (as of November 2025):
- **Query Hooks**: 45+
- **Mutation Hooks**: 30+
- **Zustand Stores**: 3
- **Cache Hit Rate**: 85%
- **Average Query Time**: 120ms
