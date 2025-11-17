# Monitoring & Observability Architecture

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Stack**: Prometheus · Grafana · Loki · PagerDuty

---

## Overview

SkillCore implements comprehensive monitoring and observability using **Prometheus** for metrics, **Grafana** for visualization, **Loki** for log aggregation, and **PagerDuty** for alerting. This ensures system reliability, performance optimization, and rapid incident response.

---

## Monitoring Stack

```
Application Metrics → Prometheus → Grafana Dashboards
Application Logs → Loki → Grafana Log Explorer
Alerts → Prometheus Alertmanager → PagerDuty/Slack
Traces (Future) → OpenTelemetry → Jaeger
```

---

## Prometheus Metrics

### Metrics Exposition

**API Metrics Endpoint**:
```typescript
// src/routes/metrics.ts
import { Router } from 'express'
import { register } from 'prom-client'

const router = Router()

// Expose metrics at /metrics endpoint
router.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType)
  res.end(await register.metrics())
})

export default router
```

### Application Metrics

**HTTP Request Metrics**:
```typescript
import { Counter, Histogram, Gauge } from 'prom-client'

// HTTP request counter
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code', 'tenant_id'],
})

// HTTP request duration
export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5, 10],
})

// Active requests gauge
export const httpActiveRequests = new Gauge({
  name: 'http_active_requests',
  help: 'Number of active HTTP requests',
  labelNames: ['method', 'route'],
})

// Middleware to track metrics
export function metricsMiddleware(req, res, next) {
  const start = Date.now()
  
  httpActiveRequests.inc({ method: req.method, route: req.route?.path })
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000
    
    httpRequestsTotal.inc({
      method: req.method,
      route: req.route?.path || 'unknown',
      status_code: res.statusCode,
      tenant_id: req.user?.tenantId || 'anonymous',
    })
    
    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || 'unknown',
        status_code: res.statusCode,
      },
      duration
    )
    
    httpActiveRequests.dec({ method: req.method, route: req.route?.path })
  })
  
  next()
}
```

**Database Metrics**:
```typescript
import { Counter, Histogram } from 'prom-client'

// Database query counter
export const dbQueriesTotal = new Counter({
  name: 'db_queries_total',
  help: 'Total number of database queries',
  labelNames: ['model', 'operation', 'status'],
})

// Database query duration
export const dbQueryDuration = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries in seconds',
  labelNames: ['model', 'operation'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
})

// Prisma middleware
prisma.$use(async (params, next) => {
  const start = Date.now()
  
  try {
    const result = await next(params)
    const duration = (Date.now() - start) / 1000
    
    dbQueriesTotal.inc({
      model: params.model || 'unknown',
      operation: params.action,
      status: 'success',
    })
    
    dbQueryDuration.observe(
      {
        model: params.model || 'unknown',
        operation: params.action,
      },
      duration
    )
    
    return result
  } catch (error) {
    dbQueriesTotal.inc({
      model: params.model || 'unknown',
      operation: params.action,
      status: 'error',
    })
    
    throw error
  }
})
```

**EventBus Metrics**:
```typescript
export const eventbusPublishedTotal = new Counter({
  name: 'eventbus_published_total',
  help: 'Total number of events published',
  labelNames: ['event_type', 'tenant_id'],
})

export const eventbusConsumedTotal = new Counter({
  name: 'eventbus_consumed_total',
  help: 'Total number of events consumed',
  labelNames: ['event_type', 'handler', 'status'],
})

export const eventbusProcessingDuration = new Histogram({
  name: 'eventbus_processing_duration_seconds',
  help: 'Duration of event processing in seconds',
  labelNames: ['event_type', 'handler'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5, 10, 30],
})

export const eventbusQueueDepth = new Gauge({
  name: 'eventbus_queue_depth',
  help: 'Number of pending events in queue',
  labelNames: ['consumer_group'],
})

export const eventbusDLQTotal = new Counter({
  name: 'eventbus_dlq_total',
  help: 'Total number of events in dead letter queue',
  labelNames: ['event_type'],
})
```

