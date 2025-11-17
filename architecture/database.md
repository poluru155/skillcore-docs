# Database Architecture

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Technology**: PostgreSQL 15 · Prisma ORM 6.2 · Row-Level Security

---

## Overview

SkillCore uses **PostgreSQL 15** with **Prisma ORM** for type-safe database access. The schema follows **OneRoster 1.3** standards with extensions for educational workflows, implementing **DDD bounded contexts**, **multi-tenancy**, **FERPA compliance**, and **soft deletes**.

---

## Database Architecture

```
Application Layer (Prisma Client)
         ↓
Prisma Middleware (Tenant isolation, Soft deletes, Audit logging)
         ↓
PostgreSQL Database
         ↓
Row-Level Security (RLS) + Indexes + Constraints
```

---

## Schema Organization

### Bounded Contexts (DDD)

The schema is organized into **8 bounded contexts**:

1. **Rostering Context** (~15 tables)
   - Organizations (districts, schools, boards)
   - Academic sessions (school years, terms, grading periods)
   - Courses (subject, grade level)
   - Classes/Sections
   - Users (students, teachers, parents, admin)
   - Enrollments (student-class relationships)

2. **Gradebook Context** (~8 tables)
   - Categories (homework, tests, projects)
   - Score scales (percentage, letter grades, standards)
   - Line items (assignments, assessments)
   - Results (grades, scores)
   - Grade history

3. **Attendance Context** (~5 tables)
   - Attendance records
   - Attendance codes
   - Daily aggregations
   - Absence patterns

4. **Assessment Context** (~12 tables)
   - Tests/Quizzes
   - Questions (multiple choice, essay, etc.)
   - Question banks
   - Test submissions
   - Auto-grading results

5. **Special Programs Context** (~10 tables)
   - IEP (Individualized Education Program)
   - Section 504 plans
   - ESL (English as Second Language)
   - Gifted programs

6. **Intervention Context** (~8 tables)
   - Interventions (MTSS/RTI)
   - Intervention goals
   - Progress monitoring
   - Team meetings

7. **Communication Context** (~8 tables)
   - Messages (teacher-parent, teacher-student)
   - Announcements
   - Conversations
   - Conferences

8. **Analytics Context** (~6 tables)
   - Student analytics
   - Class performance
   - Trend data
   - Predictive models

---

## Multi-Tenancy Design

### Tenant Isolation Strategy

**Denormalized Tenant Fields**:
```prisma
model Student {
  id          String   @id @default(uuid())
  
  // Multi-tenant fields (denormalized for performance)
  districtId  String   // District org sourcedId
  schoolId    String?  // School org sourcedId (optional for district-level users)
  tenantId    String   // Alias for districtId (legacy compatibility)
  
  // ... other fields
  
  @@index([districtId])
  @@index([schoolId])
  @@index([tenantId])
}
```

**Why Denormalization?**:
- **Performance**: Single-table queries without joins
- **Simplicity**: Clear tenant boundaries
- **Security**: Easy RLS policy implementation
- **Trade-off**: Slight data duplication vs. query performance

**Tenant Context Middleware**:
```typescript
// Automatic tenant filtering
prisma.$use(async (params, next) => {
  const tenantScopedModels = [
    'Student', 'Teacher', 'Class', 'Grade',
    'Attendance', 'IEP', 'Section504', 'Intervention',
  ]
  
  if (tenantScopedModels.includes(params.model)) {
    if (params.action === 'findMany' || params.action === 'findFirst') {
      params.args.where = {
        ...params.args.where,
        tenantId: currentTenantId,
      }
    }
  }
  
  return next(params)
})
```

### Organization Hierarchy

**Org Structure**:
```
Board (type: 'board')
  └── District (type: 'district') [districtId = sourcedId]
        ├── School A (type: 'school', parentId = district.id)
        ├── School B (type: 'school', parentId = district.id)
        └── School C (type: 'school', parentId = district.id)
```

**Org Model**:
```prisma
model Org {
  sourcedId         String       @id
  type              OrgType      // 'board', 'district', 'school', 'department'
  name              String
  identifier        String?      // External system ID
  
  // Hierarchy
  parentId          String?
  parent            Org?         @relation("OrgHierarchy", fields: [parentId], references: [sourcedId])
  children          Org[]        @relation("OrgHierarchy")
  
  // Multi-tenant
  districtId        String       // Self-referencing for districts, parent district for schools
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime     @default(now())
  updatedAt         DateTime     @updatedAt
  
  @@index([districtId])
  @@index([type])
  @@index([parentId])
}
```

---

## FERPA Compliance

### Soft Delete Pattern

