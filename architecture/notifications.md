# Multi-Channel Notification Architecture

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Channels**: Email (SendGrid) · SMS (Twilio) · Push (Firebase FCM)

---

## Overview

SkillCore implements a comprehensive multi-channel notification system supporting **email**, **SMS**, and **push notifications**. The architecture is built on event-driven processing, external service integrations, and robust delivery tracking.

---

## Architecture Pattern

```
User Action → Domain Event → Notification Event Handler → Channel Selector
                                                                  ↓
                    ┌──────────────────────────────────────────────┴──────┐
                    ↓                      ↓                               ↓
              Email Queue            SMS Queue                    Push Queue
           (SendGrid API)         (Twilio API)             (Firebase FCM API)
                    ↓                      ↓                               ↓
              Email Webhook          SMS Webhook                 FCM Callback
              (Delivery Status)      (Delivery Status)           (Delivery Status)
                    ↓                      ↓                               ↓
              Event Handler          Event Handler                Event Handler
                    ↓                      ↓                               ↓
           NotificationDeliveryLog  NotificationDeliveryLog  NotificationDeliveryLog
                    └──────────────────────┬───────────────────────────────┘
                                           ↓
                                  Analytics Dashboard
```

---

## Channel 1: Email (SendGrid)

### Configuration

**SendGrid Setup**:
```typescript
import sgMail from '@sendgrid/mail'

sgMail.setApiKey(process.env.SENDGRID_API_KEY)

const sendEmail = async (emailData: EmailData) => {
  const msg = {
    to: emailData.to,
    from: {
      email: 'noreply@skillcore.com',
      name: 'SkillCore',
    },
    templateId: emailData.templateId,
    dynamicTemplateData: emailData.templateData,
    trackingSettings: {
      clickTracking: { enable: true },
      openTracking: { enable: true },
    },
  }
  
  const response = await sgMail.send(msg)
  return response
}
```

**Environment Variables**:
```env
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxxxxxxx
SENDGRID_FROM_EMAIL=noreply@skillcore.com
SENDGRID_FROM_NAME=SkillCore
SENDGRID_WEBHOOK_SECRET=xxxxxxxxxxxxx
```

### Email Templates

**Template Types** (SendGrid Dynamic Templates):

1. **welcome-email** (`d-abc123`)
   - Sent when: New user account created
   - Recipients: Students, Teachers, Parents
   - Variables: `name`, `role`, `schoolName`, `loginUrl`

2. **grade-posted** (`d-def456`)
   - Sent when: Grade posted by teacher
   - Recipients: Students, Parents
   - Variables: `studentName`, `className`, `assignmentName`, `grade`, `percentage`

3. **attendance-alert** (`d-ghi789`)
   - Sent when: Student absent
   - Recipients: Parents
   - Variables: `studentName`, `date`, `status`, `reason`
   - Priority: High (emergency absences)

4. **assignment-due** (`d-jkl012`)
   - Sent when: Assignment due in 1-3 days
   - Recipients: Students, Parents
   - Variables: `studentName`, `className`, `assignmentName`, `dueDate`, `daysRemaining`

5. **intervention-created** (`d-mno345`)
   - Sent when: Intervention created for student
   - Recipients: Counselors, Teachers, Parents
   - Variables: `studentName`, `interventionType`, `tier`, `description`

6. **iep-review-reminder** (`d-pqr678`)
   - Sent when: IEP review approaching
   - Recipients: Special Ed Coordinator, Case Manager, Parents
   - Variables: `studentName`, `reviewDate`, `daysUntilReview`

7. **conference-scheduled** (`d-stu901`)
   - Sent when: Parent-teacher conference scheduled
   - Recipients: Parents, Teachers
   - Variables: `studentName`, `teacherName`, `date`, `time`, `location`, `zoomLink`

8. **weekly-digest** (`d-vwx234`)
   - Sent when: Weekly summary (Sundays)
   - Recipients: Students, Parents
   - Variables: `studentName`, `gradesPosted`, `upcomingAssignments`, `attendanceSummary`

