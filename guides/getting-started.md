# Getting Started

**Last Updated**: November 16, 2025  
**Purpose**: Setup, installation, and development guide

---

## Prerequisites

### Required Software
- **Node.js**: Latest LTS (v20+)
- **pnpm**: v9.0+ (REQUIRED - do not use npm or yarn)
- **Git**: v2.40+
- **VS Code**: Recommended IDE

### Recommended VS Code Extensions
- ESLint
- Prettier - Code formatter
- Tailwind CSS IntelliSense
- TypeScript Vue Plugin (Volar)
- GraphQL: Language Feature Support

---

## Initial Setup

### 1. Clone Repository
```bash
git clone https://github.com/poluru155/skillcore-spa.git
cd skillcore-spa
```

### 2. Install Dependencies
```bash
# Install pnpm globally if needed
npm install -g pnpm

# Install project dependencies
pnpm install
```

### 3. Environment Configuration
```bash
# Copy environment template
cp .env.example .env.local

# Edit .env.local with your configuration
# See Environment Variables section below
```

### 4. Start Development Server
```bash
# Start frontend dev server (port 3002)
pnpm dev

# Open browser to http://localhost:3002
```

---

## Environment Variables

Create `.env.local` with these variables:

### Frontend Configuration
```bash
# API Mode (graphql, rest, or external_mock)
VITE_API_MODE=external_mock

# API URLs
VITE_API_URL=http://localhost:8081
VITE_GRAPHQL_URL=http://localhost:8080/graphql
VITE_WS_URL=ws://localhost:8080/graphql

# Feature Flags
VITE_USE_GRAPHQL_SUBSCRIPTIONS=false
VITE_USE_WEBSOCKET_NOTIFICATIONS=false
VITE_ENABLE_ANALYTICS=true

# Authentication
VITE_AUTH_PROVIDER=local
VITE_JWT_SECRET=your-secret-key
```

### Backend Configuration (if running locally)
```bash
DATABASE_URL=postgresql://user:pass@localhost:5432/skillcore
REDIS_URL=redis://localhost:6379
NODE_ENV=development
```

---

## Available Scripts

### Development
```bash
# Start dev server with hot reload
pnpm dev

# Start with specific port
pnpm dev --port 3003

# Start with host binding
pnpm dev --host
```

### Building
```bash
# Type check without building
pnpm type-check

# Build for production
pnpm build

# Preview production build
pnpm preview
```

### Testing
```bash
# Run all tests
pnpm test

# Run tests in watch mode
pnpm test:watch

# Run tests with coverage
pnpm test:coverage
```

### Code Quality
```bash
# Run ESLint
pnpm lint

# Fix ESLint errors
pnpm lint:fix

# Format code with Prettier
pnpm format

# Type check
pnpm type-check
```

### Bundle Analysis
```bash
# Analyze bundle size
pnpm bundle:analyze

# Check bundle size history
pnpm bundle:size

# Guard against bundle size regression
pnpm bundle:guard
```

---

## Project Structure

```
skillcore-spa/
├── src/                          # Source code
│   ├── components/              # React components
│   │   ├── ui/                  # Base UI components (PRIORITY 1)
│   │   ├── educational/         # Domain components
│   │   ├── forms/               # Form components
│   │   ├── charts/              # Data visualization
│   │   └── layout/              # Layout components
│   ├── pages/                   # Route pages (React Router)
│   │   ├── teacher/             # Teacher pages
│   │   ├── student/             # Student pages
│   │   ├── parent/              # Parent pages
│   │   ├── counselor/           # Counselor pages
│   │   └── admin/               # Admin pages
│   ├── services/                # Service layer
│   │   ├── role-based/          # Role-specific services
│   │   ├── apiService.ts        # API integration
│   │   └── educationalDataService.ts
│   ├── hooks/                   # Custom React hooks
│   │   ├── queries/             # TanStack Query hooks
│   │   └── mutations/           # Mutation hooks
│   ├── stores/                  # Zustand stores
│   ├── types/                   # TypeScript types
│   ├── lib/                     # Utility functions
│   └── graphql/                 # GraphQL operations
├── public/                      # Static assets
├── docs-new/                    # Documentation (this folder)
└── memory-bank/                 # Development notes
```

---

## Development Workflow

### 1. Create New Feature
```bash
# Create feature branch
git checkout -b feature/my-feature

# Make changes following component guidelines
# See docs-new/guides/components.md
```

### 2. Follow Coding Standards
```typescript
// ✅ CORRECT: UI folder components first
import { Button } from '@/components/ui/button'
import { Card } from '@/components/ui/card'

// ✅ CORRECT: Radix UI for layout (only if no UI equivalent)
import { Text, Flex, Badge } from '@radix-ui/themes'

// ❌ WRONG: Custom styled divs
<div className="p-4 bg-white rounded"> {/* Use Card instead */}
```

### 3. Type Safety
```typescript
// ✅ CORRECT: Strict TypeScript
interface StudentProps {
  student: Student
  onSelect: (student: Student) => void
}

// ❌ WRONG: Using 'any'
function handleClick(data: any) { /* Never use 'any' */ }
```

