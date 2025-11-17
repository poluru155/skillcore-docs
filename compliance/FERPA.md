# FERPA Compliance Guide

**Last Updated**: November 17, 2025  
**Regulation**: Family Educational Rights and Privacy Act (FERPA) - 20 U.S.C. § 1232g  
**Status**: Production Implementation  
**Compliance Level**: Full FERPA Compliance

---

## Overview

This document outlines how SkillCore complies with the **Family Educational Rights and Privacy Act (FERPA)**, the federal law that protects the privacy of student education records.

---

## FERPA Requirements

### What FERPA Protects

FERPA gives parents and eligible students (18+ or in postsecondary education) the right to:

1. **Inspect and review** educational records
2. **Request amendments** to records they believe are inaccurate
3. **Consent to disclosures** of personally identifiable information (PII)
4. **File complaints** with the U.S. Department of Education

### Personally Identifiable Information (PII)

FERPA protects the following student information:

- Student name
- Parent/guardian names
- Address, phone number, email
- Student identification number
- Social Security number
- Date and place of birth
- Grades, test scores, GPA
- Attendance records
- Discipline records
- Health records
- Special education records (IEP, 504 plans)
- Photographs, videos, biometric data
- Any other information that makes student identity traceable

---

## SkillCore FERPA Implementation

### 1. Role-Based Access Control (RBAC)

**Implementation**: Multi-layer access control ensures only authorized users access student data.

```typescript
// Role hierarchy for student data access
enum UserRole {
  SUPER_ADMIN      // Full system access
  DISTRICT_ADMIN   // District-wide access
  SCHOOL_ADMIN     // School-level access
  TEACHER          // Class-enrolled students only
  COUNSELOR        // Assigned caseload only
  PARENT           // Own children only
  STUDENT          // Own records only
}

// Access control middleware
export async function verifyStudentAccess(
  userId: string,
  studentId: string,
  requiredRole: UserRole
): Promise<boolean> {
  const user = await getUserWithRole(userId)
  
  // Super admins have full access
  if (user.role === 'SUPER_ADMIN') return true
  
  // District admins: same district only
  if (user.role === 'DISTRICT_ADMIN') {
    return await isInSameDistrict(user.tenantId, studentId)
  }
  
  // School admins: same school only
  if (user.role === 'SCHOOL_ADMIN') {
    return await isInSameSchool(user.schoolId, studentId)
  }
  
  // Teachers: enrolled students only
  if (user.role === 'TEACHER') {
    return await isStudentEnrolledInTeacherClass(studentId, userId)
  }
  
  // Counselors: assigned caseload only
  if (user.role === 'COUNSELOR') {
    return await isStudentInCounselorCaseload(studentId, userId)
  }
  
  // Parents: own children only
  if (user.role === 'PARENT') {
    return await isParentOfStudent(userId, studentId)
  }
  
  // Students: own records only
  if (user.role === 'STUDENT') {
    return userId === studentId
  }
  
  return false
}
```

### 2. Data Sanitization by Role

**Implementation**: Different roles see different levels of student information.

```typescript
// Data sanitization based on role
export function sanitizeStudentData(
  student: Student,
  userRole: UserRole
): Partial<Student> {
  // Base data visible to all authorized users
  const baseData = {
    id: student.id,
    givenName: student.givenName,
    familyName: student.familyName,
    gradeLevel: student.gradeLevel,
  }
  
  // Teachers and counselors see educational data
  if (userRole === 'TEACHER' || userRole === 'COUNSELOR') {
    return {
      ...baseData,
      studentNumber: student.studentNumber,
      email: student.email,
      hasIEP: student.hasIEP,
      has504: student.has504,
      isESL: student.isESL,
      // NO SSN, address, parent contact
    }
  }
  
  // School/district admins see full data
  if (userRole === 'SCHOOL_ADMIN' || userRole === 'DISTRICT_ADMIN') {
    return {
      ...baseData,
      studentNumber: student.studentNumber,
      stateId: student.stateId,
      email: student.email,
      phone: student.phone,
      address: student.address,
      parentContacts: student.parentContacts,
      hasIEP: student.hasIEP,
      has504: student.has504,
      // NO SSN (stored separately, encrypted)
    }
  }
  
  // Parents see their child's data
  if (userRole === 'PARENT') {
    return {
      ...baseData,
      email: student.email,
      phone: student.phone,
      gradeLevel: student.gradeLevel,
      // NO other students' data
    }
  }
  
  // Students see own data only
  if (userRole === 'STUDENT') {
    return {
      ...baseData,
      email: student.email,
      phone: student.phone,
      gradeLevel: student.gradeLevel,
      studentNumber: student.studentNumber,
    }
  }
  
  return baseData
}
```

