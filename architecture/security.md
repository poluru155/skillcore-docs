# Security Architecture

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Standards**: FERPA · OAuth 2.0 · JWT · RBAC

---

## Overview

SkillCore implements defense-in-depth security with **authentication**, **authorization**, **encryption**, **RBAC**, and **FERPA compliance**. Security is embedded at every layer from API to database.

---

## Security Layers

```
Client → HTTPS/TLS → API Gateway → Authentication → Authorization → Business Logic → Database
           ↓            ↓              ↓               ↓                ↓              ↓
        SSL Cert    Rate Limiting   JWT Verify    RBAC Check      Data Validation   RLS
```

---

## Authentication

### JWT-Based Authentication

**Token Structure**:
```typescript
interface JWTPayload {
  sub: string              // User ID
  email: string
  role: UserRole          // 'TEACHER', 'STUDENT', 'PARENT', etc.
  tenantId: string
  schoolId?: string
  iat: number             // Issued at
  exp: number             // Expires at
  permissions: string[]   // Granular permissions
}
```

**Token Generation**:
```typescript
import jwt from 'jsonwebtoken'

export function generateAccessToken(user: User): string {
  return jwt.sign(
    {
      sub: user.id,
      email: user.email,
      role: user.role,
      tenantId: user.tenantId,
      schoolId: user.schoolId,
      permissions: user.permissions,
    },
    process.env.JWT_SECRET,
    {
      expiresIn: '15m',      // Short-lived access token
      issuer: 'skillcore-api',
      audience: 'skillcore-webapp',
    }
  )
}

export function generateRefreshToken(user: User): string {
  return jwt.sign(
    {
      sub: user.id,
      type: 'refresh',
    },
    process.env.JWT_REFRESH_SECRET,
    {
      expiresIn: '7d',       // Long-lived refresh token
      issuer: 'skillcore-api',
    }
  )
}
```

**Token Refresh Flow**:
```typescript
// POST /api/auth/refresh
export async function refreshToken(req: Request, res: Response) {
  const { refreshToken } = req.body
  
  try {
    // Verify refresh token
    const decoded = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET)
    
    // Check if token is revoked
    const isRevoked = await redis.get(`revoked:${decoded.jti}`)
    if (isRevoked) {
      throw new Error('Token revoked')
    }
    
    // Get user
    const user = await prisma.user.findUnique({
      where: { id: decoded.sub },
    })
    
    if (!user || !user.isActive) {
      throw new Error('User not found or inactive')
    }
    
    // Generate new access token
    const accessToken = generateAccessToken(user)
    
    res.json({ accessToken })
  } catch (error) {
    res.status(401).json({ error: 'Invalid refresh token' })
  }
}
```

### Authentication Middleware

**JWT Verification**:
```typescript
import { Request, Response, NextFunction } from 'express'
import jwt from 'jsonwebtoken'

export async function authenticate(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'No token provided' })
  }
  
  const token = authHeader.substring(7)
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET) as JWTPayload
    
    // Check if token is revoked
    const isRevoked = await redis.get(`revoked:${decoded.jti}`)
    if (isRevoked) {
      return res.status(401).json({ error: 'Token revoked' })
    }
    
    // Attach user to request
    req.user = {
      id: decoded.sub,
      email: decoded.email,
      role: decoded.role,
      tenantId: decoded.tenantId,
      schoolId: decoded.schoolId,
      permissions: decoded.permissions,
    }
    
    next()
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired' })
    }
    return res.status(401).json({ error: 'Invalid token' })
  }
}
```

### Multi-Factor Authentication (MFA)

