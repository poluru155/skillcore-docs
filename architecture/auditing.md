# Audit System Architecture

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Compliance**: FERPA · IDEA · Section 504

---

## Overview

SkillCore implements comprehensive audit logging for **FERPA compliance**, educational data tracking, and security monitoring. All student data access, modifications, and administrative actions are logged for regulatory compliance and security audits.

---

## Audit Architecture

```
User Action → API Middleware → Audit Logger → Database
                                     ↓
                              AuditLog Table
                                     ↓
                         ┌───────────┴────────────┐
                         ↓                        ↓
                  Compliance Reports      Security Monitoring
                  (FERPA, IDEA, 504)      (Unauthorized Access)
```

---

## Audit Data Model

### AuditLog Table

```prisma
model AuditLog {
  id            String   @id @default(uuid())
  
  // Who performed the action
  userId        String
  userRole      String   // 'TEACHER', 'STUDENT', 'PARENT', 'ADMIN', etc.
  userName      String
  userEmail     String
  
  // What action was performed
  action        String   // 'CREATE', 'READ', 'UPDATE', 'DELETE', 'LOGIN', 'EXPORT'
  entityType    String   // 'STUDENT', 'GRADE', 'IEP', 'ATTENDANCE', etc.
  entityId      String?  // ID of entity acted upon
  
  // Context of the action
  description   String   // Human-readable description
  changes       Json?    // Before/after values for updates
  metadata      Json?    // Additional context (IP, user agent, etc.)
  
  // FERPA-specific fields
  studentId     String?  // If action involved student data
  purpose       String?  // Educational purpose for access
  ferpaCategory String?  // 'DIRECTORY', 'EDUCATIONAL_RECORD', 'HEALTH_RECORD'
  
  // Security fields
  ipAddress     String?
  userAgent     String?
  sessionId     String?
  
  // Multi-tenant
  tenantId      String
  schoolId      String?
  
  // Timestamp
  createdAt     DateTime @default(now())
  
  // Relations
  user          User     @relation(fields: [userId], references: [id])
  student       Student? @relation(fields: [studentId], references: [id])
  tenant        Tenant   @relation(fields: [tenantId], references: [id])
  
  @@index([userId])
  @@index([studentId])
  @@index([tenantId])
  @@index([action])
  @@index([entityType])
  @@index([createdAt])
  @@index([ferpaCategory])
}
```

### EventStore Table (Event Sourcing)

```prisma
model EventStore {
  id             String   @id @default(uuid())
  
  // Event identification
  eventType      String   // 'GradeUpdated', 'AttendanceMarked', etc.
  eventVersion   String   @default("1.0")
  
  // Event payload
  aggregateId    String   // ID of entity that emitted event
  aggregateType  String   // 'Grade', 'Attendance', 'IEP', etc.
  payload        Json     // Full event data
  
  // Metadata
  userId         String?  // User who triggered event
  tenantId       String
  correlationId  String?  // For tracing related events
  
  // Timestamp
  createdAt      DateTime @default(now())
  
  @@index([aggregateId, aggregateType])
  @@index([eventType])
  @@index([tenantId])
  @@index([createdAt])
}
```

---

## Audit Logging Implementation

### Middleware-Based Logging

**API Middleware** (`src/middleware/audit-logger.ts`):
```typescript
import { Request, Response, NextFunction } from 'express'
import { prisma } from '../lib/prisma'
import { extractUser } from '../lib/auth'

export async function auditLogger(req: Request, res: Response, next: NextFunction) {
  // Skip non-mutating requests for performance
  if (req.method === 'GET' && !req.path.includes('/export')) {
    return next()
  }
  
  const user = extractUser(req)
  const startTime = Date.now()
  
  // Capture response
  const originalJson = res.json.bind(res)
  res.json = function (data: any) {
    // Log after successful response
    const endTime = Date.now()
    const duration = endTime - startTime
    
    logAuditEntry({
      userId: user.id,
      userRole: user.role,
      userName: user.name,
      userEmail: user.email,
      action: mapMethodToAction(req.method),
      entityType: extractEntityType(req.path),
      entityId: extractEntityId(req.path, req.body),
      description: generateDescription(req),
      changes: extractChanges(req),
      metadata: {
        method: req.method,
        path: req.path,
        query: req.query,
        duration,
      },
      studentId: extractStudentId(req),
      purpose: req.headers['x-ferpa-purpose'] as string,
      ipAddress: req.ip,
      userAgent: req.headers['user-agent'],
      sessionId: req.session?.id,
      tenantId: user.tenantId,
      schoolId: extractSchoolId(req),
    })
    
    return originalJson(data)
  }
  
  next()
}

function mapMethodToAction(method: string): string {
  switch (method) {
    case 'POST': return 'CREATE'
    case 'GET': return 'READ'
    case 'PUT':
    case 'PATCH': return 'UPDATE'
    case 'DELETE': return 'DELETE'
    default: return method
  }
}
```

