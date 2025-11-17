# REST API Reference

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**API Version**: v1  
**OpenAPI**: 3.0.3

---

## Overview

SkillCore's REST API provides **HTTP endpoints** for operations that don't fit the GraphQL model, such as **file uploads**, **exports**, **webhooks**, and **health checks**. All endpoints follow **RESTful conventions** with **JSON** request/response bodies.

**Base URL**: `https://api.skillcore.app/api/v1`

---

## Authentication

All REST endpoints require **JWT authentication** via the `Authorization` header (except public health checks):

```http
GET /api/v1/students/123/report
Authorization: Bearer <access_token>
Accept: application/json
```

---

## Health & Status

### GET /health

**Description**: Health check endpoint for monitoring  
**Authentication**: None  
**Rate Limit**: Unlimited

**Response**:
```json
{
  "status": "healthy",
  "timestamp": "2025-11-17T10:30:00Z",
  "uptime": 864000,
  "version": "1.5.2",
  "services": {
    "database": "healthy",
    "redis": "healthy",
    "eventBus": "healthy"
  }
}
```

### GET /metrics

**Description**: Prometheus metrics  
**Authentication**: None  
**Rate Limit**: Unlimited

**Response**:
```text
# HELP api_http_requests_total Total HTTP requests
# TYPE api_http_requests_total counter
api_http_requests_total{method="GET",status="200"} 1234567

# HELP api_http_request_duration_seconds HTTP request duration
# TYPE api_http_request_duration_seconds histogram
api_http_request_duration_seconds_bucket{le="0.1"} 89234
...
```

---

## File Operations

### POST /upload

**Description**: Upload a file (avatar, attachment, etc.)  
**Authentication**: Required  
**Content-Type**: `multipart/form-data`  
**Rate Limit**: 10 requests/minute

**Request**:
```http
POST /api/v1/upload
Authorization: Bearer <token>
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="document.pdf"
Content-Type: application/pdf

<binary data>
--boundary
Content-Disposition: form-data; name="type"

assignment_attachment
--boundary--
```

**Response**:
```json
{
  "success": true,
  "file": {
    "id": "file-123",
    "fileName": "document.pdf",
    "fileSize": 102400,
    "mimeType": "application/pdf",
    "url": "https://cdn.skillcore.app/files/file-123.pdf",
    "uploadedAt": "2025-11-17T10:30:00Z"
  }
}
```

**Error Response**:
```json
{
  "success": false,
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "File size exceeds maximum of 10MB",
    "maxSize": 10485760
  }
}
```

### GET /files/:fileId

**Description**: Download a file  
**Authentication**: Required (access verification)  
**Rate Limit**: 100 requests/minute

**Response**: File binary data with appropriate `Content-Type` header

---

## Export Operations

### POST /export/gradebook

**Description**: Export gradebook to Excel/CSV  
**Authentication**: Required (TEACHER or ADMIN)  
**Rate Limit**: 5 requests/minute

**Request**:
```json
{
  "classId": "class-123",
  "academicSessionId": "session-456",
  "format": "xlsx",  // xlsx, csv, pdf
  "includeComments": true,
  "includeStatistics": true
}
```

**Response**:
```json
{
  "success": true,
  "exportId": "export-789",
  "status": "processing",
  "estimatedCompletion": "2025-11-17T10:35:00Z"
}
```

**Poll for Completion**:
```http
GET /api/v1/export/export-789/status
```

**Response** (completed):
```json
{
  "exportId": "export-789",
  "status": "completed",
  "downloadUrl": "https://exports.skillcore.app/export-789.xlsx",
  "expiresAt": "2025-11-18T10:30:00Z",
  "fileSize": 524288
}
```

### POST /export/student-report

**Description**: Export student progress report  
**Authentication**: Required (TEACHER, PARENT, or STUDENT)  
**Rate Limit**: 10 requests/minute

**Request**:
```json
{
  "studentId": "student-123",
  "academicSessionId": "session-456",
  "format": "pdf",
  "includeGrades": true,
  "includeAttendance": true,
  "includeBehavior": true,
  "includeComments": true
}
```

**Response**:
```json
{
  "success": true,
  "reportUrl": "https://reports.skillcore.app/student-123-report.pdf",
  "generatedAt": "2025-11-17T10:30:00Z",
  "expiresAt": "2025-11-24T10:30:00Z"
}
```

### POST /export/attendance

**Description**: Export attendance records  
**Authentication**: Required (TEACHER or ADMIN)  
**Rate Limit**: 5 requests/minute

