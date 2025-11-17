# System Administrator Portal

**Last Updated**: November 17, 2025  
**Status**: ✅ Production-Ready

---

## Overview

The System Administrator Portal provides comprehensive infrastructure management, monitoring, security, maintenance, and operational oversight for the SkillCore platform. Designed for DevOps, SRE, and system administrators managing the multi-tenant SaaS platform.

---

## Dashboard

**View**: `/system-admin/dashboard`

### Platform Health

**System Status**:
- Platform uptime (99.9%+ target)
- Active users (real-time)
- API response time (p50, p95, p99)
- Error rate (last hour)
- Database connections
- Queue depth (BullMQ)
- Event processing lag

**Alert Summary**:
- 🔴 Critical: System outages, data loss, security breaches
- 🟡 Warning: High resource usage, performance degradation, failed jobs
- 🟢 Info: Deployments, maintenance windows, routine tasks

**Service Health**:
- ✅ API Server (3 instances)
- ✅ Event Workers (5 instances)
- ✅ Database (Neon Postgres)
- ✅ Redis (Event streams + cache)
- ✅ BullMQ (7 queues)
- ✅ External Services (SendGrid, Twilio, Firebase)

---

## Infrastructure Management

**View**: `/system-admin/infrastructure`

### Server Management

**API Servers**:
- View all API instances
- Instance health
- CPU, memory, disk usage
- Request rate
- Error rate
- Restart/scale actions

**Event Workers**:
- Worker instances
- Consumer group status
- Event processing rate
- Handler execution times
- Failed event count
- DLQ depth

**Database**:
- Connection pool status
- Active queries
- Slow queries (> 1s)
- Database size
- Table sizes
- Index usage

**Redis**:
- Memory usage
- Key count
- Hit rate
- Stream metrics
- Pub/sub channels

### Kubernetes Management

**Cluster Overview** (if using K8s):
- Nodes
- Pods status
- Deployments
- Services
- ConfigMaps/Secrets
- Resource utilization

**Scaling**:
- Manual scaling (set replicas)
- Auto-scaling policies
- Resource limits
- Pod disruption budgets

---

## Queue Management

**View**: `/system-admin/queues`

### BullMQ Dashboard

**7 Production Queues**:

**1. notification-delivery**:
- Waiting: 45 jobs
- Active: 10 jobs
- Completed: 15,234 jobs (last 24h)
- Failed: 23 jobs
- Concurrency: 10
- Actions: Pause, Resume, Clean, Drain

**2. grade-processing**:
- Real-time metrics
- Job details
- Failure analysis

**3. attendance-aggregation**:
- Daily scheduled jobs
- Processing status

**4. report-generation**:
- PDF/Excel generation
- Queue depth monitoring

**5. data-archival**:
- Archival jobs
- S3 upload status

**6. intervention-detection**:
- At-risk student identification
- Automated intervention creation

**7. compliance-reporting**:
- FERPA audit logs
- Compliance report generation

### Queue Actions

**For Each Queue**:
- View jobs (waiting, active, completed, failed)
- Pause queue (stop processing)
- Resume queue
- Retry failed jobs
- Remove jobs
- Clean old jobs
- Export job data

### Failed Jobs

**DLQ Management**:
- View all failed jobs across queues
- Failure reason
- Stack trace
- Retry count
- Manual retry
- Delete permanently
- Export for analysis

**Failure Patterns**:
- Most common errors
- Failing job types
- Time-based patterns
- Alert thresholds

---

## Scheduled Jobs Management

**View**: `/system-admin/scheduled-jobs`

### Cron Jobs

**Daily Jobs** (2:00 AM):
- Stream cleanup (Redis)
- Attendance intervention detection
- IEP review reminders
- Grade posting digest

**Weekly Jobs** (Sundays, 3:00 AM):
- Performance trend calculations
- Resource usage analytics
- Inactive user notifications

**Monthly Jobs** (1st of month, 4:00 AM):
- Compliance audit reports
- Data archival to S3
- System health summary

### Job Monitoring

**For Each Scheduled Job**:
- Last run time
- Next run time
- Success/failure status
- Execution duration
- Output logs
- Manual trigger option

**Job History**:
- Execution logs
- Success rate
- Average duration
- Failure alerts

---

## Event System Management

**View**: `/system-admin/events`

### Event Streams

**Redis Streams**:
- `domain-events` (primary stream)
- `chat-events`
- `notification-events`

**For Each Stream**:
- Message count
- Consumer groups
- Pending messages
- Lag (time behind)
- Throughput (msgs/sec)

### Event Workers

**Worker Instances**:
- Worker ID and hostname
- Consumer group membership
- Events processed (last hour)
- Event processing rate
- Failed events
- Restart worker

**Event Handlers**:
- 79 registered handlers
- Handler execution times
- Success/failure rates
- Most frequently called
- Slowest handlers