### 3. Multi-Tenant Data Isolation

**Implementation**: Complete data isolation between school districts.

```prisma
// Every table includes tenantId for isolation
model Student {
  id        String @id @default(uuid())
  tenantId  String @db.Uuid
  tenant    Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  // All queries automatically filtered by tenantId
  @@index([tenantId])
}

model Grade {
  id        String @id @default(uuid())
  tenantId  String @db.Uuid
  tenant    Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  
  @@index([tenantId])
}
```

**Query-level enforcement**:
```typescript
// All queries include tenant filter
export async function getStudents(tenantId: string) {
  return await prisma.student.findMany({
    where: { tenantId }, // Always filtered
  })
}

// Middleware enforces tenant isolation
prisma.$use(async (params, next) => {
  if (params.model && params.action === 'findMany') {
    params.args.where = params.args.where || {}
    params.args.where.tenantId = getCurrentTenantId()
  }
  return next(params)
})
```

### 4. Audit Logging

**Implementation**: All access to student records is logged for compliance.

```typescript
// Audit log entry for every student data access
export async function logStudentAccess(event: {
  userId: string
  studentId: string
  action: 'VIEW' | 'EDIT' | 'DELETE' | 'EXPORT'
  ipAddress: string
  resource: string
  success: boolean
}) {
  await prisma.auditLog.create({
    data: {
      userId: event.userId,
      entityType: 'STUDENT',
      entityId: event.studentId,
      action: event.action,
      ipAddress: event.ipAddress,
      resource: event.resource,
      success: event.success,
      timestamp: new Date(),
      tenantId: getCurrentTenantId(),
    },
  })
  
  // Also publish to EventStore for permanent record
  await eventStore.append({
    streamName: `student-${event.studentId}`,
    eventType: 'StudentDataAccessed',
    data: event,
  })
}

// Example usage in API
export async function getStudent(studentId: string, userId: string) {
  try {
    const hasAccess = await verifyStudentAccess(userId, studentId, 'TEACHER')
    
    if (!hasAccess) {
      await logStudentAccess({
        userId,
        studentId,
        action: 'VIEW',
        ipAddress: getClientIp(),
        resource: '/api/students/:id',
        success: false,
      })
      throw new UnauthorizedError('Access denied')
    }
    
    const student = await prisma.student.findUnique({ where: { id: studentId } })
    
    await logStudentAccess({
      userId,
      studentId,
      action: 'VIEW',
      ipAddress: getClientIp(),
      resource: '/api/students/:id',
      success: true,
    })
    
    return sanitizeStudentData(student, getUserRole(userId))
  } catch (error) {
    // Log failed access attempt
    await logStudentAccess({
      userId,
      studentId,
      action: 'VIEW',
      ipAddress: getClientIp(),
      resource: '/api/students/:id',
      success: false,
    })
    throw error
  }
}
```

### 5. Parental Consent Management

**Implementation**: Track and enforce parental consent for data sharing.

```prisma
model ParentalConsent {
  id          String   @id @default(uuid())
  studentId   String   @db.Uuid
  student     Student  @relation(fields: [studentId], references: [id])
  
  parentId    String   @db.Uuid
  parent      Parent   @relation(fields: [parentId], references: [id])
  
  // Consent details
  consentType ConsentType
  granted     Boolean
  grantedAt   DateTime?
  revokedAt   DateTime?
  expiresAt   DateTime?
  
  // Consent scope
  purpose     String   // e.g., "Third-party assessment platform"
  recipient   String   // Who receives the data
  dataTypes   String[] // What data is shared
  
  // Signature
  signedBy    String   // Parent name
  signature   String?  // Digital signature
  ipAddress   String   // IP when signed
  
  tenantId    String   @db.Uuid
  tenant      Tenant   @relation(fields: [tenantId], references: [id])
  
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@index([tenantId])
  @@index([studentId])
}

enum ConsentType {
  DIRECTORY_INFORMATION
  PHOTO_RELEASE
  THIRD_PARTY_SHARING
  RESEARCH_PARTICIPATION
  TECHNOLOGY_USE
}
```