**Request**:
```json
{
  "schoolId": "school-123",
  "startDate": "2025-09-01",
  "endDate": "2025-11-17",
  "format": "xlsx",
  "gradeLevel": "10"  // Optional filter
}
```

**Response**:
```json
{
  "success": true,
  "exportId": "export-attendance-456",
  "status": "processing"
}
```

---

## Webhook Endpoints

### POST /webhooks/sendgrid

**Description**: SendGrid email event webhook  
**Authentication**: SendGrid signature verification  
**Rate Limit**: Unlimited

**Request**:
```json
[
  {
    "email": "parent@example.com",
    "timestamp": 1700226600,
    "event": "delivered",
    "sg_event_id": "sendgrid-event-123",
    "sg_message_id": "msg-456"
  },
  {
    "email": "teacher@example.com",
    "timestamp": 1700226601,
    "event": "open",
    "sg_event_id": "sendgrid-event-124",
    "sg_message_id": "msg-457"
  }
]
```

**Response**:
```json
{
  "success": true,
  "processed": 2
}
```

### POST /webhooks/twilio

**Description**: Twilio SMS event webhook  
**Authentication**: Twilio signature verification  
**Rate Limit**: Unlimited

**Request**:
```http
POST /api/v1/webhooks/twilio
Content-Type: application/x-www-form-urlencoded

MessageSid=SM123456
MessageStatus=delivered
To=%2B15555551234
From=%2B15555555678
```

**Response**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Message>Event received</Message>
</Response>
```

---

## Batch Operations

### POST /batch/update-grades

**Description**: Batch update grades for multiple students  
**Authentication**: Required (TEACHER)  
**Rate Limit**: 10 requests/minute

**Request**:
```json
{
  "lineItemId": "lineitem-123",
  "grades": [
    {
      "studentId": "student-1",
      "score": 85,
      "scoreStatus": "fully_graded"
    },
    {
      "studentId": "student-2",
      "score": 92,
      "scoreStatus": "fully_graded"
    },
    {
      "studentId": "student-3",
      "score": 78,
      "scoreStatus": "fully_graded",
      "comment": "Good improvement!"
    }
  ],
  "postImmediately": true
}
```

**Response**:
```json
{
  "success": true,
  "successCount": 3,
  "errorCount": 0,
  "results": [
    {
      "studentId": "student-1",
      "resultId": "result-1",
      "status": "success"
    },
    {
      "studentId": "student-2",
      "resultId": "result-2",
      "status": "success"
    },
    {
      "studentId": "student-3",
      "resultId": "result-3",
      "status": "success"
    }
  ]
}
```

### POST /batch/mark-attendance

**Description**: Batch mark attendance for multiple students  
**Authentication**: Required (TEACHER)  
**Rate Limit**: 20 requests/minute

**Request**:
```json
{
  "classId": "class-123",
  "date": "2025-11-17",
  "periodId": "period-1",
  "records": [
    {
      "studentId": "student-1",
      "status": "present"
    },
    {
      "studentId": "student-2",
      "status": "absent",
      "reason": "Sick"
    },
    {
      "studentId": "student-3",
      "status": "tardy",
      "notes": "Arrived at 8:15 AM"
    }
  ]
}
```

**Response**:
```json
{
  "success": true,
  "successCount": 3,
  "errorCount": 0,
  "date": "2025-11-17"
}
```

---

## Import Operations

### POST /import/students

**Description**: Bulk import students from CSV  
**Authentication**: Required (ADMIN)  
**Content-Type**: `multipart/form-data`  
**Rate Limit**: 5 requests/hour

**Request**:
```http
POST /api/v1/import/students
Authorization: Bearer <token>
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="students.csv"
Content-Type: text/csv

studentNumber,givenName,familyName,gradeLevel,dateOfBirth,email
123456,John,Doe,10,2008-05-15,john.doe@example.com
123457,Jane,Smith,10,2008-06-20,jane.smith@example.com
--boundary
Content-Disposition: form-data; name="schoolId"