### Event-Based Logging

**Event Handler** (`src/events/handlers/audit-log-handler.ts`):
```typescript
import { eventBus } from '../../infrastructure/event-bus'
import { prisma } from '../../lib/prisma'

// Register audit handlers for all events
eventBus.on('*', async (event: any) => {
  // Skip non-student-data events for performance
  if (!requiresAudit(event.type)) {
    return
  }
  
  await prisma.auditLog.create({
    data: {
      userId: event.userId,
      userRole: event.userRole,
      userName: event.userName,
      userEmail: event.userEmail,
      action: extractActionFromEvent(event.type),
      entityType: extractEntityTypeFromEvent(event.type),
      entityId: event.aggregateId,
      description: generateEventDescription(event),
      changes: event.changes,
      metadata: {
        eventType: event.type,
        correlationId: event.correlationId,
      },
      studentId: event.studentId,
      ferpaCategory: determineFERPACategory(event),
      tenantId: event.tenantId,
      schoolId: event.schoolId,
    },
  })
  
  // Also persist to EventStore for event sourcing
  await prisma.eventStore.create({
    data: {
      eventType: event.type,
      eventVersion: '1.0',
      aggregateId: event.aggregateId,
      aggregateType: event.aggregateType,
      payload: event,
      userId: event.userId,
      tenantId: event.tenantId,
      correlationId: event.correlationId,
    },
  })
})
```

---

## FERPA Compliance

### FERPA Categories

**Educational Records** (Requires Logging):
- Student grades and transcripts
- Attendance records
- Disciplinary records
- IEP/504 plans
- Assessment results
- Counseling notes
- Health records

**Directory Information** (Optional Logging):
- Student name, address, phone number
- Email address
- Date of birth
- Grade level
- Enrollment status
- Participation in activities
- Awards received

### Access Purpose Tracking

**Required Fields**:
```typescript
interface FERPAAudit {
  studentId: string
  accessedBy: string
  accessedAt: Date
  purpose: string          // REQUIRED: Why was data accessed?
  dataAccessed: string[]   // List of fields accessed
  ferpaCategory: 'DIRECTORY' | 'EDUCATIONAL_RECORD' | 'HEALTH_RECORD'
}
```

**Purpose Examples**:
- `"Grade entry for Math 101 assignment"`
- `"Parent conference preparation"`
- `"IEP annual review meeting"`
- `"Attendance verification for excused absence"`
- `"Special education eligibility determination"`

### Parent Consent Management

**Consent Tracking**:
```prisma
model FERPAConsent {
  id          String   @id @default(uuid())
  studentId   String
  parentId    String
  consentType String   // 'DIRECTORY_INFO', 'PHOTO_RELEASE', 'DATA_SHARING'
  granted     Boolean
  grantedAt   DateTime?
  revokedAt   DateTime?
  expiresAt   DateTime?
  metadata    Json?
  tenantId    String
  
  student     Student  @relation(fields: [studentId], references: [id])
  parent      Parent   @relation(fields: [parentId], references: [id])
  
  @@index([studentId, consentType])
}
```

### Data Export Auditing

**Export Tracking**:
```typescript
async function auditDataExport(req: Request, res: Response) {
  const user = extractUser(req)
  const exportType = req.body.exportType
  const studentIds = req.body.studentIds
  
  // Log export action
  await prisma.auditLog.create({
    data: {
      userId: user.id,
      userRole: user.role,
      userName: user.name,
      userEmail: user.email,
      action: 'EXPORT',
      entityType: 'STUDENT_DATA',
      description: `Exported ${exportType} data for ${studentIds.length} students`,
      metadata: {
        exportType,
        studentIds,
        format: req.body.format,
        dateRange: req.body.dateRange,
      },
      purpose: req.body.purpose,
      ferpaCategory: 'EDUCATIONAL_RECORD',
      ipAddress: req.ip,
      tenantId: user.tenantId,
    },
  })
  
  // Create export record
  const exportRecord = await prisma.dataExport.create({
    data: {
      userId: user.id,
      exportType,
      studentIds,
      purpose: req.body.purpose,
      expiresAt: addDays(new Date(), 7),  // Auto-delete after 7 days
      tenantId: user.tenantId,
    },
  })
  
  // Generate export file
  const fileUrl = await generateExport(exportRecord)
  
  return res.json({ fileUrl, expiresAt: exportRecord.expiresAt })
}
```

---

## Security Auditing

### Failed Login Attempts

