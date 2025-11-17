# Architecture Overview

**Last Updated**: November 16, 2025

---

## System Architecture

SkillCore WebApp is a modern educational platform built with React and TypeScript, featuring a comprehensive service-oriented architecture with GraphQL and REST API integration.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Client Layer (React SPA)                │
│  - Next.js 15 App Router                                    │
│  - TypeScript 5.8+ (Strict Mode)                            │
│  - Tailwind CSS 4.1 + Radix UI                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   State Management Layer                     │
│  - TanStack Query 5.90 (Server State)                       │
│  - Zustand 5.0 (Client State)                               │
│  - Apollo Client (GraphQL)                                  │
│  - React Hook Form + Zod (Forms)                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   Service Layer                              │
│  - Role-Based Services (Teacher, Student, Admin, Parent)    │
│  - Educational Data Service                                 │
│  - API Service (GraphQL/REST switcher)                      │
│  - Cache Service (Multi-level)                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                     API Layer                                │
│  ┌─────────────────┬──────────────────┬──────────────────┐ │
│  │   GraphQL API   │    REST API      │   WebSocket      │ │
│  │  (127 ops)      │  (47+ endpoints) │  (Subscriptions) │ │
│  └─────────────────┴──────────────────┴──────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   Backend Services                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Domain Layer (DDD Bounded Contexts)                   │ │
│  │  - Assessment, Rostering, Attendance, Gradebook        │ │
│  │  - Special Programs, Chat, Communication               │ │
│  │  - Demographics, Resources, Payments                   │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Application Layer (CQRS)                              │ │
│  │  - Commands (47+ handlers)                             │ │
│  │  - Queries (Read models)                               │ │
│  │  - Application Services                                │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Presentation Layer (Dual API)                         │ │
│  │  - GraphQL API (127 operations, subscriptions)         │ │
│  │  - REST API (47+ endpoints, backward compatible)       │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              Event-Driven Infrastructure                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Redis EventBus (Distributed)                          │ │
│  │  - 208 Domain Events emitting                          │ │
│  │  - 79 Event Handlers registered                        │ │
│  │  - Redis Streams for persistence                       │ │
│  │  - Consumer groups for scaling                         │ │
│  │  - Dead letter queue (DLQ)                             │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Event Workers (Background Processing)                 │ │
│  │  - Standalone worker processes                         │ │
│  │  - BullMQ queues (7 production queues)                 │ │
│  │  - Retry logic with exponential backoff                │ │
│  │  - Graceful shutdown handling                          │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│            Multi-Channel Notification System                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Email (SendGrid)                                      │ │
│  │  - Transactional emails, Templates                     │ │
│  │  - Webhooks: Delivered, Bounced, Opened, Clicked      │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  SMS (Twilio)                                          │ │
│  │  - Emergency alerts, Attendance notifications          │ │
│  │  - Webhooks: Delivered, Failed, Status callbacks      │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Push Notifications (Firebase FCM)                     │ │
│  │  - iOS/Android push, Web push                          │ │
│  │  - Topic-based messaging, Token management             │ │
│  │  - Delivery callbacks and analytics                    │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Notification Analytics                                │ │
│  │  - Delivery tracking, Engagement metrics               │ │
│  │  - NotificationDeliveryLog (18 event types)            │ │
│  │  - NotificationEngagement, NotificationAnalytics       │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                  Scheduled Jobs & Workers                    │
│  - Stream cleanup cron (daily retention)                    │
│  - Attendance intervention detection                         │
│  - Grade posting notifications                               │
│  - IEP review reminders                                     │
│  - Report generation workers                                 │
│  - Data archival jobs                                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                   Data Layer                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  PostgreSQL (Primary Database)                         │ │
│  │  - Prisma ORM 6.2                                      │ │
│  │  - Row-level security (RLS)                            │ │
│  │  - Multi-tenant isolation                              │ │
│  │  - Audit logging tables                                │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Redis (Multiple Use Cases)                            │ │
│  │  - EventBus streams and consumer groups                │ │
│  │  - Application caching layer                           │ │
│  │  - Session storage                                     │ │
│  │  - Rate limiting                                       │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  EventStore (FERPA Compliance)                         │ │
│  │  - All domain events persisted                         │ │
│  │  - Complete audit trail                                │ │
│  │  - Event replay capability                             │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

### Frontend Core
- **Framework**: React 19.1.1 (Concurrent features)
- **Language**: TypeScript 5.8+ (Strict mode)
- **Build Tool**: Vite 7.1.7 (Fast HMR)
- **Package Manager**: pnpm (Required)
- **Router**: React Router DOM 7.9.1

