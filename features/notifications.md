# Multi-Channel Notification System

**Last Updated**: November 16, 2025  
**Status**: ✅ Production-Ready

---

## Overview

SkillCore WebApp features a comprehensive multi-channel notification system supporting Email (SendGrid), SMS (Twilio), and Push Notifications (Firebase FCM) with complete delivery tracking, engagement analytics, and FERPA compliance.

---

## Supported Channels

### 1. Email (SendGrid)

**Implementation**: SendGrid Transactional Email API

**Features**:
- ✅ Transactional emails (grade posted, assignment due, etc.)
- ✅ Email templates with dynamic content
- ✅ Real-time webhooks for delivery status
- ✅ Engagement tracking (opens, clicks)
- ✅ Bounce handling (hard/soft bounces)
- ✅ Unsubscribe management

**Event Types**:
1. `EmailSent` - Email dispatched to SendGrid
2. `EmailDelivered` - SendGrid confirmed delivery
3. `EmailBounced` - Email bounced (hard or soft)
4. `EmailOpened` - Recipient opened email (tracking pixel)
5. `EmailClicked` - Link clicked in email (click tracking)
6. `EmailDeliveryFailed` - Email failed to send

**Use Cases**:
- Grade posting notifications
- Assignment due reminders
- Parent-teacher communication
- School announcements
- Report card availability
- IEP meeting invitations

**Configuration**:
```typescript
SENDGRID_API_KEY=your_api_key
SENDGRID_FROM_EMAIL=noreply@skillcore.app
SENDGRID_FROM_NAME=SkillCore Platform
```

---

### 2. SMS (Twilio)

**Implementation**: Twilio Messaging API

**Features**:
- ✅ Emergency alerts and urgent notifications
- ✅ Real-time delivery status via webhooks
- ✅ Two-way messaging support
- ✅ TCPA compliance with opt-out management
- ✅ Message templates for consistency
- ✅ International SMS support

**Event Types**:
1. `SMSSent` - SMS dispatched to Twilio
2. `SMSDelivered` - Twilio confirmed delivery
3. `SMSFailed` - SMS failed to send
4. `SMSDeliveryFailed` - Delivery failure (invalid number, carrier issue)
5. `SMSReplied` - Recipient replied to SMS

**Use Cases**:
- Emergency school closures
- Critical attendance alerts (3+ absences)
- Urgent parent notifications
- Emergency contact messages
- Quick reminders (meeting in 30 min)
- Two-factor authentication

**Configuration**:
```typescript
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE_NUMBER=+1234567890
```

**Compliance**:
- Opt-out keywords: STOP, UNSUBSCRIBE, CANCEL, END, QUIT
- Opt-in confirmation required
- Time restrictions: 8am-9pm local time
- Emergency override for critical alerts

---

### 3. Push Notifications (Firebase FCM)

**Implementation**: Firebase Cloud Messaging

**Features**:
- ✅ iOS push notifications (APNs via FCM)
- ✅ Android push notifications
- ✅ Web push notifications (PWA)
- ✅ Topic-based subscriptions
- ✅ Token management and rotation
- ✅ Badge updates and sound control
- ✅ Rich notifications with images/actions

**Event Types**:
1. `PushNotificationSent` - Push sent to Firebase FCM
2. `PushNotificationDelivered` - FCM confirmed delivery
3. `PushNotificationClicked` - User clicked notification
4. `PushNotificationFailed` - Push failed to send
5. `PushNotificationDeliveryFailed` - Delivery failure (invalid token, app uninstalled)

**Use Cases**:
- New grade posted
- New message received
- Assignment due soon
- Class announcements
- Calendar event reminders
- Real-time chat notifications

**Configuration**:
```typescript
FIREBASE_PROJECT_ID=your_project_id
FIREBASE_SERVICE_ACCOUNT=path/to/service-account.json
```

**Topics**:
- `grade-updates-{studentId}` - Grade notifications
- `class-announcements-{classId}` - Class updates
- `messages-{userId}` - Chat messages
- `emergency-alerts` - School-wide emergencies

---

## Multi-Channel Orchestration