**Queue Metrics (BullMQ)**:
```typescript
export const bullmqJobsTotal = new Counter({
  name: 'bullmq_jobs_total',
  help: 'Total number of jobs processed',
  labelNames: ['queue', 'job_type', 'status'],
})

export const bullmqJobDuration = new Histogram({
  name: 'bullmq_job_duration_seconds',
  help: 'Duration of job processing in seconds',
  labelNames: ['queue', 'job_type'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 120, 300],
})

export const bullmqQueueWaiting = new Gauge({
  name: 'bullmq_queue_waiting_total',
  help: 'Number of jobs waiting in queue',
  labelNames: ['queue'],
})

export const bullmqQueueActive = new Gauge({
  name: 'bullmq_queue_active_total',
  help: 'Number of jobs currently processing',
  labelNames: ['queue'],
})

export const bullmqQueueFailed = new Counter({
  name: 'bullmq_queue_failed_total',
  help: 'Total number of failed jobs',
  labelNames: ['queue', 'job_type'],
})

// Collect queue metrics every 30 seconds
setInterval(async () => {
  for (const queue of queues) {
    const waiting = await queue.getWaitingCount()
    const active = await queue.getActiveCount()
    
    bullmqQueueWaiting.set({ queue: queue.name }, waiting)
    bullmqQueueActive.set({ queue: queue.name }, active)
  }
}, 30000)
```

**Business Metrics**:
```typescript
// Educational metrics
export const studentsEnrolled = new Gauge({
  name: 'students_enrolled_total',
  help: 'Total number of enrolled students',
  labelNames: ['tenant_id', 'school_id'],
})

export const gradesPosted = new Counter({
  name: 'grades_posted_total',
  help: 'Total number of grades posted',
  labelNames: ['tenant_id', 'school_id', 'class_id'],
})

export const attendanceMarked = new Counter({
  name: 'attendance_marked_total',
  help: 'Total number of attendance records',
  labelNames: ['tenant_id', 'school_id', 'status'],
})

export const interventionsActive = new Gauge({
  name: 'interventions_active_total',
  help: 'Number of active interventions',
  labelNames: ['tenant_id', 'school_id', 'tier'],
})

export const notificationsSent = new Counter({
  name: 'notifications_sent_total',
  help: 'Total number of notifications sent',
  labelNames: ['tenant_id', 'channel', 'type', 'status'],
})
```

### Grafana Dashboards

**1. API Overview Dashboard**

**Panels**:
- Request rate (requests/sec)
- Error rate (%)
- P50/P95/P99 latency
- Active requests
- Requests by endpoint
- Requests by tenant
- Error breakdown by status code

**Queries**:
```promql
# Request rate
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m])

# P95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Top endpoints by requests
topk(10, sum by (route) (rate(http_requests_total[5m])))
```

**2. Database Performance Dashboard**

**Panels**:
- Query rate (queries/sec)
- Query duration P95/P99
- Slow queries (>1s)
- Queries by model
- Connection pool usage
- Transaction rate

**Queries**:
```promql
# Query rate
rate(db_queries_total[5m])

# P95 query duration
histogram_quantile(0.95, rate(db_query_duration_seconds_bucket[5m]))

# Slow queries
rate(db_query_duration_seconds_bucket{le="1"}[5m])

# Top models by query count
topk(10, sum by (model) (rate(db_queries_total[5m])))
```

**3. EventBus Dashboard**

**Panels**:
- Events published/sec
- Events consumed/sec
- Processing duration P95
- Queue depth
- Failed events
- DLQ count
- Consumer lag

**Queries**:
```promql
# Event publish rate
rate(eventbus_published_total[5m])

# Event consumption rate
rate(eventbus_consumed_total{status="success"}[5m])

# Processing duration P95
histogram_quantile(0.95, rate(eventbus_processing_duration_seconds_bucket[5m]))

# Queue depth
eventbus_queue_depth

# DLQ count
eventbus_dlq_total
```

**4. Queue Workers Dashboard**

**Panels**:
- Jobs processed/sec by queue
- Job duration P95 by queue
- Waiting jobs by queue
- Active jobs by queue
- Failed jobs by queue
- Job success rate

**Queries**:
```promql
# Jobs processed per second
rate(bullmq_jobs_total{status="completed"}[5m])

# Job duration P95
histogram_quantile(0.95, rate(bullmq_job_duration_seconds_bucket[5m]))

# Waiting jobs
bullmq_queue_waiting_total

# Job success rate
rate(bullmq_jobs_total{status="completed"}[5m]) / rate(bullmq_jobs_total[5m])
```

**5. Business Metrics Dashboard**

**Panels**:
- Total enrolled students
- Grades posted today
- Attendance marked today
- Active interventions
- Notifications sent by channel
- User logins by role