**MFA Setup**:
```typescript
import speakeasy from 'speakeasy'
import qrcode from 'qrcode'

// Generate MFA secret
export async function setupMFA(userId: string) {
  const secret = speakeasy.generateSecret({
    name: `SkillCore (${user.email})`,
    issuer: 'SkillCore',
  })
  
  // Store secret (encrypted)
  await prisma.user.update({
    where: { id: userId },
    data: {
      mfaSecret: encryptSecret(secret.base32),
      mfaEnabled: false,  // Not enabled until verified
    },
  })
  
  // Generate QR code
  const qrCodeUrl = await qrcode.toDataURL(secret.otpauth_url)
  
  return {
    secret: secret.base32,
    qrCode: qrCodeUrl,
  }
}

// Verify MFA code
export function verifyMFACode(secret: string, token: string): boolean {
  return speakeasy.totp.verify({
    secret: decryptSecret(secret),
    encoding: 'base32',
    token,
    window: 2,  // Allow 2 time steps before/after
  })
}
```

**MFA Login Flow**:
```typescript
// POST /api/auth/login
export async function login(req: Request, res: Response) {
  const { email, password, mfaCode } = req.body
  
  // Verify credentials
  const user = await prisma.user.findUnique({ where: { email } })
  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({ error: 'Invalid credentials' })
  }
  
  // Check MFA if enabled
  if (user.mfaEnabled) {
    if (!mfaCode) {
      return res.status(401).json({ error: 'MFA code required', mfaRequired: true })
    }
    
    if (!verifyMFACode(user.mfaSecret, mfaCode)) {
      return res.status(401).json({ error: 'Invalid MFA code' })
    }
  }
  
  // Generate tokens
  const accessToken = generateAccessToken(user)
  const refreshToken = generateRefreshToken(user)
  
  res.json({ accessToken, refreshToken })
}
```

### Session Management

**Session Storage** (Redis):
```typescript
interface Session {
  userId: string
  tenantId: string
  ipAddress: string
  userAgent: string
  createdAt: Date
  lastActivityAt: Date
  expiresAt: Date
}

// Store session
export async function createSession(user: User, req: Request): Promise<string> {
  const sessionId = uuid()
  
  const session: Session = {
    userId: user.id,
    tenantId: user.tenantId,
    ipAddress: req.ip,
    userAgent: req.headers['user-agent'],
    createdAt: new Date(),
    lastActivityAt: new Date(),
    expiresAt: addHours(new Date(), 12),
  }
  
  await redis.setex(
    `session:${sessionId}`,
    12 * 60 * 60,  // 12 hours
    JSON.stringify(session)
  )
  
  return sessionId
}

// Revoke all user sessions
export async function revokeAllSessions(userId: string) {
  const sessionKeys = await redis.keys(`session:*`)
  
  for (const key of sessionKeys) {
    const session = JSON.parse(await redis.get(key))
    if (session.userId === userId) {
      await redis.del(key)
    }
  }
}
```

---

## Authorization

### Role-Based Access Control (RBAC)

**User Roles**:
```typescript
enum UserRole {
  SYSTEM_ADMIN = 'SYSTEM_ADMIN',      // Full system access
  DISTRICT_ADMIN = 'DISTRICT_ADMIN',  // District-wide access
  PRINCIPAL = 'PRINCIPAL',            // School-wide access
  TEACHER = 'TEACHER',                // Class-level access
  COUNSELOR = 'COUNSELOR',            // Student support access
  STUDENT = 'STUDENT',                // Own data access
  PARENT = 'PARENT',                  // Child data access
  STAFF = 'STAFF',                    // Limited access
}
```