### Priority-Based Routing

**High Priority** (Urgent, Emergency):
- Send to ALL channels simultaneously
- Email + SMS + Push
- Example: Emergency school closure, critical attendance alert

**Normal Priority**:
- Send to preferred channel first
- Fallback to secondary if primary fails
- Example: Grade posted, assignment due

**Low Priority**:
- Send to single preferred channel only
- No fallback
- Example: Weekly digest, resource recommendations

### Fallback Logic

```typescript
// Priority: High
1. Send Push (fastest)
2. Send SMS (immediate)
3. Send Email (reliable)

// Priority: Normal
1. Send to user's preferred channel
2. If failed, try next available channel
3. Log delivery status

// Priority: Low
1. Send to preferred channel only
2. Don't retry if failed
```

### User Preferences

**Per-Channel Toggles**:
```typescript
interface NotificationPreferences {
  emailEnabled: boolean
  smsEnabled: boolean
  pushEnabled: boolean
  
  preferences: {
    gradePosted: { email: true, sms: false, push: true }
    assignmentDue: { email: true, sms: false, push: true }
    attendanceAlert: { email: true, sms: true, push: true }
    messageReceived: { email: false, sms: false, push: true }
    // ... 11 total notification types
  }
}
```

**Quiet Hours**:
- No non-emergency notifications between 9pm-7am
- Emergency notifications override quiet hours
- User can customize quiet hours

---

## Delivery Tracking & Analytics

### Delivery Logs

**NotificationDeliveryLog Model**:
```typescript
{
  id: string
  notificationId: string
  channel: 'email' | 'sms' | 'push'
  status: 'sent' | 'delivered' | 'failed' | 'bounced'
  recipientId: string
  sentAt: Date
  deliveredAt?: Date
  failedAt?: Date
  failureReason?: string
  metadata: {
    category: string      // 'grade_posted', 'attendance_alert', etc.
    priority: string      // 'low', 'normal', 'high', 'urgent'
    districtId: string
    schoolId?: string
  }
}
```

### Engagement Tracking

**NotificationEngagement Model**:
```typescript
{
  id: string
  deliveryLogId: string
  engagementType: 'opened' | 'clicked' | 'replied'
  engagedAt: Date
  metadata: {
    linkClicked?: string  // Which link in email
    deviceType?: string   // 'mobile', 'desktop'
    location?: string     // IP-based location
  }
}
```

### Analytics Aggregation

**NotificationAnalytics Model**:
```typescript
{
  id: string
  date: Date
  channel: 'email' | 'sms' | 'push'
  category: string
  districtId: string
  
  totalSent: number
  totalDelivered: number
  totalFailed: number
  totalOpened: number
  totalClicked: number
  
  deliveryRate: number      // delivered / sent
  openRate: number          // opened / delivered
  clickThroughRate: number  // clicked / opened
}
```

**Daily Rollup**:
- Aggregated every night at midnight
- Per-channel, per-category, per-district
- 90-day retention for detailed logs
- 2-year retention for aggregated analytics

---

## Webhook Processing

### SendGrid Webhooks

**Endpoint**: `POST /webhooks/sendgrid`

**Events Handled**:
- `delivered` → `EmailDelivered`
- `bounce` → `EmailBounced`
- `open` → `EmailOpened`
- `click` → `EmailClicked`
- `dropped` → `EmailDeliveryFailed`

**Validation**:
- Webhook signature verification
- Timestamp validation (prevent replay)
- IP whitelist (SendGrid IPs only)

### Twilio Webhooks

**Endpoint**: `POST /webhooks/twilio`

**Events Handled**:
- `delivered` → `SMSDelivered`
- `failed` → `SMSFailed`
- `undelivered` → `SMSDeliveryFailed`
- `received` → `SMSReplied` (for two-way SMS)

**Validation**:
- Twilio request signature validation
- Account SID verification

### Firebase FCM Callbacks

**Implementation**: Firebase Admin SDK callbacks

**Events Handled**:
- Token refresh → Update user device token
- Message sent → `PushNotificationSent`
- Message delivered → `PushNotificationDelivered`
- Token invalid → Remove device token
- App uninstalled → Deactivate push for user