### 4. State Management
```typescript
// ✅ CORRECT: TanStack Query for server state
const { data: students } = useQuery({
  queryKey: ['students', classId],
  queryFn: () => api.getStudents(classId)
})

// ✅ CORRECT: Zustand for UI state
const theme = useAppStore(state => state.theme)

// ❌ WRONG: useState for server data
const [students, setStudents] = useState([]) // Use TanStack Query
```

### 5. Commit Changes
```bash
# Run checks before commit
pnpm type-check
pnpm lint
pnpm test

# Commit with descriptive message
git add .
git commit -m "feat: add student dashboard component"

# Push to remote
git push origin feature/my-feature
```

---

## Testing Your Changes

### Unit Tests
```bash
# Run tests for specific file
pnpm test src/components/ui/button.test.tsx

# Run tests in watch mode
pnpm test:watch
```

### Manual Testing
1. Start dev server: `pnpm dev`
2. Login with test credentials:
   - Teacher: `jsmith` / `Password123!`
   - Student: `student1` / `password`
   - Admin: `admin` / `admin123`
3. Navigate to your feature
4. Test all user interactions
5. Check browser console for errors
6. Verify responsive design (mobile, tablet, desktop)

### Accessibility Testing
```bash
# Install axe DevTools extension in browser
# Run accessibility audit on your pages
# Fix any WCAG 2.1 AA violations
```

---

## Common Development Tasks

### Adding a New Page
```typescript
// 1. Create page component
// src/pages/teacher/my-new-page.tsx
export default function MyNewPage() {
  return <div>My New Page</div>
}

// 2. Add route
// src/router.tsx
{
  path: '/teacher/my-new-page',
  element: <MyNewPage />
}
```

### Creating a UI Component
```typescript
// 1. Create in ui folder if reusable
// src/components/ui/my-component.tsx

// 2. Follow Radix UI patterns
import * as React from 'react'
import { cn } from '@/lib/utils'

export interface MyComponentProps {
  children: React.ReactNode
  className?: string
}

export function MyComponent({ children, className }: MyComponentProps) {
  return (
    <div className={cn('base-styles', className)}>
      {children}
    </div>
  )
}
```

### Adding API Integration
```typescript
// 1. Define types
// src/types/entities/my-entity.ts
export interface MyEntity {
  id: string
  name: string
}

// 2. Create service method
// src/services/role-based/TeacherService.ts
async getMyEntities(): Promise<MyEntity[]> {
  return this.executeServiceMethod('getMyEntities')
}

// 3. Create React Query hook
// src/hooks/queries/use-my-entities.ts
export function useMyEntities() {
  return useQuery({
    queryKey: ['my-entities'],
    queryFn: () => teacherService.getMyEntities()
  })
}

// 4. Use in component
const { data, isLoading } = useMyEntities()
```

---

## Troubleshooting

### Common Issues

#### Port Already in Use
```bash
# Error: Port 3002 is already in use
# Solution: Kill process or use different port
pnpm dev --port 3003
```

#### pnpm Install Fails
```bash
# Error: Dependencies not resolving
# Solution: Clear cache and reinstall
pnpm store prune
rm -rf node_modules
pnpm install
```

#### TypeScript Errors
```bash
# Error: Type errors after pulling changes
# Solution: Restart TypeScript server
# VS Code: Cmd+Shift+P → "TypeScript: Restart TS Server"
```

#### Hot Reload Not Working
```bash
# Error: Changes not reflecting
# Solution: Restart dev server
# Kill: Ctrl+C
# Start: pnpm dev
```

### Getting Help

1. **Check Documentation**: Review docs-new/ folder
2. **Search Issues**: Look for similar problems in GitHub issues
3. **Check Memory Bank**: Review memory-bank/ for dev notes
4. **Ask Team**: Reach out in team chat

---

## Next Steps

### For New Developers
1. ✅ Complete this setup guide
2. 📖 Read [Architecture Overview](../architecture/overview.md)
3. 📖 Review [Component Guidelines](./components.md)
4. 📖 Check [Implementation Gaps](../status/gaps.md)
5. 🎯 Pick a small issue to start with

### For Experienced Developers
1. 📊 Review [Current Architecture](../architecture/current.md)
2. 🚧 Check [In Progress](../status/in-progress.md) work
3. 🎯 Review [Implementation Gaps](../status/gaps.md)
4. 💡 Choose feature to work on

---

## Additional Resources

### Documentation
- [Architecture Overview](../architecture/overview.md)
- [API Integration](../architecture/api-integration.md)
- [Component Guidelines](./components.md)
- [State Management Guide](./state-management.md)
- [Testing Guide](./testing.md)

### External Links
- [React Documentation](https://react.dev/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [TanStack Query](https://tanstack.com/query/latest)
- [Radix UI](https://www.radix-ui.com/)
- [Tailwind CSS](https://tailwindcss.com/)

---

**Need Help?** Check the troubleshooting section or review related documentation.