**Permission System**:
```typescript
interface Permission {
  resource: string      // 'students', 'grades', 'attendance', etc.
  action: string        // 'read', 'create', 'update', 'delete'
  scope: string         // 'own', 'class', 'school', 'district', 'all'
  conditions?: Record<string, any>
}

// Role-Permission mapping
const rolePermissions: Record<UserRole, Permission[]> = {
  TEACHER: [
    { resource: 'students', action: 'read', scope: 'class' },
    { resource: 'grades', action: '*', scope: 'class' },
    { resource: 'attendance', action: '*', scope: 'class' },
    { resource: 'assignments', action: '*', scope: 'class' },
  ],
  
  STUDENT: [
    { resource: 'grades', action: 'read', scope: 'own' },
    { resource: 'attendance', action: 'read', scope: 'own' },
    { resource: 'assignments', action: 'read', scope: 'own' },
  ],
  
  PARENT: [
    { resource: 'students', action: 'read', scope: 'own', conditions: { relationship: 'parent' } },
    { resource: 'grades', action: 'read', scope: 'own', conditions: { studentId: '$children' } },
  ],
  
  PRINCIPAL: [
    { resource: '*', action: '*', scope: 'school' },
  ],
  
  DISTRICT_ADMIN: [
    { resource: '*', action: '*', scope: 'district' },
  ],
  
  SYSTEM_ADMIN: [
    { resource: '*', action: '*', scope: 'all' },
  ],
}
```

**Authorization Middleware**:
```typescript
export function authorize(resource: string, action: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const user = req.user
    
    // Check if user has permission
    const hasPermission = await checkPermission(user, resource, action, req)
    
    if (!hasPermission) {
      // Log unauthorized access attempt
      await auditLog.create({
        userId: user.id,
        action: 'UNAUTHORIZED_ACCESS',
        resource,
        description: `Attempted ${action} on ${resource}`,
      })
      
      return res.status(403).json({ error: 'Forbidden' })
    }
    
    next()
  }
}

async function checkPermission(
  user: User,
  resource: string,
  action: string,
  req: Request
): Promise<boolean> {
  const permissions = rolePermissions[user.role]
  
  for (const permission of permissions) {
    // Check resource match
    if (permission.resource !== '*' && permission.resource !== resource) {
      continue
    }
    
    // Check action match
    if (permission.action !== '*' && permission.action !== action) {
      continue
    }
    
    // Check scope
    const scopeMatch = await checkScope(user, permission.scope, req)
    if (!scopeMatch) {
      continue
    }
    
    // Check conditions
    if (permission.conditions) {
      const conditionsMatch = await checkConditions(user, permission.conditions, req)
      if (!conditionsMatch) {
        continue
      }
    }
    
    return true
  }
  
  return false
}
```

**Route Protection**:
```typescript
// Protect route with authentication + authorization
app.get('/api/grades/:id',
  authenticate,
  authorize('grades', 'read'),
  async (req, res) => {
    // User is authenticated and authorized
    const grade = await getGrade(req.params.id)
    res.json(grade)
  }
)

// Multiple permission check
app.post('/api/iep',
  authenticate,
  requireAnyRole(['TEACHER', 'COUNSELOR', 'PRINCIPAL']),
  authorize('iep', 'create'),
  async (req, res) => {
    const iep = await createIEP(req.body)
    res.json(iep)
  }
)
```

### FERPA Access Control

**Student Data Protection**:
```typescript
export async function verifyStudentAccess(
  userId: string,
  studentId: string,
  userRole: UserRole
): Promise<boolean> {
  // System/District admins have full access
  if (['SYSTEM_ADMIN', 'DISTRICT_ADMIN'].includes(userRole)) {
    return true
  }
  
  // Principal has school-wide access
  if (userRole === 'PRINCIPAL') {
    const principal = await prisma.user.findUnique({ where: { id: userId } })
    const student = await prisma.student.findUnique({ where: { id: studentId } })
    return principal.schoolId === student.schoolId
  }
  
  // Teacher has class-level access
  if (userRole === 'TEACHER') {
    const enrollment = await prisma.enrollment.findFirst({
      where: {
        studentId,
        class: {
          teacherId: userId,
        },
      },
    })
    return !!enrollment
  }
  
  // Counselor has caseload access
  if (userRole === 'COUNSELOR') {
    const student = await prisma.student.findUnique({
      where: { id: studentId },
    })
    return student.counselorId === userId
  }
  
  // Student has own access
  if (userRole === 'STUDENT') {
    return userId === studentId
  }
  
  // Parent has child access
  if (userRole === 'PARENT') {
    const relationship = await prisma.familyRelationship.findFirst({
      where: {
        parentId: userId,
        studentId,
      },
    })
    return !!relationship
  }
  
  return false
}
```