### Email Events

**Domain Events**:
```typescript
// Sent from notification queue worker
interface EmailSent {
  notificationId: string
  to: string | string[]
  subject: string
  templateId: string
  messageId: string      // SendGrid message ID
  tenantId: string
  userId: string
  sentAt: Date
}

// Received from SendGrid webhook
interface EmailDelivered {
  notificationId: string
  messageId: string
  email: string
  timestamp: Date
  smtpResponse: string
}

interface EmailBounced {
  notificationId: string
  messageId: string
  email: string
  bounceType: 'hard' | 'soft'  // Hard: invalid email, Soft: mailbox full
  reason: string
  timestamp: Date
}

interface EmailOpened {
  notificationId: string
  messageId: string
  email: string
  userAgent: string
  ipAddress: string
  timestamp: Date
}

interface EmailClicked {
  notificationId: string
  messageId: string
  email: string
  url: string
  timestamp: Date
}

interface EmailDeliveryFailed {
  notificationId: string
  messageId: string
  email: string
  errorCode: string
  errorMessage: string
  timestamp: Date
}
```

### SendGrid Webhooks

**Webhook Configuration**:
```typescript
// POST /api/webhooks/sendgrid
import { Router } from 'express'
import { eventBus } from '../infrastructure/event-bus'

const router = Router()

router.post('/sendgrid', async (req, res) => {
  // Verify webhook signature
  const signature = req.headers['x-twilio-email-event-webhook-signature']
  const timestamp = req.headers['x-twilio-email-event-webhook-timestamp']
  
  if (!verifySignature(signature, timestamp, req.body)) {
    return res.status(401).json({ error: 'Invalid signature' })
  }
  
  const events = req.body
  
  for (const event of events) {
    const { event: eventType, email, sg_message_id } = event
    
    // Find notification by SendGrid message ID
    const notification = await prisma.notification.findFirst({
      where: { externalId: sg_message_id },
    })
    
    if (!notification) {
      console.warn('Notification not found for message:', sg_message_id)
      continue
    }
    
    // Publish domain event based on webhook type
    switch (eventType) {
      case 'delivered':
        await eventBus.publish('EmailDelivered', {
          notificationId: notification.id,
          messageId: sg_message_id,
          email,
          timestamp: new Date(event.timestamp * 1000),
          smtpResponse: event.response,
        })
        break
        
      case 'bounce':
        await eventBus.publish('EmailBounced', {
          notificationId: notification.id,
          messageId: sg_message_id,
          email,
          bounceType: event.type === 'bounce' ? 'hard' : 'soft',
          reason: event.reason,
          timestamp: new Date(event.timestamp * 1000),
        })
        break
        
      case 'open':
        await eventBus.publish('EmailOpened', {
          notificationId: notification.id,
          messageId: sg_message_id,
          email,
          userAgent: event.useragent,
          ipAddress: event.ip,
          timestamp: new Date(event.timestamp * 1000),
        })
        break
        
      case 'click':
        await eventBus.publish('EmailClicked', {
          notificationId: notification.id,
          messageId: sg_message_id,
          email,
          url: event.url,
          timestamp: new Date(event.timestamp * 1000),
        })
        break
    }
  }
  
  res.status(200).json({ success: true })
})

export default router
```

**Webhook URL**: `https://api.skillcore.com/webhooks/sendgrid`

**Events Tracked**:
- `delivered` - Email successfully delivered
- `bounce` - Email bounced (invalid address or full mailbox)
- `open` - Email opened by recipient
- `click` - Link clicked in email
- `dropped` - Email dropped by SendGrid (spam, invalid)
- `deferred` - Temporary delivery failure (retrying)
- `processed` - Email processed by SendGrid

### Email Analytics