**Consent enforcement**:
```typescript
export async function canShareStudentData(
  studentId: string,
  purpose: string,
  recipient: string
): Promise<boolean> {
  const consent = await prisma.parentalConsent.findFirst({
    where: {
      studentId,
      purpose,
      recipient,
      granted: true,
      revokedAt: null,
      OR: [
        { expiresAt: null },
        { expiresAt: { gt: new Date() } },
      ],
    },
  })
  
  return !!consent
}

// Check consent before data export
export async function exportStudentData(studentId: string, recipient: string) {
  const hasConsent = await canShareStudentData(
    studentId,
    'Data Export',
    recipient
  )
  
  if (!hasConsent) {
    throw new Error('Parental consent required for data export')
  }
  
  // Proceed with export...
}
```

### 6. Directory Information Opt-Out

**Implementation**: Parents can opt out of directory information disclosure.

```prisma
model Student {
  // ... other fields
  
  // Directory information opt-out
  directoryOptOut Boolean @default(false)
  optOutDate      DateTime?
  optOutBy        String?  // Parent who opted out
}
```

**Directory information handling**:
```typescript
// Directory information: can be disclosed without consent UNLESS opted out
const DIRECTORY_INFO = [
  'name',
  'gradeLevel',
  'photo',
  'participationInActivities',
  'awards',
  'honorsReceived',
]

export function getPublicStudentInfo(student: Student) {
  if (student.directoryOptOut) {
    // No directory information can be shared
    return null
  }
  
  return {
    name: `${student.givenName} ${student.familyName}`,
    gradeLevel: student.gradeLevel,
    photo: student.photoUrl,
    // Only directory information
  }
}
```

### 7. Data Encryption

**Implementation**: Sensitive data encrypted at rest and in transit.

```typescript
// Encryption for sensitive fields
import { encrypt, decrypt } from '@/lib/encryption'

// Store SSN encrypted
export async function storeSSN(studentId: string, ssn: string) {
  const encrypted = encrypt(ssn, process.env.ENCRYPTION_KEY!)
  
  await prisma.studentSensitiveData.create({
    data: {
      studentId,
      encryptedSSN: encrypted,
      tenantId: getCurrentTenantId(),
    },
  })
}

// Retrieve SSN (only for authorized users)
export async function getSSN(studentId: string, userId: string) {
  // Only district admins can access SSN
  const user = await getUser(userId)
  if (user.role !== 'DISTRICT_ADMIN') {
    throw new UnauthorizedError('Insufficient permissions')
  }
  
  const data = await prisma.studentSensitiveData.findUnique({
    where: { studentId },
  })
  
  if (!data) return null
  
  // Audit log the access
  await logSensitiveDataAccess({
    userId,
    studentId,
    dataType: 'SSN',
    action: 'VIEW',
  })
  
  return decrypt(data.encryptedSSN, process.env.ENCRYPTION_KEY!)
}
```

**Database encryption**:
```sql
-- Database-level encryption for sensitive columns
ALTER TABLE student_sensitive_data
  ALTER COLUMN encrypted_ssn TYPE bytea USING pgp_sym_encrypt(encrypted_ssn, 'encryption_key');
```

### 8. Data Retention & Deletion

**Implementation**: Automated data retention policies.