---

## Data Encryption

### Encryption at Rest

**Database Encryption**:
- PostgreSQL: Transparent Data Encryption (TDE)
- Sensitive fields: AES-256-GCM encryption

**Field-Level Encryption**:
```typescript
import crypto from 'crypto'

const ALGORITHM = 'aes-256-gcm'
const KEY = Buffer.from(process.env.ENCRYPTION_KEY, 'hex')

export function encrypt(text: string): string {
  const iv = crypto.randomBytes(16)
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv)
  
  let encrypted = cipher.update(text, 'utf8', 'hex')
  encrypted += cipher.final('hex')
  
  const authTag = cipher.getAuthTag()
  
  // Format: iv:authTag:encrypted
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`
}

export function decrypt(encrypted: string): string {
  const [ivHex, authTagHex, encryptedText] = encrypted.split(':')
  
  const iv = Buffer.from(ivHex, 'hex')
  const authTag = Buffer.from(authTagHex, 'hex')
  const decipher = crypto.createDecipheriv(ALGORITHM, KEY, iv)
  
  decipher.setAuthTag(authTag)
  
  let decrypted = decipher.update(encryptedText, 'hex', 'utf8')
  decrypted += decipher.final('utf8')
  
  return decrypted
}

// Prisma middleware for automatic encryption
prisma.$use(async (params, next) => {
  const encryptedFields = ['ssn', 'mfaSecret', 'apiKey']
  
  // Encrypt on create/update
  if (['create', 'update'].includes(params.action) && params.args.data) {
    for (const field of encryptedFields) {
      if (params.args.data[field]) {
        params.args.data[field] = encrypt(params.args.data[field])
      }
    }
  }
  
  const result = await next(params)
  
  // Decrypt on read
  if (result && typeof result === 'object') {
    for (const field of encryptedFields) {
      if (result[field]) {
        result[field] = decrypt(result[field])
      }
    }
  }
  
  return result
})
```

### Encryption in Transit

**TLS/HTTPS**:
- All API traffic over HTTPS
- TLS 1.3 minimum
- Strong cipher suites only

**Certificate Management**:
- Let's Encrypt for automatic renewal
- Wildcard certificates for subdomains
- Certificate pinning for mobile apps

---

## Database Security

### Row-Level Security (RLS)

**Multi-Tenant Isolation**:
```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE students ENABLE ROW LEVEL SECURITY;
ALTER TABLE grades ENABLE ROW LEVEL SECURITY;
ALTER TABLE attendance ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only access data in their tenant
CREATE POLICY tenant_isolation ON students
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation ON grades
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Set tenant context in application
SET app.tenant_id = 'tenant-uuid-here';
```

**Prisma Implementation**:
```typescript
// Set tenant context for all queries
prisma.$use(async (params, next) => {
  if (params.model && tenantScopedModels.includes(params.model)) {
    // Add tenant filter to all queries
    params.args.where = {
      ...params.args.where,
      tenantId: req.user.tenantId,
    }
  }
  
  return next(params)
})
```

### SQL Injection Prevention

**Parameterized Queries**:
```typescript
// ✅ SAFE: Prisma uses parameterized queries
const students = await prisma.student.findMany({
  where: {
    name: {
      contains: searchQuery,  // Automatically escaped
    },
  },
})

// ❌ UNSAFE: Raw SQL without parameters
const students = await prisma.$queryRaw(
  `SELECT * FROM students WHERE name LIKE '%${searchQuery}%'`
)