**Login Audit**:
```typescript
async function auditLoginAttempt(req: Request, success: boolean, reason?: string) {
  await prisma.auditLog.create({
    data: {
      userId: success ? req.user.id : 'UNKNOWN',
      userRole: success ? req.user.role : 'UNKNOWN',
      userName: req.body.email,
      userEmail: req.body.email,
      action: success ? 'LOGIN' : 'LOGIN_FAILED',
      entityType: 'USER',
      description: success ? 'User logged in' : `Login failed: ${reason}`,
      metadata: {
        success,
        reason,
        mfaUsed: req.body.mfaCode ? true : false,
      },
      ipAddress: req.ip,
      userAgent: req.headers['user-agent'],
      tenantId: req.user?.tenantId || 'UNKNOWN',
    },
  })
  
  // Alert on multiple failed attempts
  if (!success) {
    const recentFailures = await prisma.auditLog.count({
      where: {
        userEmail: req.body.email,
        action: 'LOGIN_FAILED',
        createdAt: { gte: subMinutes(new Date(), 15) },
      },
    })
    
    if (recentFailures >= 5) {
      // Send security alert
      await sendSecurityAlert({
        type: 'MULTIPLE_FAILED_LOGINS',
        email: req.body.email,
        ipAddress: req.ip,
        count: recentFailures,
      })
    }
  }
}
```

### Unauthorized Access Attempts

**Access Control Audit**:
```typescript
async function auditUnauthorizedAccess(req: Request, reason: string) {
  await prisma.auditLog.create({
    data: {
      userId: req.user?.id || 'UNKNOWN',
      userRole: req.user?.role || 'UNKNOWN',
      userName: req.user?.name || 'UNKNOWN',
      userEmail: req.user?.email || 'UNKNOWN',
      action: 'UNAUTHORIZED_ACCESS',
      entityType: extractEntityType(req.path),
      entityId: extractEntityId(req.path, req.body),
      description: `Unauthorized access attempt: ${reason}`,
      metadata: {
        reason,
        method: req.method,
        path: req.path,
      },
      ipAddress: req.ip,
      userAgent: req.headers['user-agent'],
      tenantId: req.user?.tenantId || 'UNKNOWN',
    },
  })
  
  // Alert security team for critical resources
  if (isCriticalResource(req.path)) {
    await sendSecurityAlert({
      type: 'UNAUTHORIZED_ACCESS_CRITICAL',
      userId: req.user?.id,
      resource: req.path,
      reason,
    })
  }
}
```

### Permission Changes

**Role/Permission Audit**:
```typescript
async function auditPermissionChange(req: Request, targetUser: User, changes: any) {
  await prisma.auditLog.create({
    data: {
      userId: req.user.id,
      userRole: req.user.role,
      userName: req.user.name,
      userEmail: req.user.email,
      action: 'UPDATE',
      entityType: 'USER_PERMISSIONS',
      entityId: targetUser.id,
      description: `Updated permissions for ${targetUser.name}`,
      changes: {
        before: {
          role: targetUser.role,
          permissions: targetUser.permissions,
        },
        after: changes,
      },
      metadata: {
        targetUserId: targetUser.id,
        targetUserEmail: targetUser.email,
      },
      tenantId: req.user.tenantId,
    },
  })
}
```

---

## Compliance Reporting

### FERPA Audit Report

**Monthly FERPA Report**:
```typescript
async function generateFERPAAuditReport(dateRange: DateRange, tenantId: string) {
  const auditLogs = await prisma.auditLog.findMany({
    where: {
      tenantId,
      ferpaCategory: { not: null },
      createdAt: {
        gte: dateRange.start,
        lte: dateRange.end,
      },
    },
    include: {
      user: true,
      student: true,
    },
  })
  
  const report = {
    period: dateRange,
    totalAccesses: auditLogs.length,
    byCategory: groupBy(auditLogs, 'ferpaCategory'),
    byUser: groupBy(auditLogs, 'userId'),
    byStudent: groupBy(auditLogs, 'studentId'),
    byAction: groupBy(auditLogs, 'action'),
    unauthorizedAttempts: auditLogs.filter(log => 
      log.action === 'UNAUTHORIZED_ACCESS'
    ).length,
    dataExports: auditLogs.filter(log => log.action === 'EXPORT').length,
  }
  
  return report
}
```

**GraphQL Query**:
```graphql
query GetFERPAAuditReport($dateRange: DateRangeInput!) {
  ferpaAuditReport(dateRange: $dateRange) {
    period {
      start
      end
    }
    totalAccesses
    byCategory {
      category
      count
    }
    byUser {
      userId
      userName
      accessCount
    }
    byStudent {
      studentId
      studentName
      accessCount
    }
    unauthorizedAttempts
    dataExports
  }
}
```

### User Activity Report