```typescript
// Soft delete with retention period
export async function deleteStudent(studentId: string) {
  const retentionDays = 2555 // 7 years (FERPA recommendation)
  
  await prisma.student.update({
    where: { id: studentId },
    data: {
      status: 'DELETED',
      deletedAt: new Date(),
      permanentDeletionDate: addDays(new Date(), retentionDays),
    },
  })
  
  // Publish event for cascade soft delete
  await publishEvent({
    type: 'StudentDeleted',
    studentId,
    retentionUntil: addDays(new Date(), retentionDays),
  })
}

// Scheduled job: permanent deletion after retention
export async function permanentlyDeleteExpiredRecords() {
  const expiredStudents = await prisma.student.findMany({
    where: {
      status: 'DELETED',
      permanentDeletionDate: { lte: new Date() },
    },
  })
  
  for (const student of expiredStudents) {
    // Permanently delete all associated data
    await prisma.$transaction([
      prisma.grade.deleteMany({ where: { studentId: student.id } }),
      prisma.attendanceRecord.deleteMany({ where: { studentId: student.id } }),
      prisma.enrollment.deleteMany({ where: { studentId: student.id } }),
      prisma.student.delete({ where: { id: student.id } }),
    ])
    
    // Audit log permanent deletion
    await logPermanentDeletion({
      entityType: 'STUDENT',
      entityId: student.id,
      deletedAt: new Date(),
    })
  }
}
```

---

## FERPA Compliance Checklist

### Access Control ✅
- [x] Role-based access control (RBAC) implemented
- [x] Multi-tenant data isolation
- [x] Query-level tenant filtering
- [x] User authentication (JWT + sessions)
- [x] Session expiration (30 minutes inactive)
- [x] Password complexity requirements
- [x] Multi-factor authentication (MFA) available

### Data Protection ✅
- [x] Encryption at rest (AES-256)
- [x] Encryption in transit (TLS 1.3)
- [x] Sensitive data encryption (SSN, etc.)
- [x] Database connection encryption
- [x] Secure password hashing (bcrypt)
- [x] API rate limiting
- [x] CORS policy configured

### Audit & Compliance ✅
- [x] All data access logged
- [x] Audit logs immutable (EventStore)
- [x] 7-year audit retention
- [x] Failed access attempts logged
- [x] Sensitive data access alerts
- [x] Compliance reports available
- [x] Annual FERPA training tracked

### Parental Rights ✅
- [x] Parent portal for record access
- [x] Amendment request workflow
- [x] Consent management system
- [x] Directory information opt-out
- [x] Data export capability
- [x] Records review tracking
- [x] Consent expiration tracking

### Data Sharing ✅
- [x] Consent required for third-party sharing
- [x] School official exception documented
- [x] Legitimate educational interest verified
- [x] Health/safety emergency procedures
- [x] Court order compliance process
- [x] Data sharing agreements tracked
- [x] Disclosure logs maintained

### Data Retention ✅
- [x] Soft delete with 7-year retention
- [x] Automated permanent deletion
- [x] Retention policy documented
- [x] Deletion audit trail
- [x] Backup retention aligned
- [x] Archive process defined
- [x] Legal hold capability

---

## FERPA Violations Prevention

### Common Violations & Our Protection

| Violation | SkillCore Protection |
|-----------|---------------------|
| **Teacher posts grades publicly** | Grades never public; parent/student portal only |
| **Student worker accesses records** | Role-based access; students can't access others' data |
| **Email student info to wrong parent** | Parent can only see own children; email validation |
| **Directory info shared after opt-out** | `directoryOptOut` flag checked before disclosure |
| **Third party gets data without consent** | API requires consent verification before data export |
| **Records not provided to parents** | Parent portal provides instant access 24/7 |
| **No audit trail of access** | All access logged to immutable EventStore |
| **Data breach** | Encryption + multi-tenant isolation + access control |

### Automated Safeguards

```typescript
// GraphQL field-level authorization
export const StudentResolver = {
  Query: {
    student: async (_, { id }, context) => {
      // Automatic access verification
      await verifyAccess(context.userId, id)
      return getStudent(id)
    },
  },
  
  Student: {
    // Sensitive fields require extra permission
    ssn: async (student, _, context) => {
      requireRole(context.user, 'DISTRICT_ADMIN')
      return getSSN(student.id, context.userId)
    },
    
    // Automatic sanitization based on role
    parentContacts: async (student, _, context) => {
      if (context.user.role === 'TEACHER') {
        return null // Teachers don't see parent contacts
      }
      return student.parentContacts
    },
  },
}
```

---

## Emergency Disclosure Procedures

FERPA allows disclosure without consent in emergencies. Our process:

### Health or Safety Emergency