### Event Monitoring

**Event Processing**:
- Events published (last hour)
- Events processed (last hour)
- Processing lag (time from publish to process)
- DLQ count (failed events)

**Event Types**:
- Top 10 event types by volume
- Event distribution by context
- Failed events by type

### Dead Letter Queue (DLQ)

**DLQ Dashboard**:
- Total messages in DLQ
- Oldest message timestamp
- Failure reasons
- Retry attempts
- Manual retry
- Archive/delete

---

## Monitoring & Observability

**View**: `/system-admin/monitoring`

### Metrics (Prometheus)

**System Metrics**:
- CPU usage (by instance)
- Memory usage
- Disk I/O
- Network throughput

**Application Metrics**:
- HTTP request rate
- Response times (p50, p95, p99)
- Error rate (4xx, 5xx)
- GraphQL operation times
- Database query times

**Business Metrics**:
- Active users (real-time)
- API calls per tenant
- Events published/processed
- Queue throughput
- Background job completion

### Logging (Grafana Loki)

**Log Aggregation**:
- All application logs
- Error logs
- Audit logs
- Access logs

**Log Search**:
- Full-text search
- Filter by level (info, warn, error)
- Filter by service (api, worker)
- Filter by tenant
- Time range selection
- Export logs

**Log Analytics**:
- Error frequency
- Warning trends
- User action tracking
- Performance insights

### Alerts (PagerDuty)

**Active Alerts**:
- Critical incidents
- Warning conditions
- Acknowledged alerts
- Resolved incidents

**Alert Rules**:
- High error rate (> 1%)
- Slow response times (p95 > 500ms)
- Queue backlog (> 1000 jobs)
- DLQ depth (> 50 messages)
- Database connection exhaustion
- High CPU/memory usage
- Event processing lag (> 5 minutes)

**Incident Management**:
- Create incident
- Assign to team member
- Update status
- Post-mortem analysis

---

## Security & Audit

**View**: `/system-admin/security`

### Security Dashboard

**Security Metrics**:
- Failed login attempts (last hour)
- Suspicious activity
- API rate limit violations
- Blocked IPs
- Security events

**Threat Detection**:
- SQL injection attempts
- XSS attempts
- CSRF violations
- Brute force attacks
- DDoS patterns

### Access Control

**Admin Users**:
- System administrators
- Role assignments
- Last login
- 2FA status
- Active sessions

**API Keys**:
- Service accounts
- API key management
- Key rotation
- Usage tracking
- Revoke keys

### Audit Logs

**FERPA Audit Trail**:
- Student data access (who, when, what)
- Data exports
- Permission changes
- Configuration changes
- Administrative actions

**Audit Search**:
- Search by user, action, date range
- Export audit data
- Compliance reporting
- Retention policy (7 years)

### Compliance

**FERPA Compliance**:
- Data encryption status
- Access control verification
- Audit log completeness
- Data retention policies
- Incident reports

**Security Certifications**:
- SOC 2 compliance
- GDPR compliance (if applicable)
- State-specific requirements
- Annual security audits

---

## Database Administration

**View**: `/system-admin/database`

### Database Overview

**Neon Postgres**:
- Database size
- Connection count
- Transaction rate
- Cache hit rate
- Replication lag (if applicable)

**Performance**:
- Slow queries (> 1 second)
- Most expensive queries
- Table scans (missing indexes)
- Index usage statistics
- Vacuum/analyze status

### Database Maintenance

**Backup & Recovery**:
- Automated backups (daily)
- Point-in-time recovery
- Backup retention (30 days)
- Restore operations
- Backup testing

**Schema Management**:
- Prisma migrations
- Migration history
- Pending migrations
- Rollback capabilities

**Data Integrity**:
- Foreign key violations
- Constraint violations
- Orphaned records
- Data quality checks

---

## Multi-Tenant Management

**View**: `/system-admin/tenants`

### Tenant Administration

**All Tenants**:
- Tenant ID and name
- Subscription tier
- User count
- Storage usage
- API usage
- Status (active, suspended, trial)

**Tenant Actions**:
- Create new tenant
- Suspend tenant
- Delete tenant
- Modify subscription
- Reset tenant data
- Export tenant data

### Tenant Isolation

**Data Isolation**:
- Row-level security (RLS)
- Schema isolation
- Connection pooling by tenant
- Query filtering

**Resource Limits**:
- API rate limits per tenant
- Storage quotas
- User limits
- Feature flags by tenant

### Tenant Analytics

**Usage Metrics**:
- API calls by tenant
- Storage by tenant
- User activity by tenant
- Feature adoption
- Resource consumption

---

## Performance Optimization

**View**: `/system-admin/performance`

### Performance Monitoring