**Metrics Tracked**:
```typescript
interface EmailAnalytics {
  sent: number
  delivered: number
  bounced: number
  opened: number
  clicked: number
  deliveryRate: number      // (delivered / sent) * 100
  openRate: number          // (opened / delivered) * 100
  clickRate: number         // (clicked / delivered) * 100
  bounceRate: number        // (bounced / sent) * 100
}
```

**Dashboard Queries**:
```graphql
query GetEmailAnalytics($dateRange: DateRangeInput!) {
  emailAnalytics(dateRange: $dateRange) {
    sent
    delivered
    bounced
    opened
    clicked
    deliveryRate
    openRate
    clickRate
    bounceRate
    topTemplates {
      templateId
      name
      sent
      openRate
    }
  }
}
```

---

## Channel 2: SMS (Twilio)

### Configuration

**Twilio Setup**:
```typescript
import twilio from 'twilio'

const client = twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
)

const sendSMS = async (smsData: SMSData) => {
  const message = await client.messages.create({
    to: smsData.to,              // +1234567890
    from: process.env.TWILIO_PHONE_NUMBER,
    body: smsData.message,
    statusCallback: `${process.env.API_URL}/webhooks/twilio`,
  })
  
  return message
}
```

**Environment Variables**:
```env
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxx
TWILIO_PHONE_NUMBER=+15551234567
TWILIO_WEBHOOK_SECRET=xxxxxxxxxxxxx
```

### SMS Use Cases

**High Priority** (Emergency Alerts):
- School closure notifications
- Emergency lockdown alerts
- Student safety incidents
- Critical attendance alerts (unexplained absence)
- Weather delays/closures

**Normal Priority**:
- Attendance notifications (daily absence)
- Conference reminders (1 day before)
- Event reminders
- Assignment due reminders (opt-in)

**Opt-In Required**:
- Grade postings
- Assignment reminders
- Weekly digests

**TCPA Compliance**:
- All recipients must opt-in to non-emergency SMS
- Opt-out keyword: "STOP" (auto-unsubscribe)
- Help keyword: "HELP" (send support info)
- Quiet hours: No SMS between 9 PM - 8 AM local time

### SMS Templates

**Template Format** (Character limits):
```typescript
interface SMSTemplate {
  id: string
  name: string
  message: string           // Max 160 characters (single segment)
  variables: string[]
}

const smsTemplates = {
  'attendance-alert': {
    message: '{studentName} was absent from school on {date}. Please call {schoolPhone} if this is unexpected.',
    maxLength: 160,
  },
  'school-closure': {
    message: 'URGENT: {schoolName} is closed {date} due to {reason}. Updates at {url}',
    maxLength: 160,
  },
  'conference-reminder': {
    message: 'Reminder: Conference with {teacherName} tomorrow at {time}. Location: {location}',
    maxLength: 160,
  },
  'emergency-alert': {
    message: 'EMERGENCY: {alertType} at {schoolName}. {instructions}. Do not call school.',
    maxLength: 160,
  },
}
```

### SMS Events

**Domain Events**:
```typescript
interface SMSSent {
  notificationId: string
  to: string
  message: string
  messageSid: string        // Twilio message SID
  tenantId: string
  userId: string
  sentAt: Date
}

interface SMSDelivered {
  notificationId: string
  messageSid: string
  to: string
  status: 'delivered' | 'sent' | 'failed' | 'undelivered'
  timestamp: Date
}

interface SMSFailed {
  notificationId: string
  messageSid: string
  to: string
  errorCode: string
  errorMessage: string
  timestamp: Date
}

interface SMSReplied {
  notificationId: string
  from: string
  to: string               // School phone number
  message: string
  timestamp: Date
}
```

### Twilio Webhooks

