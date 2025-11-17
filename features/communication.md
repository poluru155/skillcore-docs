# Communication System

**Last Updated**: November 17, 2025  
**Status**: Production-Ready  
**Bounded Context**: Communication Domain

---

## Overview

The Communication System enables **announcements**, **messaging**, **parent-teacher conferences**, and **emergency alerts** across districts and schools. It supports **multi-channel delivery** (email, SMS, push), **read receipts**, **attachments**, and **FERPA-compliant** message archival.

---

## Key Features

### 1. Announcements

**Announcement Model**:
```prisma
model Announcement {
  id              String            @id @default(uuid())
  
  // Content
  title           String
  body            String            @db.Text
  priority        Priority          @default(normal) // low, normal, high, urgent
  category        String?           // 'academic', 'sports', 'events', 'emergency'
  
  // Author
  authorId        String
  author          User              @relation(fields: [authorId], references: [id])
  
  // Targeting
  targetType      TargetType        // district, school, grade, class, role
  targetIds       String[]          // IDs of targets
  
  // Scheduling
  publishAt       DateTime?         // Scheduled publish time
  expiresAt       DateTime?         // Auto-archive time
  isPinned        Boolean           @default(false)
  
  // Attachments
  attachments     AnnouncementAttachment[]
  
  // Delivery
  deliveryStatus  DeliveryStatus    @default(pending)
  deliveredAt     DateTime?
  
  // Analytics
  viewCount       Int               @default(0)
  readCount       Int               @default(0)
  
  // Multi-tenant
  districtId      String
  schoolId        String?
  
  // Soft delete
  deletedAt       DateTime?
  
  // Timestamps
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt
  
  @@index([districtId])
  @@index([schoolId])
  @@index([targetType])
  @@index([publishAt])
  @@index([priority])
}
```

**Create Announcement**:
```typescript
const createAnnouncement = async (input: CreateAnnouncementInput) => {
  // 1. Validate author has permission
  if (!canCreateAnnouncement(currentUser, input.targetType)) {
    throw new UnauthorizedError('Insufficient permissions')
  }
  
  // 2. Create announcement
  const announcement = await prisma.announcement.create({
    data: {
      title: input.title,
      body: input.body,
      priority: input.priority || 'normal',
      category: input.category,
      authorId: currentUser.id,
      targetType: input.targetType,
      targetIds: input.targetIds,
      publishAt: input.publishAt || new Date(),
      expiresAt: input.expiresAt,
      districtId: currentUser.districtId,
      schoolId: input.schoolId,
      deliveryStatus: input.publishAt ? 'scheduled' : 'pending',
    }
  })
  
  // 3. Process attachments
  if (input.attachments?.length) {
    await Promise.all(
      input.attachments.map(file =>
        uploadAttachment(announcement.id, file)
      )
    )
  }
  
  // 4. Publish immediately or schedule
  if (!input.publishAt || input.publishAt <= new Date()) {
    await deliverAnnouncement(announcement.id)
  } else {
    await scheduleAnnouncementDelivery(announcement.id, input.publishAt)
  }
  
  // 5. Publish event
  await eventBus.publish('announcement.created', {
    announcementId: announcement.id,
    targetType: announcement.targetType,
    priority: announcement.priority,
    districtId: announcement.districtId,
  })
  
  return announcement
}
```