school-123
--boundary--
```

**Response**:
```json
{
  "success": true,
  "importId": "import-789",
  "status": "processing",
  "totalRows": 2
}
```

**Poll for Status**:
```http
GET /api/v1/import/import-789/status
```

**Response**:
```json
{
  "importId": "import-789",
  "status": "completed",
  "totalRows": 2,
  "successCount": 2,
  "errorCount": 0,
  "errors": [],
  "completedAt": "2025-11-17T10:35:00Z"
}
```

---

## SIS Integration

### POST /sis/sync

**Description**: Trigger SIS data synchronization  
**Authentication**: Required (ADMIN)  
**Rate Limit**: 1 request/hour

**Request**:
```json
{
  "sisProvider": "PowerSchool",  // PowerSchool, Infinite Campus, Skyward, etc.
  "syncType": "full",  // full, incremental
  "entities": ["students", "classes", "grades", "attendance"]
}
```

**Response**:
```json
{
  "success": true,
  "syncJobId": "sync-123",
  "status": "queued",
  "estimatedDuration": 1800  // seconds
}
```

**Poll for Status**:
```http
GET /api/v1/sis/sync/sync-123/status
```

**Response**:
```json
{
  "syncJobId": "sync-123",
  "status": "completed",
  "startedAt": "2025-11-17T10:30:00Z",
  "completedAt": "2025-11-17T10:55:00Z",
  "results": {
    "students": {
      "created": 25,
      "updated": 350,
      "errors": 0
    },
    "classes": {
      "created": 5,
      "updated": 45,
      "errors": 0
    },
    "grades": {
      "created": 1200,
      "updated": 500,
      "errors": 2
    },
    "attendance": {
      "created": 8500,
      "updated": 100,
      "errors": 0
    }
  },
  "errors": [
    {
      "entity": "grades",
      "record": "grade-xyz",
      "error": "Invalid score value"
    }
  ]
}
```

---

## Analytics Endpoints

### GET /analytics/dashboard

**Description**: Get dashboard analytics summary  
**Authentication**: Required  
**Rate Limit**: 60 requests/minute

**Query Parameters**:
- `role`: User role (teacher, parent, student, admin)
- `academicSessionId`: Academic session ID
- `schoolId`: School ID (admin only)

**Response** (Teacher):
```json
{
  "role": "teacher",
  "teacherId": "teacher-123",
  "academicSession": {
    "id": "session-456",
    "title": "2025-2026 School Year"
  },
  "summary": {
    "totalStudents": 125,
    "totalClasses": 5,
    "avgClassSize": 25,
    "avgClassGrade": 85.4,
    "attendanceRate": 94.2
  },
  "classes": [
    {
      "id": "class-1",
      "title": "Algebra I - Period 1",
      "studentCount": 28,
      "classAverage": 87.2,
      "attendanceRate": 95.1,
      "assignmentsDue": 3,
      "studentsAtRisk": 2
    }
  ],
  "recentActivity": [
    {
      "type": "grade_posted",
      "timestamp": "2025-11-17T09:15:00Z",
      "description": "Posted grades for Chapter 5 Quiz"
    }
  ],
  "alerts": [
    {
      "type": "low_grade",
      "severity": "medium",
      "studentId": "student-789",
      "message": "Student John Doe has a D in Algebra I"
    }
  ]
}
```

---

## Error Responses

### Standard Error Format

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "details": [
      {
        "field": "studentNumber",
        "message": "Student number must be 6-10 digits"
      }
    ]
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | Insufficient permissions for operation |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 400 | Request validation failed |
| `DUPLICATE_RESOURCE` | 409 | Resource already exists |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Internal server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |

---

## Rate Limiting

**Headers**:
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1700226660
```

**Rate Limit Exceeded Response**:
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Try again in 45 seconds.",
    "retryAfter": 45
  }
}
```

---

## OpenAPI Specification

**Download**: [https://api.skillcore.app/api-docs/openapi.json](https://api.skillcore.app/api-docs/openapi.json)

**Swagger UI**: [https://api.skillcore.app/api-docs](https://api.skillcore.app/api-docs)

---

## SDK Support

### TypeScript/JavaScript

```typescript
import { SkillCoreAPI } from '@skillcore/sdk'

const api = new SkillCoreAPI({
  baseURL: 'https://api.skillcore.app',
  apiKey: process.env.SKILLCORE_API_KEY,
})

// Upload file
const file = await api.upload({
  file: fileBlob,
  type: 'assignment_attachment',
})

// Export gradebook
const exportJob = await api.exportGradebook({
  classId: 'class-123',
  academicSessionId: 'session-456',
  format: 'xlsx',
})

// Poll for completion
const result = await api.pollExport(exportJob.exportId)
console.log('Download URL:', result.downloadUrl)
```

---

## Related Documentation

- [GraphQL API](./graphql.md) - GraphQL operations
- [WebSocket API](./websocket.md) - Real-time subscriptions
- [Authentication](../architecture/security.md) - JWT authentication

---

**REST API Statistics** (as of November 2025):
- **Total Endpoints**: 47
- **Average Response Time**: 95ms
- **Uptime**: 99.99%
- **Daily Requests**: 2.5M