**Webhook Configuration**:
```typescript
// POST /api/webhooks/twilio
router.post('/twilio', async (req, res) => {
  // Verify Twilio signature
  const signature = req.headers['x-twilio-signature']
  const url = `${process.env.API_URL}/webhooks/twilio`
  
  if (!twilio.validateRequest(process.env.TWILIO_AUTH_TOKEN, signature, url, req.body)) {
    return res.status(401).json({ error: 'Invalid signature' })
  }
  
  const { MessageSid, MessageStatus, To, ErrorCode, ErrorMessage } = req.body
  
  // Find notification by Twilio message SID
  const notification = await prisma.notification.findFirst({
    where: { externalId: MessageSid },
  })
  
  if (!notification) {
    console.warn('Notification not found for message:', MessageSid)
    return res.status(200).send()
  }
  
  // Publish domain event
  if (MessageStatus === 'delivered') {
    await eventBus.publish('SMSDelivered', {
      notificationId: notification.id,
      messageSid: MessageSid,
      to: To,
      status: MessageStatus,
      timestamp: new Date(),
    })
  } else if (['failed', 'undelivered'].includes(MessageStatus)) {
    await eventBus.publish('SMSFailed', {
      notificationId: notification.id,
      messageSid: MessageSid,
      to: To,
      errorCode: ErrorCode,
      errorMessage: ErrorMessage,
      timestamp: new Date(),
    })
  }
  
  res.status(200).send()
})

// Incoming SMS webhook
router.post('/twilio/incoming', async (req, res) => {
  const { From, To, Body, MessageSid } = req.body
  
  // Handle STOP keyword (opt-out)
  if (Body.trim().toUpperCase() === 'STOP') {
    await prisma.user.update({
      where: { phoneNumber: From },
      data: {
        notificationPreferences: {
          sms: false,
        },
      },
    })
    
    // Reply confirmation
    const twiml = new twilio.twiml.MessagingResponse()
    twiml.message('You have been unsubscribed from SMS notifications. Text START to resubscribe.')
    res.type('text/xml').send(twiml.toString())
    return
  }
  
  // Handle HELP keyword
  if (Body.trim().toUpperCase() === 'HELP') {
    const twiml = new twilio.twiml.MessagingResponse()
    twiml.message('SkillCore School Notifications. Text STOP to unsubscribe. Support: help@skillcore.com')
    res.type('text/xml').send(twiml.toString())
    return
  }
  
  // Publish reply event
  await eventBus.publish('SMSReplied', {
    from: From,
    to: To,
    message: Body,
    timestamp: new Date(),
  })
  
  res.status(200).send()
})
```

### SMS Rate Limiting

**Twilio Limits**:
- 1 message per second per long code
- 100 messages per second per short code
- Daily sending limits vary by account type

**SkillCore Limits**:
- 10 SMS per phone number per hour (non-emergency)
- Unlimited emergency alerts
- Batch send: 100 recipients per batch (staggered)

---

## Channel 3: Push Notifications (Firebase FCM)

### Configuration

**Firebase Setup**:
```typescript
import admin from 'firebase-admin'
import serviceAccount from './firebase-service-account.json'

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
})

const sendPushNotification = async (pushData: PushData) => {
  const message = {
    notification: {
      title: pushData.title,
      body: pushData.body,
      imageUrl: pushData.imageUrl,
    },
    data: pushData.data,            // Custom data payload
    tokens: pushData.tokens,        // Array of FCM tokens
    android: {
      priority: pushData.priority === 'high' ? 'high' : 'normal',
      notification: {
        sound: pushData.priority === 'high' ? 'urgent_alert' : 'default',
        channelId: pushData.channel || 'general',
      },
    },
    apns: {
      payload: {
        aps: {
          sound: pushData.priority === 'high' ? 'urgent_alert.caf' : 'default',
          badge: pushData.badge,
        },
      },
    },
    webpush: {
      notification: {
        icon: '/icon-192.png',
        badge: '/badge-72.png',
      },
    },
  }
  
  const response = await admin.messaging().sendMulticast(message)
  return response
}
```

**Environment Variables**:
```env
FIREBASE_PROJECT_ID=skillcore-prod
FIREBASE_CLIENT_EMAIL=firebase-adminsdk@skillcore-prod.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
```

### Push Notification Channels (Android)