**Deliver Announcement**:
```typescript
const deliverAnnouncement = async (announcementId: string) => {
  const announcement = await prisma.announcement.findUnique({
    where: { id: announcementId },
    include: { author: true },
  })
  
  // 1. Get recipients based on targeting
  const recipients = await getAnnouncementRecipients(announcement)
  
  // 2. Create notification for each recipient
  await Promise.all(
    recipients.map(userId =>
      prisma.notification.create({
        data: {
          type: 'ANNOUNCEMENT',
          title: announcement.title,
          body: announcement.body,
          userId,
          relatedEntityType: 'Announcement',
          relatedEntityId: announcement.id,
          priority: announcement.priority,
          districtId: announcement.districtId,
        }
      })
    )
  )
  
  // 3. Send via appropriate channels based on priority
  if (announcement.priority === 'urgent') {
    // Urgent: Email + SMS + Push
    await sendMultiChannelNotification(recipients, announcement)
  } else if (announcement.priority === 'high') {
    // High: Email + Push
    await sendEmailNotification(recipients, announcement)
    await sendPushNotification(recipients, announcement)
  } else {
    // Normal/Low: In-app + Email
    await sendEmailNotification(recipients, announcement)
  }
  
  // 4. Update delivery status
  await prisma.announcement.update({
    where: { id: announcementId },
    data: {
      deliveryStatus: 'delivered',
      deliveredAt: new Date(),
    }
  })
  
  // 5. Publish event
  await eventBus.publish('announcement.delivered', {
    announcementId,
    recipientCount: recipients.length,
  })
}
```

**Get Recipients by Target**:
```typescript
const getAnnouncementRecipients = async (announcement: Announcement): Promise<string[]> => {
  let userIds: string[] = []
  
  switch (announcement.targetType) {
    case 'district':
      // All users in district
      userIds = await prisma.user.findMany({
        where: { districtId: announcement.districtId, deletedAt: null },
        select: { id: true },
      }).then(users => users.map(u => u.id))
      break
      
    case 'school':
      // All users in specified schools
      userIds = await prisma.user.findMany({
        where: {
          districtId: announcement.districtId,
          schoolId: { in: announcement.targetIds },
          deletedAt: null,
        },
        select: { id: true },
      }).then(users => users.map(u => u.id))
      break
      
    case 'grade':
      // All students in specified grades
      userIds = await prisma.student.findMany({
        where: {
          districtId: announcement.districtId,
          gradeLevel: { in: announcement.targetIds },
          enrollmentStatus: 'active',
          deletedAt: null,
        },
        select: { id: true },
      }).then(students => students.map(s => s.id))
      break
      
    case 'class':
      // All students enrolled in specified classes
      userIds = await prisma.enrollment.findMany({
        where: {
          classId: { in: announcement.targetIds },
          status: 'active',
          deletedAt: null,
        },
        select: { studentId: true },
      }).then(enrollments => enrollments.map(e => e.studentId))
      break
      
    case 'role':
      // All users with specified roles
      userIds = await prisma.user.findMany({
        where: {
          districtId: announcement.districtId,
          role: { in: announcement.targetIds },
          deletedAt: null,
        },
        select: { id: true },
      }).then(users => users.map(u => u.id))
      break
  }
  
  return userIds
}
```

### 2. Direct Messaging

**Message Model**:
```prisma
model Message {
  id                String          @id @default(uuid())
  
  // Content
  subject           String?
  body              String          @db.Text
  messageType       MessageType     @default(direct) // direct, group, announcement
  
  // Participants
  senderId          String
  sender            User            @relation("SentMessages", fields: [senderId], references: [id])
  conversationId    String
  conversation      Conversation    @relation(fields: [conversationId], references: [id])
  
  // Reply chain
  replyToId         String?
  replyTo           Message?        @relation("MessageReplies", fields: [replyToId], references: [id])
  replies           Message[]       @relation("MessageReplies")
  
  // Attachments
  attachments       MessageAttachment[]
  
  // Status
  readBy            MessageRead[]
  
  // Multi-tenant
  districtId        String
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime        @default(now())
  updatedAt         DateTime        @updatedAt
  
  @@index([conversationId])
  @@index([senderId])
  @@index([districtId])
  @@index([createdAt])
}

model Conversation {
  id                String            @id @default(uuid())
  
  // Participants
  participantIds    String[]
  participants      User[]            @relation("ConversationParticipants")
  
  // Type
  type              MessageType       @default(direct)
  
  // Metadata
  subject           String?
  lastMessageAt     DateTime?
  lastMessagePreview String?
  
  // Messages
  messages          Message[]
  
  // Multi-tenant
  districtId        String
  
  // Soft delete
  deletedAt         DateTime?
  
  // Timestamps
  createdAt         DateTime          @default(now())
  updatedAt         DateTime          @updatedAt
  
  @@index([districtId])
  @@index([lastMessageAt])
}

model MessageRead {
  id          String    @id @default(uuid())
  
  messageId   String
  message     Message   @relation(fields: [messageId], references: [id])
  userId      String
  user        User      @relation(fields: [userId], references: [id])
  readAt      DateTime  @default(now())
  
  @@unique([messageId, userId])
  @@index([userId])
}
```