### UI & Styling
- **CSS Framework**: Tailwind CSS 4.1.13 (New CSS API)
- **Component Library**: Radix UI (Headless components)
- **Icons**: Phosphor React + Lucide React
- **Animations**: Framer Motion 12.23.18
- **Charts**: Recharts (Data visualization)

### State Management
- **Server State**: TanStack Query 5.90.1 + DevTools
- **Client State**: Zustand 5.0.8
- **GraphQL Client**: Apollo Client 3.x
- **Forms**: React Hook Form 7.63 + Zod 4.1.11

### Backend Integration
- **GraphQL**: Apollo Client 3.x with code generation
- **REST**: Axios with interceptors + Fastify 5.1
- **WebSocket**: graphql-ws 6.0.6 (Subscriptions)
- **Real-time**: Redis Streams + EventBus

### External Services
- **Email**: SendGrid API (transactional + webhooks)
- **SMS**: Twilio API (emergency alerts + status webhooks)
- **Push**: Firebase Cloud Messaging (iOS/Android/Web)
- **Storage**: AWS S3 (file uploads, backups)
- **CDN**: CloudFront (asset delivery)

### Background Processing
- **Queue System**: BullMQ with Redis
- **Event Workers**: Standalone Node.js processes
- **Scheduled Jobs**: Cron-based task scheduling
- **Retry Logic**: Exponential backoff with DLQ

---

## Design Principles

### 1. Educational Standards Compliance
- **OneRoster 1.1**: Student/teacher/class interoperability
- **FERPA**: Student data privacy and protection
- **WCAG 2.1 AA**: Accessibility compliance
- **Multi-tenant**: District/school isolation

### 2. UI Component Hierarchy (MANDATORY)
```typescript
// Priority Order (STRICTLY ENFORCED):
1. UI Folder Components (/components/ui/) - ALWAYS FIRST
   - Button, Input, Card, Dialog, etc.
   
2. Radix UI Themes (ONLY if no UI equivalent)
   - Text, Badge, Flex, Box, Grid
   
3. Custom Components (LAST RESORT)
   - Only when neither UI nor Radix can achieve result
```

### 3. Service Architecture Patterns
- **Role-Based Access**: Services scoped to user roles
- **FERPA Compliance**: Audit logging for all data access
- **Multi-level Caching**: Memory → Session → LocalStorage
- **Defensive Programming**: Fallbacks for missing data

### 4. Type Safety
- **Strict TypeScript**: No `any` types allowed
- **Zod Validation**: Runtime type checking
- **GraphQL Code Gen**: Type-safe API calls
- **Educational Domain Types**: Strong typing throughout

---

## Application Structure

```
skillcore-spa/
├── src/
│   ├── app/                     # Next.js app router (future)
│   ├── components/
│   │   ├── ui/                  # Base UI components (PRIORITY 1)
│   │   ├── educational/         # Educational domain components
│   │   ├── forms/               # Form components
│   │   ├── charts/              # Data visualization
│   │   └── layout/              # Layout components
│   ├── pages/                   # Route pages
│   │   ├── teacher/             # Teacher role pages
│   │   ├── student/             # Student role pages
│   │   ├── parent/              # Parent role pages
│   │   ├── counselor/           # Counselor role pages
│   │   └── admin/               # Admin role pages
│   ├── services/
│   │   ├── role-based/          # Role-specific services
│   │   ├── apiService.ts        # API integration layer
│   │   ├── educationalDataService.ts
│   │   └── cacheService.ts
│   ├── hooks/
│   │   ├── queries/             # TanStack Query hooks
│   │   ├── mutations/           # Mutation hooks
│   │   └── use-*.ts             # Custom hooks
│   ├── stores/                  # Zustand stores
│   ├── types/                   # TypeScript types
│   ├── lib/                     # Utilities
│   └── graphql/
│       ├── operations/          # GraphQL operations
│       ├── fragments/           # Reusable fragments
│       └── generated/           # Generated types
├── api/                         # Backend API (separate repo context)
│   └── src/
│       └── contexts/            # DDD bounded contexts
│           ├── assessment-unified/
│           ├── rostering/
│           ├── attendance/
│           ├── gradebook/
│           └── special-programs/
└── docs-new/                    # This documentation
```

---

## Key Architectural Decisions

### 1. Assessment DDD Consolidation (63% Complete)
**Why**: Original architecture had incorrect bounded contexts (assessment, summative-assessment, gradebook as separate contexts)