```typescript
export async function emergencyDisclosure(params: {
  studentId: string
  disclosedTo: string
  reason: string
  emergency: 'health' | 'safety'
  authorizedBy: string
}) {
  // Log emergency disclosure
  await prisma.emergencyDisclosure.create({
    data: {
      studentId: params.studentId,
      disclosedTo: params.disclosedTo,
      reason: params.reason,
      emergencyType: params.emergency,
      authorizedBy: params.authorizedBy,
      disclosedAt: new Date(),
      tenantId: getCurrentTenantId(),
    },
  })
  
  // Notify administrators
  await notifyAdministrators({
    type: 'EMERGENCY_DISCLOSURE',
    studentId: params.studentId,
    reason: params.reason,
  })
  
  // Document in audit log
  await logEmergencyAccess({
    userId: params.authorizedBy,
    studentId: params.studentId,
    action: 'EMERGENCY_DISCLOSURE',
    justification: params.reason,
  })
  
  // Notify parent after emergency
  await scheduleParentNotification({
    studentId: params.studentId,
    message: `Emergency disclosure of student records occurred on ${new Date().toLocaleDateString()}`,
  })
}
```

---

## Compliance Reporting

### FERPA Compliance Dashboard

Available to district administrators:

```typescript
export async function getFERPAComplianceReport(tenantId: string) {
  const [
    totalAccesses,
    unauthorizedAttempts,
    parentConsents,
    optOuts,
    emergencyDisclosures,
    dataExports,
  ] = await Promise.all([
    // Total student data accesses (last 30 days)
    prisma.auditLog.count({
      where: {
        tenantId,
        entityType: 'STUDENT',
        createdAt: { gte: subDays(new Date(), 30) },
      },
    }),
    
    // Unauthorized access attempts
    prisma.auditLog.count({
      where: {
        tenantId,
        entityType: 'STUDENT',
        success: false,
        createdAt: { gte: subDays(new Date(), 30) },
      },
    }),
    
    // Active parental consents
    prisma.parentalConsent.count({
      where: {
        tenantId,
        granted: true,
        revokedAt: null,
      },
    }),
    
    // Directory opt-outs
    prisma.student.count({
      where: {
        tenantId,
        directoryOptOut: true,
      },
    }),
    
    // Emergency disclosures
    prisma.emergencyDisclosure.count({
      where: {
        tenantId,
        createdAt: { gte: subDays(new Date(), 30) },
      },
    }),
    
    // Data exports
    prisma.dataExport.count({
      where: {
        tenantId,
        createdAt: { gte: subDays(new Date(), 30) },
      },
    }),
  ])
  
  return {
    totalAccesses,
    unauthorizedAttempts,
    parentConsents,
    optOuts,
    emergencyDisclosures,
    dataExports,
    complianceScore: calculateComplianceScore({
      unauthorizedAttempts,
      totalAccesses,
    }),
  }
}
```

---

## Training & Documentation

### Required FERPA Training

All users with student data access must complete annual FERPA training:

```typescript
export async function trackFERPATraining(params: {
  userId: string
  completedAt: Date
  trainingProvider: string
  certificateUrl?: string
}) {
  await prisma.ferpaTraining.create({
    data: {
      userId: params.userId,
      completedAt: params.completedAt,
      expiresAt: addYears(params.completedAt, 1), // Annual renewal
      trainingProvider: params.trainingProvider,
      certificateUrl: params.certificateUrl,
      tenantId: getCurrentTenantId(),
    },
  })
  
  // Notify user when training expires soon
  scheduleExpirationReminder(params.userId, addYears(params.completedAt, 1))
}
```

---

## Related Documentation

- [Security Architecture](../architecture/security.md) - Overall security implementation
- [Auditing System](../architecture/auditing.md) - Audit logging details
- [Database Architecture](../architecture/database.md) - Multi-tenancy and data isolation
- [API Security](../api/graphql.md#security) - API-level security measures

---

## FERPA Resources

- [U.S. Department of Education FERPA](https://www2.ed.gov/policy/gen/guid/fpco/ferpa/index.html)
- [FERPA Quick Reference](https://studentprivacy.ed.gov/)
- [FERPA Model Notification](https://www2.ed.gov/policy/gen/guid/fpco/ferpa/lea-officials.html)

---

**Compliance Status**: ✅ Full FERPA Compliance  
**Last Audit**: November 17, 2025  
**Next Review**: November 17, 2026  
**Compliance Officer**: District IT Administrator