**Response Times**:
- API endpoint latencies
- GraphQL operation times
- Database query times
- External API calls (SendGrid, Twilio)

**Optimization Opportunities**:
- Slow endpoints
- N+1 queries
- Missing indexes
- Cache hit rate
- CDN effectiveness

### Caching

**Redis Cache**:
- Cache size
- Hit/miss ratio
- Eviction rate
- TTL distribution
- Cache keys by pattern

**CDN (Cloudflare)**:
- Cache hit rate
- Bandwidth savings
- Geographic distribution
- Invalidation requests

---

## Deployment Management

**View**: `/system-admin/deployments`

### Deployment History

**Recent Deployments**:
- Deployment timestamp
- Version/commit hash
- Environment (production, staging)
- Deployed by
- Status (success, failed, rolled back)
- Duration

### CI/CD Pipeline

**GitHub Actions**:
- Build status
- Test results
- Deployment logs
- Rollback capability

**Deployment Process**:
1. Code push to main branch
2. Automated tests
3. Build Docker images
4. Deploy to staging
5. Run E2E tests
6. Deploy to production (manual approval)
7. Health checks
8. Rollback if needed

### Feature Flags

**Feature Management**:
- Active feature flags
- Rollout percentage
- Enabled for specific tenants
- A/B testing
- Kill switches

---

## External Services

**View**: `/system-admin/external-services`

### Email (SendGrid)

**SendGrid Dashboard**:
- Email sent (last 24h)
- Delivery rate
- Bounce rate
- Open rate
- Click rate
- Spam reports

**Webhooks**:
- Webhook status
- Event types tracked
- Failed webhook deliveries
- Retry queue

### SMS (Twilio)

**Twilio Dashboard**:
- SMS sent (last 24h)
- Delivery rate
- Error rate
- Cost per SMS
- Monthly spend

**Phone Numbers**:
- Active phone numbers
- Message throughput
- Number configuration

### Push Notifications (Firebase FCM)

**FCM Dashboard**:
- Push notifications sent
- Delivery rate
- Click-through rate
- Failed deliveries

**Device Management**:
- Active device tokens
- Platform distribution (iOS, Android, Web)
- Token refresh rate

---

## Maintenance & Operations

**View**: `/system-admin/maintenance`

### Maintenance Windows

**Scheduled Maintenance**:
- Plan maintenance window
- Notify tenants
- Status page updates
- Maintenance tasks checklist

**Maintenance Tasks**:
- Database maintenance
- Index rebuilding
- Log rotation
- Certificate renewal
- Dependency updates

### System Updates

**Dependency Management**:
- npm package updates
- Security patches
- Breaking changes review
- Update schedule

**Database Migrations**:
- Pending migrations
- Migration history
- Rollback plan
- Data migration scripts

---

## Disaster Recovery

**View**: `/system-admin/disaster-recovery`

### Backup Strategy

**Automated Backups**:
- Database backups (daily)
- Redis persistence
- File storage backups (S3)
- Configuration backups

**Recovery Objectives**:
- RTO (Recovery Time Objective): 1 hour
- RPO (Recovery Point Objective): 1 hour

### Incident Response

**Runbooks**:
- Database failure
- API outage
- Data breach
- External service failure
- Network issues

**Contact Information**:
- On-call engineer
- Escalation path
- Vendor contacts
- Emergency procedures

---

## Cost Management

**View**: `/system-admin/costs`

### Infrastructure Costs

**Monthly Costs**:
- Neon Postgres: $XXX
- Redis Cloud: $XXX
- Vercel hosting: $XXX
- S3 storage: $XXX
- Cloudflare CDN: $XXX
- SendGrid: $XXX
- Twilio: $XXX
- Firebase: $XXX

**Cost Optimization**:
- Resource right-sizing
- Reserved instances
- Auto-scaling policies
- Storage lifecycle policies

---

## Reports & Analytics

**View**: `/system-admin/reports`

### System Reports

**Daily Reports**:
- System health summary
- Error summary
- Performance metrics
- User activity

**Weekly Reports**:
- Uptime report
- Incident summary
- Cost analysis
- Capacity planning

**Monthly Reports**:
- SLA compliance
- Security audit
- Performance trends
- Growth metrics

---

## Settings & Configuration

**View**: `/system-admin/settings`

### Global Configuration

**Platform Settings**:
- Platform name and branding
- Maintenance mode toggle
- Feature flags (global)
- Default tenant settings

**Environment Variables**:
- View/edit environment variables
- Secrets management
- Configuration versioning

### Admin Users

**System Administrators**:
- Add/remove admins
- Role assignments
- Permission levels
- 2FA enforcement

---

## Related Documentation

- [Architecture Overview](../architecture/overview.md)
- [Event System](../architecture/events.md)
- [Background Workers](../features/workers.md)
- [Deployment Guide](../guides/deployment.md)