**Queries**:
```promql
# Enrolled students
students_enrolled_total

# Grades posted today
increase(grades_posted_total[24h])

# Attendance marked today
increase(attendance_marked_total[24h])

# Active interventions by tier
interventions_active_total
```

---

## Log Aggregation (Loki)

### Log Collection

**Structured Logging**:
```typescript
import winston from 'winston'
import LokiTransport from 'winston-loki'

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'skillcore-api',
    environment: process.env.NODE_ENV,
  },
  transports: [
    // Console
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    }),
    
    // Loki
    new LokiTransport({
      host: process.env.LOKI_URL,
      labels: {
        app: 'skillcore-api',
        environment: process.env.NODE_ENV,
      },
      json: true,
      format: winston.format.json(),
      replaceTimestamp: true,
      onConnectionError: (err) => console.error(err),
    }),
  ],
})

export default logger
```

**Contextual Logging**:
```typescript
// Add request context to logs
export function loggerMiddleware(req, res, next) {
  req.logger = logger.child({
    requestId: req.id,
    userId: req.user?.id,
    tenantId: req.user?.tenantId,
    method: req.method,
    path: req.path,
  })
  
  next()
}

// Usage in handlers
app.get('/api/students/:id', async (req, res) => {
  req.logger.info('Fetching student', { studentId: req.params.id })
  
  try {
    const student = await getStudent(req.params.id)
    req.logger.info('Student fetched successfully')
    res.json(student)
  } catch (error) {
    req.logger.error('Failed to fetch student', { error: error.message, stack: error.stack })
    res.status(500).json({ error: 'Internal server error' })
  }
})
```

### Log Queries (LogQL)

**Common Queries**:
```logql
# All error logs
{app="skillcore-api"} |= "error"

# Logs for specific user
{app="skillcore-api"} | json | userId="user-123"

# Slow database queries
{app="skillcore-api"} | json | duration > 1000

# Failed login attempts
{app="skillcore-api"} | json | action="LOGIN_FAILED"

# Rate of 5xx errors
rate({app="skillcore-api"} | json | status_code =~ "5.." [5m])

# Top error messages
topk(10, count_over_time({app="skillcore-api"} |= "error" [1h]))
```

**Grafana Loki Integration**:
- **Log Explorer**: Search and filter logs
- **Live Tail**: Real-time log streaming
- **Log Context**: View logs around specific timestamp
- **Log Metrics**: Convert logs to metrics

---

## Alerting

### Prometheus Alertmanager

**Alert Rules** (`prometheus-alerts.yml`):
```yaml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High API latency"
          description: "P95 latency is {{ $value }}s (threshold: 2s)"
      
      # Queue depth too high
      - alert: HighQueueDepth
        expr: bullmq_queue_waiting_total > 1000
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Queue {{ $labels.queue }} has high depth"
          description: "{{ $labels.queue }} has {{ $value }} waiting jobs"
      
      # Dead letter queue growing
      - alert: DLQGrowing
        expr: increase(eventbus_dlq_total[1h]) > 100
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Dead letter queue growing rapidly"
          description: "DLQ grew by {{ $value }} events in last hour"
      
      # Database slow queries
      - alert: SlowDatabaseQueries
        expr: |
          histogram_quantile(0.95, rate(db_query_duration_seconds_bucket[5m])) > 1
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Slow database queries detected"
          description: "P95 query duration is {{ $value }}s"
      
      # Worker not processing jobs
      - alert: WorkerStalled
        expr: |
          rate(bullmq_jobs_total{status="completed"}[10m]) == 0
          and bullmq_queue_waiting_total > 0
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Worker for {{ $labels.queue }} appears stalled"
          description: "No jobs processed in 10 minutes but {{ $value }} jobs waiting"

  - name: business_alerts
    interval: 1m
    rules:
      # Too many failed interventions
      - alert: InterventionCreationFailing
        expr: |
          rate(eventbus_consumed_total{event_type="InterventionCreated",status="failed"}[15m]) > 5
        labels:
          severity: warning
          team: product
        annotations:
          summary: "High intervention creation failure rate"
          
      # Notification delivery failure
      - alert: NotificationDeliveryFailing
        expr: |
          rate(notifications_sent_total{status="failed"}[15m]) / rate(notifications_sent_total[15m]) > 0.1
        for: 10m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "High notification delivery failure rate"
          description: "{{ $value | humanizePercentage }} of notifications failing"
```

### PagerDuty Integration