**Send Message**:
```typescript
const sendMessage = async (input: SendMessageInput) => {
  // 1. Verify sender can message recipients
  await verifyMessagingPermissions(currentUser.id, input.recipientIds)
  
  // 2. Find or create conversation
  let conversation = await prisma.conversation.findFirst({
    where: {
      type: 'direct',
      participantIds: {
        hasEvery: [currentUser.id, ...input.recipientIds],
      },
      deletedAt: null,
    }
  })
  
  if (!conversation) {
    conversation = await prisma.conversation.create({
      data: {
        type: input.recipientIds.length > 1 ? 'group' : 'direct',
        participantIds: [currentUser.id, ...input.recipientIds],
        subject: input.subject,
        districtId: currentUser.districtId,
      }
    })
  }
  
  // 3. Create message
  const message = await prisma.message.create({
    data: {
      subject: input.subject,
      body: input.body,
      senderId: currentUser.id,
      conversationId: conversation.id,
      messageType: conversation.type,
      districtId: currentUser.districtId,
    }
  })
  
  // 4. Update conversation
  await prisma.conversation.update({
    where: { id: conversation.id },
    data: {
      lastMessageAt: new Date(),
      lastMessagePreview: message.body.substring(0, 100),
    }
  })
  
  // 5. Send notifications to recipients
  await Promise.all(
    input.recipientIds.map(recipientId =>
      prisma.notification.create({
        data: {
          type: 'MESSAGE_RECEIVED',
          title: `New message from ${currentUser.givenName} ${currentUser.familyName}`,
          body: message.body.substring(0, 100),
          userId: recipientId,
          relatedEntityType: 'Message',
          relatedEntityId: message.id,
          districtId: currentUser.districtId,
        }
      })
    )
  )
  
  // 6. Publish event
  await eventBus.publish('message.sent', {
    messageId: message.id,
    conversationId: conversation.id,
    senderId: currentUser.id,
    recipientIds: input.recipientIds,
  })
  
  return message
}
```

**Mark Message as Read**:
```typescript
const markMessageAsRead = async (messageId: string, userId: string) => {
  await prisma.messageRead.upsert({
    where: {
      messageId_userId: {
        messageId,
        userId,
      }
    },
    create: {
      messageId,
      userId,
      readAt: new Date(),
    },
    update: {
      readAt: new Date(),
    }
  })
  
  await eventBus.publish('message.read', {
    messageId,
    userId,
  })
}
```

### 3. Parent-Teacher Conferences

**Conference Model**:
```prisma
model Conference {
  id              String          @id @default(uuid())
  
  // Participants
  teacherId       String
  teacher         User            @relation("TeacherConferences", fields: [teacherId], references: [id])
  parentId        String
  parent          User            @relation("ParentConferences", fields: [parentId], references: [id])
  studentId       String
  student         Student         @relation(fields: [studentId], references: [id])
  
  // Scheduling
  scheduledAt     DateTime
  duration        Int             @default(30) // minutes
  meetingType     MeetingType     @default(in_person) // in_person, virtual, phone
  location        String?         // Room number or Zoom link
  
  // Status
  status          ConferenceStatus @default(scheduled) // scheduled, confirmed, completed, cancelled
  
  // Notes
  agenda          String?         @db.Text
  notes           String?         @db.Text
  followUpNeeded  Boolean         @default(false)
  
  // Multi-tenant
  districtId      String
  schoolId        String
  
  // Soft delete
  deletedAt       DateTime?
  
  // Timestamps
  createdAt       DateTime        @default(now())
  updatedAt       DateTime        @updatedAt
  
  @@index([teacherId])
  @@index([parentId])
  @@index([studentId])
  @@index([scheduledAt])
  @@index([status])
}
```