**Solution**: Unified assessment context with:
- Formative assessments (exit tickets, quick polls)
- Summative assessments (tests, projects, rubrics)
- Grading (grade book, grade entries)
- Auto-grading (AI-powered)

**Status**: Phase 8/12 complete, GraphQL resolvers done

### 2. Event-Driven Architecture
**Implementation**: Redis EventBus with distributed processing

**Event Infrastructure**:
- **208 Domain Events** emitting across 7 bounded contexts
- **79 Event Handlers** registered and operational
- **Redis Streams** for event persistence and ordering
- **Consumer Groups** for horizontal scaling
- **Dead Letter Queue** for failed event processing
- **EventStore** (PostgreSQL) for FERPA audit trail

**Event Coverage by Context**:
- ✅ Special Programs (25 events, 6 handlers) - IEP, 504, ESL, Gifted
- ✅ Communication (15 events, 8 handlers) - Messages, Announcements
- ✅ Notifications (18 events, 6 handlers) - Email, SMS, Push delivery
- ✅ Gradebook (12 handlers) - Grade updates, calculations
- ✅ Attendance (4 handlers) - Attendance tracking, interventions
- ✅ Chat (10 handlers) - FERPA-compliant messaging
- ✅ Demographics (6 handlers) - Student/teacher updates
- ✅ Rostering (12 handlers) - Enrollment, class management

**CQRS Commands**:
- **47+ Command Handlers** for write operations
- **Command validation** with Zod schemas
- **Optimistic concurrency** control
- **Event sourcing** for critical entities

### 3. Role-Based Services
**Pattern**: Separate service classes for each role
- `TeacherService` - Class management, gradebook, scheduling
- `StudentService` - Grades, assignments, progress
- `ParentService` - Child monitoring (FERPA-compliant)
- `AdminService` - 3-level hierarchy (school/district/system)

### 4. Dual API Implementation (GraphQL + REST)
**Strategy**: 
- **GraphQL Primary**: Complex queries, nested data, real-time
- **REST Fallback**: Simple CRUD, backward compatibility, webhooks
- **WebSocket**: GraphQL subscriptions for real-time updates

**GraphQL API**:
- **127 Operations Total**:
  - 60+ Queries (complex reads, analytics)
  - 50+ Mutations (write operations)
  - 15+ Subscriptions (real-time updates)
- **Schema**: Type-safe with code generation
- **Resolvers**: Field-level resolution with DataLoader
- **Subscriptions**: Grade updates, attendance, notifications, messages

**REST API**:
- **47+ Endpoints**: CRUD operations across contexts
- **Backward Compatible**: Legacy client support
- **Webhook Receivers**: SendGrid, Twilio, Firebase callbacks
- **OpenAPI/Swagger**: Complete API documentation

**WebSocket Integration**:
- **graphql-ws 6.0.6**: Standards-compliant subscriptions
- **Auto-reconnection**: 5 retry attempts with backoff
- **JWT Authentication**: Secure WebSocket connections
- **Lifecycle Management**: Connected/disconnected/error logging

---

## Performance Characteristics

### Bundle Size (Current)
- **Main Entry**: ~264 KB raw
- **Charts Lib**: ~438 KB (Recharts)
- **Total CSS**: ~869 KB (target: <200 KB gzipped)
- **Total Build**: 6.4 MB (precache)

### Optimization Strategy
1. ✅ Route-level lazy loading
2. ✅ Manual chunk splitting by domain
3. ⏳ Tailwind CSS pruning (in progress)
4. ⏳ Progressive hydration for analytics
5. ⏳ Chart library optimization

---

## Security & Compliance

### FERPA Compliance
- Role-based data access control
- Audit logging for all sensitive operations
- Parent consent management
- Data sanitization based on user role

### Multi-Tenant Security
- Tenant context in all API calls
- Row-level security (RLS) in database
- District/school data isolation
- Cross-tenant access prevention

### Authentication & Authorization
- JWT-based authentication
- Role-based access control (RBAC)
- Session management
- Automatic token refresh

---

## Event Handlers & Processors

### Notification Delivery Handlers (6 handlers)
- **NotificationDeliveryAnalyticsHandler**: Tracks delivery across all channels
- **EmailDeliveryHandler**: SendGrid webhook processing
- **SMSDeliveryHandler**: Twilio webhook processing
- **PushNotificationHandler**: Firebase FCM callbacks
- **NotificationEngagementHandler**: Opens, clicks, replies
- **DeliveryFailureHandler**: Retry logic and DLQ management