**Alertmanager Configuration** (`alertmanager.yml`):
```yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  
  routes:
    # Critical alerts to PagerDuty
    - match:
        severity: critical
      receiver: pagerduty
      continue: true
    
    # Warnings to Slack
    - match:
        severity: warning
      receiver: slack
    
    # Business alerts to product team
    - match:
        team: product
      receiver: slack-product

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://localhost:5001/'
  
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '<PAGERDUTY_SERVICE_KEY>'
        description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
        severity: '{{ .Labels.severity }}'
  
  - name: 'slack'
    slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'
        channel: '#eng-alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ .Annotations.description }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
  
  - name: 'slack-product'
    slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'
        channel: '#product-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ .Annotations.summary }}'
```

### Slack Notifications

**Alert Format**:
```
🚨 CRITICAL: High Error Rate
Error rate is 8.5% (threshold: 5%)

📊 Details:
- Service: skillcore-api
- Environment: production
- Time: 2025-11-17 14:23:00 UTC

🔗 Links:
- Dashboard: https://grafana.skillcore.com/d/api-overview
- Runbook: https://docs.skillcore.com/runbooks/high-error-rate
```

---

## Health Checks

### Application Health

**Health Check Endpoint** (`/health`):
```typescript
import { Router } from 'express'
import { prisma } from '../lib/prisma'
import { redis } from '../lib/redis'

const router = Router()

router.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    checks: {
      database: 'unknown',
      redis: 'unknown',
      eventbus: 'unknown',
    },
  }
  
  try {
    // Database check
    await prisma.$queryRaw`SELECT 1`
    health.checks.database = 'healthy'
  } catch (error) {
    health.checks.database = 'unhealthy'
    health.status = 'unhealthy'
  }
  
  try {
    // Redis check
    await redis.ping()
    health.checks.redis = 'healthy'
  } catch (error) {
    health.checks.redis = 'unhealthy'
    health.status = 'unhealthy'
  }
  
  try {
    // EventBus check
    const streamInfo = await redis.xinfo('STREAM', 'events:*')
    health.checks.eventbus = 'healthy'
  } catch (error) {
    health.checks.eventbus = 'unhealthy'
    health.status = 'degraded'
  }
  
  const statusCode = health.status === 'healthy' ? 200 : 503
  res.status(statusCode).json(health)
})

export default router
```

**Kubernetes Liveness/Readiness**:
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: api
    image: skillcore/api:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2
```

---

## System Admin Dashboard

### Monitoring Overview

**Access**: `/admin/monitoring` (System Admin role only)

**Features**:
- Real-time metrics visualization
- Log search and filtering
- Active alerts list
- Health check status
- Performance trends
- Resource utilization

**Key Metrics Displayed**:
- API request rate (requests/sec)
- Error rate (%)
- P95 latency (ms)
- Database query rate
- Queue depths
- Worker status
- EventBus throughput
- Active users

---

## Best Practices

### Monitoring Design
1. **RED Metrics**: Rate, Errors, Duration for all services
2. **USE Metrics**: Utilization, Saturation, Errors for resources
3. **Business Metrics**: Track educational KPIs
4. **SLI/SLO**: Define service level indicators and objectives
5. **Cardinality**: Limit high-cardinality labels

### Alerting Strategy
1. **Actionable**: Only alert on actionable conditions
2. **Severity Levels**: Critical (PagerDuty), Warning (Slack), Info (logs)
3. **Runbooks**: Link alerts to runbooks
4. **Alert Fatigue**: Tune thresholds to reduce noise
5. **On-Call Rotation**: Define clear escalation paths

### Performance
1. **Metric Collection**: Use pull model (Prometheus scraping)
2. **Retention**: 15 days in Prometheus, 90 days in long-term storage
3. **Sampling**: Sample high-volume metrics if needed
4. **Aggregation**: Pre-aggregate metrics where possible
5. **Query Optimization**: Use recording rules for complex queries

---

## Related Documentation

- [Queue Architecture](./queues.md) - Queue metrics
- [Events Architecture](./events.md) - EventBus metrics
- [System Admin Portal](../roles/system-admin.md) - Monitoring dashboard

---

**Production Metrics** (as of November 2025):
- **Prometheus Retention**: 15 days
- **Metrics Scraped**: 500+ metrics
- **Dashboards**: 12 Grafana dashboards
- **Alert Rules**: 25 active rules
- **Uptime**: 99.9% (last 30 days)
- **P95 API Latency**: 180ms
- **Error Rate**: 0.3%