**Never Hard Delete Student Data**:
```prisma
model Student {
  id          String    @id @default(uuid())
  // ... student fields
  
  // Soft delete (FERPA requirement: 7-year retention)
  deletedAt   DateTime?
  deletedBy   String?
  
  @@index([deletedAt])
}

// Prisma middleware for soft deletes
prisma.$use(async (params, next) => {
  if (params.action === 'delete') {
    params.action = 'update'
    params.args.data = {
      deletedAt: new Date(),
      deletedBy: currentUserId,
    }
  }
  
  if (params.action === 'deleteMany') {
    params.action = 'updateMany'
    params.args.data = {
      deletedAt: new Date(),
      deletedBy: currentUserId,
    }
  }
  
  // Exclude soft-deleted records from queries
  if (params.action === 'findMany' || params.action === 'findFirst') {
    params.args.where = {
      ...params.args.where,
      deletedAt: null,
    }
  }
  
  return next(params)
})
```

### Audit Logging

**Audit Triggers** (PostgreSQL):
```sql
-- Audit log table
CREATE TABLE audit_log (
  id SERIAL PRIMARY KEY,
  table_name TEXT NOT NULL,
  record_id TEXT NOT NULL,
  operation TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
  old_data JSONB,
  new_data JSONB,
  changed_by TEXT,
  changed_at TIMESTAMP DEFAULT NOW()
);

-- Audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO audit_log (table_name, record_id, operation, new_data, changed_by)
    VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', row_to_json(NEW), current_user);
    RETURN NEW;
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO audit_log (table_name, record_id, operation, old_data, new_data, changed_by)
    VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), current_user);
    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_log (table_name, record_id, operation, old_data, changed_by)
    VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', row_to_json(OLD), current_user);
    RETURN OLD;
  END IF;
END;
$$ LANGUAGE plpgsql;

-- Apply to student data tables
CREATE TRIGGER audit_trigger_students
  AFTER INSERT OR UPDATE OR DELETE ON students
  FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

CREATE TRIGGER audit_trigger_grades
  AFTER INSERT OR UPDATE OR DELETE ON grades
  FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

---

## Key Tables & Relationships

### Rostering Context

**Student Table**:
```prisma
model Student {
  id                String      @id @default(uuid())
  
  // OneRoster fields
  sourcedId         String      @unique
  status            EntityStatus @default(active)
  givenName         String
  familyName        String
  middleName        String?
  identifier        String?     // Student number
  email             String?
  sms               String?     // Mobile number
  phone             String?     // Home phone
  
  // Demographics
  gradeLevel        String      // K-12
  dateOfBirth       DateTime?
  gender            String?
  
  // Multi-tenant
  districtId        String
  schoolId          String
  
  // Relationships
  enrollments       Enrollment[]
  grades            Result[]
  attendance        AttendanceRecord[]
  ieps              IEP[]
  section504s       Section504[]
  interventions     Intervention[]
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt
  
  @@index([districtId])
  @@index([schoolId])
  @@index([sourcedId])
  @@index([identifier])
  @@index([deletedAt])
}
```

**Class/Section Table**:
```prisma
model Class {
  id                String      @id @default(uuid())
  
  // OneRoster fields
  sourcedId         String      @unique
  status            EntityStatus @default(active)
  title             String      // "Algebra I - Period 3"
  classCode         String?     // "ALG1-03"
  classType         ClassType   @default(homeroom)
  
  // Relationships
  courseId          String
  course            Course      @relation(fields: [courseId], references: [id])
  schoolId          String
  school            Org         @relation(fields: [schoolId], references: [sourcedId])
  academicSessionId String
  academicSession   AcademicSession @relation(fields: [academicSessionId], references: [id])
  
  // Teachers
  primaryTeacherId  String?
  primaryTeacher    User?       @relation("PrimaryTeacher", fields: [primaryTeacherId], references: [id])
  coTeachers        User[]      @relation("CoTeachers")
  
  // Students
  enrollments       Enrollment[]
  
  // Gradebook
  categories        Category[]
  lineItems         LineItem[]
  
  // Multi-tenant
  districtId        String
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt
  
  @@index([districtId])
  @@index([schoolId])
  @@index([courseId])
  @@index([academicSessionId])
}
```

### Gradebook Context

**LineItem (Assignment) Table**:
```prisma
model LineItem {
  id                String      @id @default(uuid())
  
  // OneRoster fields
  sourcedId         String      @unique
  status            EntityStatus @default(active)
  title             String      // "Chapter 5 Quiz"
  description       String?
  assignDate        DateTime?
  dueDate           DateTime?
  
  // Grading
  categoryId        String
  category          Category    @relation(fields: [categoryId], references: [id])
  resultValueMin    Float       @default(0)
  resultValueMax    Float       // Points possible
  
  // Relationships
  classId           String
  class             Class       @relation(fields: [classId], references: [id])
  results           Result[]    // Student grades
  
  // Multi-tenant
  districtId        String
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt
  
  @@index([classId])
  @@index([categoryId])
  @@index([dueDate])
}
```

**Result (Grade) Table**:
```prisma
model Result {
  id                String      @id @default(uuid())
  
  // OneRoster fields
  sourcedId         String      @unique
  status            EntityStatus @default(active)
  score             Float       // Numeric score
  scoreStatus       ScoreStatus @default(not_submitted)
  comment           String?     // Teacher feedback
  
  // Relationships
  lineItemId        String
  lineItem          LineItem    @relation(fields: [lineItemId], references: [id])
  studentId         String
  student           Student     @relation(fields: [studentId], references: [id])
  gradedById        String?
  gradedBy          User?       @relation(fields: [gradedById], references: [id])
  
  // Versioning (for grade changes)
  version           Int         @default(1)
  previousVersionId String?
  previousVersion   Result?     @relation("GradeHistory", fields: [previousVersionId], references: [id])
  nextVersions      Result[]    @relation("GradeHistory")
  
  // Timestamps
  postedAt          DateTime?   // When grade was released to student
  gradedAt          DateTime?   // When teacher entered grade
  
  // Multi-tenant
  districtId        String
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt
  
  @@index([lineItemId])
  @@index([studentId])
  @@index([districtId])
  @@unique([lineItemId, studentId, version])
}
```

### Attendance Context

**AttendanceRecord Table**:
```prisma
model AttendanceRecord {
  id                String            @id @default(uuid())
  
  // Core fields
  studentId         String
  student           Student           @relation(fields: [studentId], references: [id])
  date              DateTime          @db.Date
  status            AttendanceStatus  // present, absent, tardy, etc.
  periodId          String?           // For period-by-period attendance
  
  // Details
  markedById        String
  markedBy          User              @relation(fields: [markedById], references: [id])
  markedAt          DateTime          @default(now())
  notes             String?
  reason            String?           // For absences
  
  // Multi-tenant
  districtId        String
  schoolId          String
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime          @default(now())
  updatedAt         DateTime          @updatedAt
  
  @@unique([studentId, date, periodId])
  @@index([studentId, date])
  @@index([districtId])
  @@index([date])
}
```

---

## Performance Optimization

### Indexes

**Critical Indexes**:
```prisma
// Multi-tenant queries
@@index([districtId])
@@index([schoolId])
@@index([tenantId])