**Channel Configuration**:
```typescript
const notificationChannels = {
  general: {
    id: 'general',
    name: 'General Notifications',
    description: 'General app notifications',
    importance: 'default',
    sound: 'default',
  },
  grades: {
    id: 'grades',
    name: 'Grade Updates',
    description: 'New grades posted',
    importance: 'high',
    sound: 'grade_posted',
  },
  attendance: {
    id: 'attendance',
    name: 'Attendance Alerts',
    description: 'Attendance notifications',
    importance: 'high',
    sound: 'attendance_alert',
  },
  messages: {
    id: 'messages',
    name: 'Messages',
    description: 'New messages from teachers',
    importance: 'high',
    sound: 'message_received',
  },
  emergency: {
    id: 'emergency',
    name: 'Emergency Alerts',
    description: 'Critical emergency notifications',
    importance: 'urgent',
    sound: 'emergency_alert',
  },
}
```

### Push Notification Topics

**Topic Subscriptions**:
```typescript
// Subscribe user to topics
await admin.messaging().subscribeToTopic(fcmTokens, 'school-123')
await admin.messaging().subscribeToTopic(fcmTokens, 'grade-10')
await admin.messaging().subscribeToTopic(fcmTokens, 'class-math-101')

// Send to topic
await admin.messaging().send({
  notification: {
    title: 'School Closure',
    body: 'School closed tomorrow due to weather',
  },
  topic: 'school-123',
})
```

**Topic Naming Convention**:
- `school-{schoolId}` - All users in school
- `grade-{gradeLevel}` - All users in grade level
- `class-{classId}` - All students/parents in class
- `role-{role}` - All users with role (teachers, students, parents)
- `tenant-{tenantId}` - All users in district

### Push Notification Events

**Domain Events**:
```typescript
interface PushNotificationSent {
  notificationId: string
  tokens: string[]
  title: string
  body: string
  messageId: string         // FCM message ID
  tenantId: string
  userId: string
  sentAt: Date
}

interface PushNotificationDelivered {
  notificationId: string
  messageId: string
  token: string
  deliveredAt: Date
}

interface PushNotificationClicked {
  notificationId: string
  token: string
  clickedAt: Date
  actionId?: string         // Notification action button clicked
}

interface PushNotificationFailed {
  notificationId: string
  token: string
  errorCode: string
  errorMessage: string
  failedAt: Date
}
```

### FCM Token Management

**Token Registration**:
```typescript
// Client-side (React app)
import { getMessaging, getToken } from 'firebase/messaging'

const messaging = getMessaging()
const token = await getToken(messaging, {
  vapidKey: process.env.VITE_FIREBASE_VAPID_KEY,
})

// Register token with backend
await apiService.registerFCMToken({ token, platform: 'web' })
```

**Token Storage**:
```prisma
model UserDevice {
  id        String   @id @default(uuid())
  userId    String
  fcmToken  String   @unique
  platform  String   // 'ios', 'android', 'web'
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  user      User     @relation(fields: [userId], references: [id])
}
```

**Token Refresh**:
- Tokens refreshed every 60 days
- Invalid tokens automatically removed
- Failed delivery removes token from database

---

## Notification Orchestration

### Multi-Channel Delivery

**Channel Selection Logic**:
```typescript
async function sendNotification(notification: NotificationData) {
  const user = await prisma.user.findUnique({
    where: { id: notification.userId },
    include: { notificationPreferences: true, devices: true },
  })
  
  const channels = []
  
  // Determine channels based on priority and user preferences
  if (notification.priority === 'emergency') {
    // Emergency: Send to all channels
    channels.push('email', 'sms', 'push')
  } else {
    // Normal: Respect user preferences
    if (user.notificationPreferences.email) {
      channels.push('email')
    }
    if (user.notificationPreferences.sms) {
      channels.push('sms')
    }
    if (user.notificationPreferences.push && user.devices.length > 0) {
      channels.push('push')
    }
  }
  
  // Send to each channel
  const results = await Promise.allSettled([
    channels.includes('email') ? sendEmail(notification) : null,
    channels.includes('sms') ? sendSMS(notification) : null,
    channels.includes('push') ? sendPush(notification) : null,
  ])
  
  return results
}
```