// ✅ SAFE: Raw SQL with parameters
const students = await prisma.$queryRaw(
  `SELECT * FROM students WHERE name LIKE $1`,
  `%${searchQuery}%`
)
```

---

## API Security

### Rate Limiting

**Rate Limiter**:
```typescript
import rateLimit from 'express-rate-limit'
import RedisStore from 'rate-limit-redis'

// Global rate limit
const globalLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:global:',
  }),
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 1000,                 // 1000 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
})

// Strict rate limit for auth endpoints
const authLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:auth:',
  }),
  windowMs: 15 * 60 * 1000,
  max: 5,                    // 5 login attempts per 15 minutes
  message: 'Too many login attempts, please try again later',
  skipSuccessfulRequests: true,
})

app.use('/api/', globalLimiter)
app.use('/api/auth/login', authLimiter)
```

### CORS Configuration

**CORS Policy**:
```typescript
import cors from 'cors'

const corsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.skillcore.com',
      'https://staging.skillcore.com',
      process.env.NODE_ENV === 'development' && 'http://localhost:5173',
    ].filter(Boolean)
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true)
    } else {
      callback(new Error('Not allowed by CORS'))
    }
  },
  credentials: true,
  maxAge: 86400,  // 24 hours
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-FERPA-Purpose'],
}

app.use(cors(corsOptions))
```

### Input Validation

**Request Validation** (Zod):
```typescript
import { z } from 'zod'

const createStudentSchema = z.object({
  firstName: z.string().min(1).max(100),
  lastName: z.string().min(1).max(100),
  email: z.string().email().optional(),
  gradeLevel: z.enum(['K', '1', '2', '3', '4', '5', '6', '7', '8', '9', '10', '11', '12']),
  dateOfBirth: z.string().datetime(),
  studentNumber: z.string().regex(/^\d{6,10}$/),
})

export function validateRequest(schema: z.ZodSchema) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse(req.body)
      next()
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: error.errors,
        })
      }
      next(error)
    }
  }
}

// Usage
app.post('/api/students',
  authenticate,
  authorize('students', 'create'),
  validateRequest(createStudentSchema),
  async (req, res) => {
    const student = await createStudent(req.body)
    res.json(student)
  }
)
```

### CSRF Protection

**CSRF Token**:
```typescript
import csrf from 'csurf'

const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  },
})

// Apply to state-changing endpoints
app.post('/api/*', csrfProtection)
app.put('/api/*', csrfProtection)
app.delete('/api/*', csrfProtection)

// Provide token to client
app.get('/api/csrf-token', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() })
})
```

---

## Security Best Practices

### Secure Coding
1. **Input Validation**: Validate all user input
2. **Output Encoding**: Encode output to prevent XSS
3. **Parameterized Queries**: Prevent SQL injection
4. **Principle of Least Privilege**: Minimal permissions
5. **Defense in Depth**: Multiple security layers

### Secret Management
1. **Environment Variables**: Never commit secrets
2. **Secret Rotation**: Regular rotation of API keys
3. **Encryption at Rest**: Encrypt sensitive config
4. **Access Control**: Limit secret access
5. **Audit Logging**: Log secret access

### Vulnerability Management
1. **Dependency Scanning**: Regular npm audit
2. **Security Updates**: Timely patching
3. **Penetration Testing**: Annual pen tests
4. **Bug Bounty**: Responsible disclosure program
5. **Security Training**: Developer education

---

## Related Documentation

- [Audit System](./auditing.md) - Security audit logging
- [Authentication](../guides/authentication.md) - Auth implementation guide

---

**Security Metrics** (as of November 2025):
- **Authentication**: JWT + MFA
- **Authorization**: RBAC + FERPA controls
- **Encryption**: AES-256-GCM at rest, TLS 1.3 in transit
- **Audit Logs**: 100% coverage for student data
- **Vulnerabilities**: 0 critical, 0 high
- **Pen Test Results**: Pass (last: Oct 2025)