**Schedule Conference**:
```typescript
const scheduleConference = async (input: ScheduleConferenceInput) => {
  // 1. Verify teacher availability
  const conflictingConference = await prisma.conference.findFirst({
    where: {
      teacherId: input.teacherId,
      scheduledAt: {
        gte: input.scheduledAt,
        lt: addMinutes(input.scheduledAt, input.duration),
      },
      status: { in: ['scheduled', 'confirmed'] },
      deletedAt: null,
    }
  })
  
  if (conflictingConference) {
    throw new Error('Teacher has conflicting conference at this time')
  }
  
  // 2. Create conference
  const conference = await prisma.conference.create({
    data: {
      teacherId: input.teacherId,
      parentId: input.parentId,
      studentId: input.studentId,
      scheduledAt: input.scheduledAt,
      duration: input.duration || 30,
      meetingType: input.meetingType || 'in_person',
      location: input.location,
      agenda: input.agenda,
      status: 'scheduled',
      districtId: currentUser.districtId,
      schoolId: currentUser.schoolId,
    }
  })
  
  // 3. Send confirmation emails
  await sendConferenceConfirmationEmail(conference)
  
  // 4. Create calendar events (optional)
  await createCalendarEvent(conference)
  
  // 5. Publish event
  await eventBus.publish('conference.scheduled', {
    conferenceId: conference.id,
    teacherId: input.teacherId,
    parentId: input.parentId,
    studentId: input.studentId,
    scheduledAt: input.scheduledAt,
  })
  
  return conference
}
```

### 4. Emergency Alerts

**Emergency Alert**:
```typescript
const sendEmergencyAlert = async (input: EmergencyAlertInput) => {
  // 1. Verify sender has emergency alert permission
  if (!hasPermission(currentUser, 'SEND_EMERGENCY_ALERT')) {
    throw new UnauthorizedError('Insufficient permissions for emergency alerts')
  }
  
  // 2. Create announcement with urgent priority
  const announcement = await prisma.announcement.create({
    data: {
      title: `🚨 EMERGENCY ALERT: ${input.title}`,
      body: input.body,
      priority: 'urgent',
      category: 'emergency',
      authorId: currentUser.id,
      targetType: input.targetType,
      targetIds: input.targetIds,
      publishAt: new Date(),
      districtId: currentUser.districtId,
      schoolId: input.schoolId,
      deliveryStatus: 'pending',
    }
  })
  
  // 3. Get all recipients
  const recipients = await getAnnouncementRecipients(announcement)
  
  // 4. Send via ALL channels immediately (email, SMS, push)
  await Promise.all([
    sendEmergencyEmail(recipients, announcement),
    sendEmergencySMS(recipients, announcement),
    sendEmergencyPush(recipients, announcement),
  ])
  
  // 5. Log emergency alert
  await prisma.auditLog.create({
    data: {
      entityType: 'Announcement',
      entityId: announcement.id,
      action: 'EMERGENCY_ALERT',
      userId: currentUser.id,
      metadata: {
        title: input.title,
        recipientCount: recipients.length,
        targetType: input.targetType,
      },
      districtId: currentUser.districtId,
    }
  })
  
  // 6. Publish event
  await eventBus.publish('emergency_alert.sent', {
    announcementId: announcement.id,
    recipientCount: recipients.length,
    districtId: currentUser.districtId,
  })
  
  return announcement
}
```

---

## GraphQL API

### Queries