---

## Notification Templates

### Email Templates

**Grade Posted Template**:
```html
Subject: New Grade Posted - {{assignment.title}}

Hi {{student.name}},

Your teacher {{teacher.name}} has posted a new grade:

Assignment: {{assignment.title}}
Grade: {{grade.score}}/{{grade.maxScore}} ({{grade.percentage}}%)
Feedback: {{grade.feedback}}

View full details: {{link}}
```

**Attendance Alert Template**:
```html
Subject: Attendance Alert - {{student.name}}

Dear {{parent.name}},

This is to notify you that {{student.name}} was absent from school today:

Date: {{attendance.date}}
Period: {{attendance.period}}
Class: {{class.name}}

If this was an excused absence, please contact the attendance office.
```

### SMS Templates

**Emergency Alert**:
```
URGENT: School closure due to {{reason}}. All classes cancelled {{date}}. Updates at {{link}}
```

**Attendance Alert**:
```
{{student.name}} absent from {{class.name}} today. Reply EXCUSE for excused absence or call office.
```

### Push Notification Templates

**Grade Posted**:
```json
{
  "title": "New Grade: {{assignment.title}}",
  "body": "{{grade.percentage}}% - {{teacher.name}}",
  "icon": "grade_icon",
  "badge": "{{unread_count}}",
  "data": {
    "type": "grade_posted",
    "gradeId": "{{grade.id}}",
    "deepLink": "/grades/{{grade.id}}"
  }
}
```

---

## FERPA Compliance

### Data Protection
- ✅ Student data encrypted in transit and at rest
- ✅ No student info in push notification bodies
- ✅ Secure deep links with authentication required
- ✅ Parent consent required for all notifications
- ✅ Audit logging for all deliveries

### Privacy Controls
- ✅ Parents can opt out of specific notification types
- ✅ Students 18+ control their own preferences
- ✅ Teachers cannot access parent contact info directly
- ✅ District admin can disable channels system-wide

### Audit Trail
- Every notification delivery logged
- Retention: 7 years for compliance
- Searchable by student, parent, teacher
- Export capability for compliance reviews

---

## Performance & Scalability

### Rate Limiting
- **SendGrid**: 100 emails/second per API key
- **Twilio**: 1000 SMS/second (configurable)
- **Firebase**: 1 million concurrent connections

### Batching
- Email: Batch up to 1000 recipients per request
- SMS: Send individually (no batching)
- Push: Topic messages (unlimited recipients)

### Queue Management
- **BullMQ Queue**: `notification-delivery`
- **Workers**: 3-5 workers for parallel processing
- **Priority**: Urgent → High → Normal → Low
- **Retry**: 5 attempts with exponential backoff

---

## Monitoring & Alerts

### Metrics Tracked
- Delivery rate per channel
- Average delivery time
- Failure rate and reasons
- Engagement rates (open, click)
- Cost per notification (Twilio, SendGrid)

### Alerts
- Delivery rate < 95% → Alert ops team
- Failure rate > 5% → Investigate immediately
- Webhook processing lag > 5 min → Scale workers
- SendGrid/Twilio API errors → PagerDuty alert

---

## Cost Optimization

### SendGrid Pricing
- Free tier: 100 emails/day
- Pro: $19.95/mo for 40,000 emails
- Current usage: ~5,000 emails/day

### Twilio Pricing
- SMS: $0.0079 per message (US)
- Current usage: ~200 SMS/day
- Estimated cost: ~$50/month

### Firebase FCM
- Free for all tiers
- No message limits
- No cost optimization needed

### Strategies
1. Use push instead of SMS when possible (free vs paid)
2. Batch digest emails instead of individual (reduce volume)
3. Smart routing: prefer cheaper channels for low-priority
4. Respect quiet hours to reduce unnecessary sends

---

## Related Documentation

- [Event Architecture](../architecture/events.md) - Event system
- [API Integration](../architecture/api-integration.md) - API patterns
- [Features Index](./README.md) - All features