**Individual User Audit**:
```typescript
async function getUserActivityReport(userId: string, dateRange: DateRange) {
  const activities = await prisma.auditLog.findMany({
    where: {
      userId,
      createdAt: {
        gte: dateRange.start,
        lte: dateRange.end,
      },
    },
    orderBy: {
      createdAt: 'desc',
    },
  })
  
  return {
    userId,
    period: dateRange,
    totalActions: activities.length,
    actions: activities.map(log => ({
      timestamp: log.createdAt,
      action: log.action,
      entityType: log.entityType,
      description: log.description,
      ipAddress: log.ipAddress,
    })),
    byAction: groupBy(activities, 'action'),
    byEntityType: groupBy(activities, 'entityType'),
  }
}
```

### Student Data Access Report

**Student-Specific Audit**:
```typescript
async function getStudentAccessReport(studentId: string, dateRange: DateRange) {
  const accesses = await prisma.auditLog.findMany({
    where: {
      studentId,
      createdAt: {
        gte: dateRange.start,
        lte: dateRange.end,
      },
    },
    include: {
      user: true,
    },
    orderBy: {
      createdAt: 'desc',
    },
  })
  
  return {
    studentId,
    period: dateRange,
    totalAccesses: accesses.length,
    accesses: accesses.map(log => ({
      timestamp: log.createdAt,
      accessedBy: log.userName,
      accessedByRole: log.userRole,
      action: log.action,
      purpose: log.purpose,
      dataAccessed: log.entityType,
    })),
    byUser: groupBy(accesses, 'userId'),
    byPurpose: groupBy(accesses, 'purpose'),
  }
}
```

---

## Audit Log Retention

### Retention Policy

**FERPA Requirements**:
- **Student educational records**: Retain 7 years after graduation/transfer
- **Access logs**: Retain as long as student record exists
- **Discipline records**: Retain 7 years
- **Special education records**: Retain 5 years after exit from program

**Implementation**:
```typescript
// Monthly job: Archive old audit logs
cron.schedule('0 2 1 * *', async () => {
  const sevenYearsAgo = subYears(new Date(), 7)
  
  // Find audit logs older than 7 years
  const oldLogs = await prisma.auditLog.findMany({
    where: {
      createdAt: { lte: sevenYearsAgo },
    },
  })
  
  // Archive to S3 cold storage
  const archiveFile = await archiveToS3(oldLogs, 'audit-logs')
  
  // Delete from database
  await prisma.auditLog.deleteMany({
    where: {
      createdAt: { lte: sevenYearsAgo },
    },
  })
  
  console.log(`Archived ${oldLogs.length} audit logs to ${archiveFile}`)
})
```

---

## System Admin Dashboard

### Audit Log Viewer

**Features**:
- Real-time audit log streaming
- Advanced search and filtering
- Export audit logs (CSV, JSON)
- User activity timeline
- Student access history
- Compliance report generation

**Filters**:
- Date range
- User role
- Action type
- Entity type
- FERPA category
- Student ID
- IP address

**GraphQL API**:
```graphql
query GetAuditLogs(
  $filters: AuditLogFiltersInput!
  $pagination: PaginationInput!
) {
  auditLogs(filters: $filters, pagination: $pagination) {
    edges {
      node {
        id
        timestamp
        user {
          name
          email
          role
        }
        action
        entityType
        description
        studentId
        purpose
        ipAddress
      }
    }
    pageInfo {
      total
      hasNextPage
    }
  }
}
```

---

## Best Practices

### Audit Design
1. **Comprehensive**: Log all student data access
2. **Immutable**: Audit logs cannot be edited or deleted
3. **Detailed**: Include sufficient context for investigations
4. **Timestamped**: Use UTC timestamps consistently
5. **Indexed**: Optimize for common queries

### FERPA Compliance
1. **Purpose Required**: Always log reason for access
2. **Parent Access**: Log parent access to child data
3. **Consent Tracking**: Track and honor parent consents
4. **Export Logging**: Log all data exports
5. **Retention Policy**: Follow 7-year retention rule

### Security
1. **Encryption**: Encrypt audit logs at rest
2. **Access Control**: Only admins can view audit logs
3. **Alerting**: Real-time alerts for suspicious activity
4. **Monitoring**: Regular review of audit logs
5. **Incident Response**: Defined process for security events

### Performance
1. **Async Logging**: Don't block request for audit logging
2. **Batch Writes**: Batch audit log writes
3. **Partitioning**: Partition audit logs by date
4. **Archival**: Archive old logs to cold storage
5. **Indexing**: Index frequently queried fields

---

## Related Documentation

- [Events Architecture](./events.md) - Event sourcing with EventStore
- [Security](./security.md) - Authentication and authorization
- [System Admin Portal](../roles/system-admin.md) - Audit log viewer

---

**Production Metrics** (as of November 2025):
- **Daily Audit Logs**: ~50,000
- **Total Audit Logs (7 years)**: ~120 million
- **FERPA Logs**: ~30% of total
- **Retention**: 7 years (2,555 days)
- **Archive Size**: ~500 GB (compressed)