### Fallback Logic

**Channel Fallback**:
```typescript
// If email fails, try SMS for high-priority notifications
if (emailResult.status === 'rejected' && notification.priority === 'high') {
  await sendSMS(notification)
}

// If push fails, try email
if (pushResult.status === 'rejected') {
  await sendEmail(notification)
}
```

### Delivery Tracking

**NotificationDeliveryLog Table**:
```prisma
model NotificationDeliveryLog {
  id             String   @id @default(uuid())
  notificationId String
  channel        String   // 'email', 'sms', 'push'
  status         String   // 'sent', 'delivered', 'failed', 'bounced', 'opened', 'clicked'
  externalId     String?  // SendGrid message ID, Twilio SID, FCM message ID
  errorCode      String?
  errorMessage   String?
  metadata       Json?    // Additional data (IP, user agent, etc.)
  createdAt      DateTime @default(now())
  
  notification   Notification @relation(fields: [notificationId], references: [id])
  
  @@index([notificationId])
  @@index([channel, status])
}
```

---

## Analytics & Reporting

### Delivery Metrics

**GraphQL Query**:
```graphql
query GetNotificationAnalytics($dateRange: DateRangeInput!) {
  notificationAnalytics(dateRange: $dateRange) {
    email {
      sent
      delivered
      bounced
      opened
      clicked
      deliveryRate
      openRate
      clickRate
    }
    sms {
      sent
      delivered
      failed
      deliveryRate
    }
    push {
      sent
      delivered
      clicked
      deliveryRate
      clickRate
    }
    overall {
      totalSent
      totalDelivered
      overallDeliveryRate
    }
  }
}
```

### User Engagement Metrics

**Engagement Tracking**:
- Email open rate by template
- Email click rate by template
- SMS reply rate
- Push notification click rate
- Opt-out rate by channel

**Reporting Dashboard**:
- Daily delivery summary
- Channel performance comparison
- Template effectiveness
- User engagement trends
- Bounce/failure analysis

---

## Best Practices

### Notification Design
1. **Clear Subject/Title**: Descriptive and actionable
2. **Concise Message**: Get to the point quickly
3. **Call to Action**: What should user do next?
4. **Personalization**: Use recipient name and context
5. **Mobile-Friendly**: Optimize for mobile devices

### Delivery Optimization
1. **Batch Sending**: Group notifications to avoid rate limits
2. **Quiet Hours**: Respect local time zones
3. **Deduplication**: Avoid sending duplicate notifications
4. **Priority Routing**: High-priority to fastest channel
5. **Retry Logic**: Retry transient failures

### FERPA Compliance
1. **Data Minimization**: Only include necessary student data
2. **Secure Transmission**: TLS for all channels
3. **Audit Logging**: Log all notification sends
4. **Opt-In Required**: Explicit consent for non-essential notifications
5. **Right to Opt-Out**: Easy unsubscribe mechanism

### Cost Optimization
1. **Email Preferred**: Email cheapest channel (use when possible)
2. **SMS Sparingly**: Reserve for high-priority/emergency
3. **Push Free**: Use push notifications aggressively
4. **Template Reuse**: Avoid custom emails when templates work
5. **Batch Processing**: Group sends to reduce API calls

---

## Related Documentation

- [Queue Architecture](./queues.md) - Notification delivery queues
- [Events Architecture](./events.md) - Notification event system
- [Communication Feature](../features/communication.md) - User-to-user messaging

---

**Production Metrics** (as of November 2025):
- **Daily Notifications**: ~15,000
- **Email**: 12,000/day (80%)
- **SMS**: 500/day (3.3%)
- **Push**: 2,500/day (16.7%)
- **Overall Delivery Rate**: 98.5%
- **Email Open Rate**: 42%
- **Push Click Rate**: 15%