### Special Programs Handlers (6 handlers)
- **CoordinatorNotificationHandler**: Program coordinator alerts
- **TeacherAccommodationHandler**: Accommodation notifications
- **IEPInterventionHandler**: Auto-create Tier 2 interventions
- **StudentProfileUpdateHandler**: Student metadata updates
- **ComplianceReportHandler**: IDEA/Section 504 audit logging
- **ArchiveProgramDataHandler**: Program exit audit trail

### Gradebook Handlers (12 handlers)
- Grade calculation and aggregation
- GPA updates and letter grade conversion
- Grade posting notifications
- Assignment weight recalculation
- Category average updates
- Class performance metrics

### Attendance Handlers (4 handlers)
- Daily attendance aggregation
- Absence intervention triggers
- Attendance rate calculations
- Parent notification dispatch

### Communication Handlers (8 handlers)
- Message delivery coordination
- Announcement publishing
- Conference scheduling
- Parent-teacher communication routing
- Read receipt processing
- Notification preference management

---

## Multi-Channel Notification Architecture

### Channel Implementations

**Email Channel (SendGrid)**:
- **Events**: EmailSent, EmailDelivered, EmailBounced, EmailOpened, EmailClicked, EmailDeliveryFailed
- **Templates**: Welcome, Grade Posted, Attendance Alert, Assignment Due
- **Webhooks**: Real-time delivery status processing
- **Analytics**: Open rates, click-through rates, bounce tracking

**SMS Channel (Twilio)**:
- **Events**: SMSSent, SMSDelivered, SMSFailed, SMSDeliveryFailed, SMSReplied
- **Use Cases**: Emergency alerts, attendance notifications, quick reminders
- **Webhooks**: Status callbacks for delivery confirmation
- **Compliance**: TCPA compliance, opt-out management

**Push Notification Channel (Firebase FCM)**:
- **Events**: PushNotificationSent, PushNotificationDelivered, PushNotificationClicked, PushNotificationFailed, PushNotificationDeliveryFailed
- **Platforms**: iOS, Android, Web (PWA)
- **Features**: Topic subscriptions, token management, badge updates
- **Analytics**: Delivery rates, click-through rates, engagement tracking

### Multi-Channel Orchestration
- **Priority-based routing**: High-priority sends to all channels
- **Fallback logic**: SMS if email fails for urgent messages
- **User preferences**: Per-channel, per-notification-type toggles
- **Delivery tracking**: Unified analytics across all channels
- **Compliance**: FERPA-compliant logging for all communications

---

## Background Workers & Scheduled Jobs

### Event Workers
- **Deployment**: Standalone Node.js processes (separate from API)
- **Configuration**: `pnpm worker` or `pnpm worker:dev`
- **Scaling**: Multiple workers consume from same consumer group
- **Graceful Shutdown**: SIGTERM/SIGINT handling
- **Health Checks**: `/health/events` endpoint with stream metrics

### BullMQ Queues (7 Production Queues)
1. **notification-delivery**: Email, SMS, Push dispatching
2. **grade-processing**: Grade calculation and posting
3. **attendance-aggregation**: Daily attendance rollups
4. **report-generation**: PDF/Excel report creation
5. **data-archival**: Old data cleanup and archiving
6. **intervention-detection**: At-risk student identification
7. **compliance-reporting**: FERPA audit log generation

### Scheduled Jobs (Cron)
- **Daily**:
  - Stream cleanup (Redis retention management)
  - Attendance intervention detection
  - IEP review reminders (30/60/90 days)
  - Grade posting digest emails
- **Weekly**:
  - Performance trend calculations
  - Resource usage analytics
  - Inactive user notifications
- **Monthly**:
  - Compliance audit reports
  - Data archival to cold storage
  - System health summaries

### Retry & Error Handling
- **Exponential Backoff**: 1s, 2s, 4s, 8s, 16s delays
- **Max Retries**: 5 attempts before DLQ
- **Dead Letter Queue**: Failed events for manual review
- **Monitoring**: Prometheus metrics for queue depth, processing time
- **Alerts**: PagerDuty integration for critical failures

---

## Next Steps

See [Future Architecture](./future.md) for planned improvements and roadmap.

---

**Related Documents**:
- [Current Architecture](./current.md) - Current implementation details
- [Event Architecture](./events.md) - Complete event system documentation
- [API Integration](./api-integration.md) - API integration patterns
- [Database Schema](./database.md) - Data models and relationships