// Soft delete filtering
@@index([deletedAt])

// Foreign key lookups
@@index([studentId])
@@index([classId])
@@index([teacherId])

// Date range queries
@@index([date])
@@index([dueDate])
@@index([createdAt])

// Composite indexes for common queries
@@index([studentId, classId])
@@index([districtId, schoolId])
@@index([classId, dueDate])
```

**Index Usage**:
```sql
-- Query: Get all grades for student in a class
EXPLAIN ANALYZE
SELECT * FROM results
WHERE student_id = 'student-123'
  AND line_item_id IN (
    SELECT id FROM line_items WHERE class_id = 'class-456'
  );
-- Uses index: results(student_id, line_item_id)

-- Query: Get attendance for student in date range
EXPLAIN ANALYZE
SELECT * FROM attendance_records
WHERE student_id = 'student-123'
  AND date BETWEEN '2025-01-01' AND '2025-12-31';
-- Uses index: attendance_records(student_id, date)
```

### Connection Pooling

**Prisma Connection Pool**:
```typescript
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Connection pool configuration
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=20&pool_timeout=10"
```

**PgBouncer** (Production):
```ini
[databases]
skillcore = host=localhost port=5432 dbname=skillcore

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
reserve_pool_timeout = 3
```

---

## Migrations

### Migration Strategy

**Development**:
```bash
# Create migration
pnpm prisma migrate dev --name add_intervention_goals

# Reset database (dev only)
pnpm prisma migrate reset

# Generate Prisma Client
pnpm prisma generate
```

**Production**:
```bash
# Deploy migrations
pnpm prisma migrate deploy

# Verify migration
pnpm prisma migrate status
```

### Migration Best Practices

1. **Never Edit Migrations**: Migrations are immutable once applied
2. **Test Migrations**: Test on staging before production
3. **Data Migrations**: Separate schema and data migrations
4. **Backwards Compatibility**: Avoid breaking changes
5. **Rollback Plan**: Always have rollback procedure

**Example Migration**:
```sql
-- Migration: 20251117_add_intervention_goals
-- Description: Add goals table for intervention tracking