**Get Announcements**:
```graphql
query GetAnnouncements($filter: AnnouncementFilter, $orderBy: AnnouncementOrderBy) {
  announcements(filter: $filter, orderBy: $orderBy) {
    edges {
      node {
        id
        title
        body
        priority
        category
        author {
          id
          givenName
          familyName
        }
        publishAt
        expiresAt
        isPinned
        viewCount
        readCount
        attachments {
          id
          fileName
          fileUrl
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Get Conversations**:
```graphql
query GetConversations {
  conversations(orderBy: { field: LAST_MESSAGE_AT, direction: DESC }) {
    edges {
      node {
        id
        type
        subject
        participants {
          id
          givenName
          familyName
          role
        }
        lastMessageAt
        lastMessagePreview
        unreadCount
      }
    }
  }
}
```

**Get Messages**:
```graphql
query GetMessages($conversationId: ID!) {
  messages(conversationId: $conversationId, orderBy: { field: CREATED_AT, direction: ASC }) {
    edges {
      node {
        id
        subject
        body
        sender {
          id
          givenName
          familyName
        }
        createdAt
        readBy {
          userId
          readAt
        }
        attachments {
          id
          fileName
          fileUrl
        }
      }
    }
  }
}
```

### Mutations

**Create Announcement**:
```graphql
mutation CreateAnnouncement($input: CreateAnnouncementInput!) {
  createAnnouncement(input: $input) {
    announcement {
      id
      title
      body
      priority
      targetType
      publishAt
    }
    errors {
      field
      message
    }
  }
}
```

**Send Message**:
```graphql
mutation SendMessage($input: SendMessageInput!) {
  sendMessage(input: $input) {
    message {
      id
      subject
      body
      conversationId
      createdAt
    }
    errors {
      field
      message
    }
  }
}
```

**Schedule Conference**:
```graphql
mutation ScheduleConference($input: ScheduleConferenceInput!) {
  scheduleConference(input: $input) {
    conference {
      id
      teacher {
        id
        givenName
        familyName
      }
      parent {
        id
        givenName
        familyName
      }
      student {
        id
        givenName
        familyName
      }
      scheduledAt
      duration
      meetingType
      location
      status
    }
    errors {
      field
      message
    }
  }
}
```

### Subscriptions

**New Message**:
```graphql
subscription OnNewMessage($userId: ID!) {
  messageReceived(userId: $userId) {
    id
    subject
    body
    sender {
      id
      givenName
      familyName
    }
    conversationId
    createdAt
  }
}
```

**New Announcement**:
```graphql
subscription OnNewAnnouncement($userId: ID!) {
  announcementPublished(userId: $userId) {
    id
    title
    body
    priority
    category
    author {
      givenName
      familyName
    }
  }
}
```

---

## UI Components

### Announcement Card

```typescript
import { Card } from '@/components/ui/card'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Text, Flex, Box } from '@radix-ui/themes'
import { Pin, Eye, Calendar, User } from 'lucide-react'

interface AnnouncementCardProps {
  announcement: Announcement
  onRead?: () => void
}

export function AnnouncementCard({ announcement, onRead }: AnnouncementCardProps) {
  const priorityColors = {
    urgent: 'red',
    high: 'orange',
    normal: 'blue',
    low: 'gray',
  } as const
  
  return (
    <Card className={`p-6 ${announcement.isPinned ? 'border-blue-500 border-2' : ''}`}>
      <Flex direction="column" gap="3">
        {/* Header */}
        <Flex align="center" justify="between">
          <Flex align="center" gap="2">
            {announcement.isPinned && <Pin className="h-4 w-4 text-blue-500" />}
            <Text size="4" weight="bold">{announcement.title}</Text>
          </Flex>
          <Badge color={priorityColors[announcement.priority]} size="1">
            {announcement.priority.toUpperCase()}
          </Badge>
        </Flex>

        {/* Body */}
        <Text size="3">{announcement.body}</Text>

        {/* Metadata */}
        <Flex gap="4" wrap="wrap">
          <Flex align="center" gap="1">
            <User className="h-3 w-3 text-gray-400" />
            <Text size="1" color="gray">
              {announcement.author.givenName} {announcement.author.familyName}
            </Text>
          </Flex>
          <Flex align="center" gap="1">
            <Calendar className="h-3 w-3 text-gray-400" />
            <Text size="1" color="gray">
              {format(announcement.publishAt, 'MMM d, yyyy')}
            </Text>
          </Flex>
          <Flex align="center" gap="1">
            <Eye className="h-3 w-3 text-gray-400" />
            <Text size="1" color="gray">
              {announcement.viewCount} views
            </Text>
          </Flex>
        </Flex>

        {/* Actions */}
        <Flex gap="2">
          {onRead && (
            <Button variant="primary" onClick={onRead}>
              Mark as Read
            </Button>
          )}
          {announcement.attachments.length > 0 && (
            <Button variant="outline">
              View Attachments ({announcement.attachments.length})
            </Button>
          )}
        </Flex>
      </Flex>
    </Card>
  )
}
```

### Message Thread

```typescript
import { Card } from '@/components/ui/card'
import { Input } from '@/components/ui/input'
import { Button } from '@/components/ui/button'
import { Text, Flex, Box } from '@radix-ui/themes'
import { Send } from 'lucide-react'