-- Add goals table
CREATE TABLE intervention_goals (
  id TEXT PRIMARY KEY,
  intervention_id TEXT NOT NULL REFERENCES interventions(id),
  description TEXT NOT NULL,
  target_date DATE,
  status TEXT NOT NULL DEFAULT 'active',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Add index
CREATE INDEX idx_intervention_goals_intervention_id ON intervention_goals(intervention_id);

-- Add audit trigger
CREATE TRIGGER audit_trigger_intervention_goals
  AFTER INSERT OR UPDATE OR DELETE ON intervention_goals
  FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

---

## Database Backups

### Backup Strategy

**Automated Backups**:
- **Frequency**: Daily at 2:00 AM UTC
- **Retention**: 30 days rolling
- **Location**: AWS S3 (encrypted)
- **Type**: Full database dump + WAL archiving

**Backup Script**:
```bash
#!/bin/bash
# daily-backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="skillcore_backup_${DATE}.sql.gz"

# Dump database
pg_dump $DATABASE_URL | gzip > /tmp/$BACKUP_FILE

# Upload to S3
aws s3 cp /tmp/$BACKUP_FILE s3://skillcore-backups/production/$BACKUP_FILE

# Cleanup old backups (keep last 30 days)
aws s3 ls s3://skillcore-backups/production/ | \
  while read -r line; do
    createDate=$(echo $line | awk '{print $1" "$2}')
    createDate=$(date -d "$createDate" +%s)
    olderThan=$(date -d "30 days ago" +%s)
    if [[ $createDate -lt $olderThan ]]; then
      fileName=$(echo $line | awk '{print $4}')
      aws s3 rm s3://skillcore-backups/production/$fileName
    fi
  done

# Cleanup local file
rm /tmp/$BACKUP_FILE
```

### Point-in-Time Recovery (PITR)

**WAL Archiving**:
```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://skillcore-backups/wal/%f'
archive_timeout = 300  # 5 minutes
```

**Restore Process**:
```bash
# 1. Restore base backup
gunzip -c skillcore_backup_20251117.sql.gz | psql $DATABASE_URL

# 2. Restore WAL files up to target time
# PostgreSQL automatically applies WAL files

# 3. Verify restoration
psql $DATABASE_URL -c "SELECT NOW();"
```

---

## Monitoring

### Database Metrics

**Key Metrics**:
- Connection count
- Query duration (P50, P95, P99)
- Table sizes
- Index usage
- Cache hit ratio
- Replication lag (if applicable)

**Prometheus Exporter**:
```yaml
# postgres_exporter
DATA_SOURCE_NAME: "postgresql://user:pass@host:5432/skillcore"

# Metrics exposed
pg_stat_database_*
pg_stat_user_tables_*
pg_stat_user_indexes_*
pg_locks_*
```

**Slow Query Log**:
```sql
-- Enable slow query logging
ALTER DATABASE skillcore SET log_min_duration_statement = 1000; -- 1 second

-- View slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

## Best Practices

### Schema Design
1. **Normalize First**: Follow 3NF, denormalize for performance
2. **Foreign Keys**: Always define relationships
3. **Indexes**: Index foreign keys and frequently queried fields
4. **Soft Deletes**: Never hard delete student data (FERPA)
5. **Timestamps**: Always include createdAt, updatedAt

### Query Optimization
1. **Select Specific Fields**: Avoid SELECT *
2. **Use Indexes**: Ensure queries use indexes
3. **Limit Results**: Paginate large result sets
4. **Avoid N+1**: Use Prisma's include/select
5. **Connection Pooling**: Use PgBouncer in production

### Security
1. **Row-Level Security**: Enable RLS for multi-tenancy
2. **Least Privilege**: Grant minimal permissions
3. **Encryption**: Encrypt sensitive fields
4. **Parameterized Queries**: Prevent SQL injection
5. **Audit Logging**: Log all data changes

---

## Related Documentation

- [Audit System](./auditing.md) - Audit logging and FERPA compliance
- [Security](./security.md) - Database security and RLS
- [API Integration](../guides/api-integration.md) - Prisma usage patterns

---

**Database Metrics** (as of November 2025):
- **Total Tables**: 72
- **Total Indexes**: 180+
- **Database Size**: 45 GB
- **Daily Growth**: ~500 MB
- **Query Performance (P95)**: 120ms
- **Connection Pool**: 20 connections
- **Uptime**: 99.99%