interface MessageThreadProps {
  conversation: Conversation
  messages: Message[]
  onSendMessage: (body: string) => void
}

export function MessageThread({ conversation, messages, onSendMessage }: MessageThreadProps) {
  const [newMessage, setNewMessage] = useState('')
  
  const handleSend = () => {
    if (newMessage.trim()) {
      onSendMessage(newMessage)
      setNewMessage('')
    }
  }
  
  return (
    <Card className="p-6 flex flex-col h-full">
      {/* Header */}
      <Box className="border-b pb-4 mb-4">
        <Text size="4" weight="bold">{conversation.subject}</Text>
        <Text size="2" color="gray">
          {conversation.participants.map(p => `${p.givenName} ${p.familyName}`).join(', ')}
        </Text>
      </Box>

      {/* Messages */}
      <Box className="flex-1 overflow-y-auto space-y-4 mb-4">
        {messages.map(message => (
          <Flex
            key={message.id}
            direction="column"
            gap="1"
            className={`${
              message.senderId === currentUser.id
                ? 'items-end'
                : 'items-start'
            }`}
          >
            <Card
              className={`p-3 max-w-[70%] ${
                message.senderId === currentUser.id
                  ? 'bg-blue-500 text-white'
                  : 'bg-gray-100'
              }`}
            >
              <Text size="2">{message.body}</Text>
            </Card>
            <Text size="1" color="gray">
              {format(message.createdAt, 'MMM d, h:mm a')}
              {message.readBy.length > 0 && ' · Read'}
            </Text>
          </Flex>
        ))}
      </Box>

      {/* Input */}
      <Flex gap="2">
        <Input
          placeholder="Type a message..."
          value={newMessage}
          onChange={e => setNewMessage(e.target.value)}
          onKeyPress={e => e.key === 'Enter' && handleSend()}
          className="flex-1"
        />
        <Button onClick={handleSend} disabled={!newMessage.trim()}>
          <Send className="h-4 w-4" />
        </Button>
      </Flex>
    </Card>
  )
}
```

---

## Domain Events

```typescript
// Announcements
'announcement.created'          // New announcement created
'announcement.published'        // Announcement published to recipients
'announcement.delivered'        // Announcement delivered via channels
'announcement.viewed'           // User viewed announcement
'announcement.expired'          // Announcement expired

// Messages
'message.sent'                  // New message sent
'message.delivered'             // Message delivered to recipient
'message.read'                  // Message read by recipient

// Conferences
'conference.scheduled'          // Conference scheduled
'conference.confirmed'          // Conference confirmed by parent
'conference.completed'          // Conference completed
'conference.cancelled'          // Conference cancelled

// Emergency
'emergency_alert.sent'          // Emergency alert broadcast
```

---

## Related Documentation

- [Notifications](../architecture/notifications.md) - Multi-channel delivery
- [Queues](../architecture/queues.md) - Background message processing
- [Security](../architecture/security.md) - Message encryption

---

**Communication Statistics** (as of November 2025):
- **Total Announcements**: 450K
- **Total Messages**: 12M
- **Conferences Scheduled**: 85K
- **Emergency Alerts**: 1,250
- **Average Response Time**: 4.5 hours
