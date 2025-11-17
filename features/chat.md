# Chat System

**Last Updated**: November 16, 2025  
**Status**: ✅ Production-Ready (FERPA-Compliant)

---

## Overview

SkillCore WebApp provides a real-time, FERPA-compliant messaging system for educational stakeholders with role-based access control, content filtering, file attachments, and comprehensive audit trails.

---

## Features

### Core Messaging

**1. Direct Messages (1-on-1)**:
- Teacher ↔ Parent communication
- Teacher ↔ Student communication (logged)
- Counselor ↔ Student communication (FERPA-protected)
- Admin ↔ Staff communication

**2. Group Conversations**:
- Class discussions (teacher + students)
- Team conversations (IEP teams, intervention teams)
- Department chats (teacher collaboration)
- Parent-teacher conferences (multi-participant)

**3. Broadcast Messages**:
- District-wide announcements
- School-wide updates
- Class announcements
- Emergency notifications

### FERPA Compliance

**Access Control**:
```typescript
// Role-based message visibility
function canAccessConversation(user: User, conversation: Conversation): boolean {
  // Teachers can message students in their classes
  if (user.role === 'TEACHER') {
    return conversation.participants.some(p => 
      p.role === 'STUDENT' && isInMyClass(user.id, p.id)
    )
  }

  // Parents can only message their children's teachers
  if (user.role === 'PARENT') {
    const myChildren = getMyChildren(user.id)
    return conversation.participants.every(p =>
      p.role === 'TEACHER' || myChildren.includes(p.id)
    )
  }

  // Students can only message their teachers
  if (user.role === 'STUDENT') {
    return conversation.participants.every(p =>
      p.role === 'TEACHER' || p.id === user.id
    )
  }

  return false
}
```

**Audit Logging**:
```typescript
// Every message action is logged
interface MessageAuditLog {
  id: string
  action: 'SENT' | 'READ' | 'DELETED' | 'EDITED' | 'EXPORTED'
  messageId: string
  conversationId: string
  userId: string
  userRole: UserRole
  timestamp: Date
  ipAddress: string
  metadata: {
    recipientIds?: string[]
    editHistory?: string[]
    deletionReason?: string
  }
}
```

**Data Retention**:
- Active messages: Retained indefinitely
- Deleted messages: Soft delete with 30-day recovery window
- Audit logs: Retained for 7 years (FERPA requirement)
- Exported conversations: Timestamped, immutable records

### Content Filtering

**Profanity Filter**:
```typescript
import Filter from 'bad-words'

const filter = new Filter()

// Student messages are filtered
function filterStudentMessage(content: string, role: UserRole): string {
  if (role === 'STUDENT') {
    return filter.clean(content)
  }
  return content
}
```

**Inappropriate Content Detection**:
```typescript
// Flag for manual review
const flaggedPatterns = [
  /bullying/i,
  /threat/i,
  /self.harm/i,
  // ... additional patterns
]

function detectInappropriateContent(content: string): boolean {
  return flaggedPatterns.some(pattern => pattern.test(content))
}
```

**Auto-Flagging**:
- Profanity in student messages → Notify teacher
- Threats detected → Notify admin + counselor
- Self-harm language → Immediate counselor notification

---

## Architecture

### Database Schema

```prisma
model Conversation {
  id              String               @id @default(uuid())
  type            ConversationType     // DIRECT, GROUP, BROADCAST
  title           String?              // For group conversations
  participants    ConversationParticipant[]
  messages        Message[]
  createdAt       DateTime             @default(now())
  updatedAt       DateTime             @updatedAt
  lastMessageAt   DateTime?
  archivedAt      DateTime?
}

model ConversationParticipant {
  id              String        @id @default(uuid())
  conversationId  String
  userId          String
  role            UserRole
  joinedAt        DateTime      @default(now())
  lastReadAt      DateTime?
  unreadCount     Int           @default(0)
  mutedUntil      DateTime?
  conversation    Conversation  @relation(fields: [conversationId], references: [id])
  user            User          @relation(fields: [userId], references: [id])
}

model Message {
  id              String        @id @default(uuid())
  conversationId  String
  senderId        String
  content         String        @db.Text
  contentFiltered String?       @db.Text  // For student messages
  flagged         Boolean       @default(false)
  flagReason      String?
  attachments     MessageAttachment[]
  readReceipts    MessageReadReceipt[]
  reactions       MessageReaction[]
  editHistory     MessageEdit[]
  deletedAt       DateTime?
  deletedBy       String?
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
  conversation    Conversation  @relation(fields: [conversationId], references: [id])
  sender          User          @relation(fields: [senderId], references: [id])
}

model MessageAttachment {
  id              String        @id @default(uuid())
  messageId       String
  filename        String
  mimetype        String
  size            Int           // Bytes
  url             String        // S3 URL
  uploadedAt      DateTime      @default(now())
  message         Message       @relation(fields: [messageId], references: [id])
}

model MessageReadReceipt {
  id              String        @id @default(uuid())
  messageId       String
  userId          String
  readAt          DateTime      @default(now())
  message         Message       @relation(fields: [messageId], references: [id])
  user            User          @relation(fields: [userId], references: [id])
}

model MessageEdit {
  id              String        @id @default(uuid())
  messageId       String
  previousContent String        @db.Text
  editedAt        DateTime      @default(now())
  editedBy        String
  message         Message       @relation(fields: [messageId], references: [id])
}
```

### Real-Time Updates

**WebSocket Subscriptions**:
```graphql
subscription onNewMessage($conversationId: ID!) {
  messageReceived(conversationId: $conversationId) {
    id
    content
    sender {
      id
      name
      avatar
      role
    }
    createdAt
    attachments {
      id
      filename
      url
    }
  }
}

subscription onTypingIndicator($conversationId: ID!) {
  userTyping(conversationId: $conversationId) {
    userId
    userName
    isTyping
  }
}

subscription onReadReceipt($messageId: ID!) {
  messageRead(messageId: $messageId) {
    userId
    userName
    readAt
  }
}
```

**Redis Pub/Sub**:
```typescript
// Publish typing indicator
async function publishTypingIndicator(
  conversationId: string,
  userId: string,
  isTyping: boolean
) {
  await redis.publish(`chat:typing:${conversationId}`, JSON.stringify({
    userId,
    isTyping,
    timestamp: Date.now()
  }))
}

// Subscribe to typing events
redis.subscribe(`chat:typing:${conversationId}`)
redis.on('message', (channel, message) => {
  const data = JSON.parse(message)
  websocket.send({ type: 'TYPING', data })
})
```

---

## Event System

### Domain Events

```typescript
// Message Lifecycle Events
export class MessageSent extends DomainEvent {
  conversationId: string
  messageId: string
  senderId: string
  recipientIds: string[]
  contentFlagged: boolean
}

export class MessageRead extends DomainEvent {
  messageId: string
  userId: string
  readAt: Date
}

export class MessageDeleted extends DomainEvent {
  messageId: string
  deletedBy: string
  reason?: string
}

export class MessageEdited extends DomainEvent {
  messageId: string
  previousContent: string
  newContent: string
  editedBy: string
}

export class MessageFlagged extends DomainEvent {
  messageId: string
  reason: string
  flaggedBy: string // 'SYSTEM' or userId
  severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL'
}

// Conversation Events
export class ConversationCreated extends DomainEvent {
  conversationId: string
  type: ConversationType
  participantIds: string[]
}

export class ParticipantAdded extends DomainEvent {
  conversationId: string
  userId: string
  addedBy: string
}

export class ParticipantRemoved extends DomainEvent {
  conversationId: string
  userId: string
  removedBy: string
}

// Attachment Events
export class AttachmentUploaded extends DomainEvent {
  messageId: string
  attachmentId: string
  filename: string
  size: number
}

export class AttachmentVirusDetected extends DomainEvent {
  attachmentId: string
  filename: string
  virusName: string
}
```

### Event Handlers

**1. MessageDeliveryHandler**:
```typescript
export class MessageDeliveryHandler implements IEventHandler<MessageSent> {
  async handle(event: MessageSent): Promise<void> {
    // Send push notifications to offline recipients
    const offlineRecipients = await getOfflineUsers(event.recipientIds)
    
    for (const recipient of offlineRecipients) {
      await notificationQueue.add('push-notification', {
        userId: recipient.id,
        title: 'New Message',
        body: `${event.senderName}: ${event.contentPreview}`,
        data: {
          conversationId: event.conversationId,
          messageId: event.messageId
        }
      })
    }
  }
}
```

**2. ContentModerationHandler**:
```typescript
export class ContentModerationHandler implements IEventHandler<MessageFlagged> {
  async handle(event: MessageFlagged): Promise<void> {
    if (event.severity === 'CRITICAL') {
      // Notify admin + counselor immediately
      await notificationService.sendUrgent({
        recipients: ['ADMIN', 'COUNSELOR'],
        title: 'Critical Message Flagged',
        body: `Message contains concerning content: ${event.reason}`,
        link: `/messages/${event.messageId}`
      })
    } else if (event.severity === 'HIGH') {
      // Add to moderation queue
      await moderationQueue.add('review-message', {
        messageId: event.messageId,
        reason: event.reason
      })
    }
  }
}
```

**3. ParentNotificationHandler**:
```typescript
export class ParentNotificationHandler implements IEventHandler<MessageSent> {
  async handle(event: MessageSent): Promise<void> {
    // If teacher messages student, notify parent
    if (event.senderRole === 'TEACHER' && event.recipientRole === 'STUDENT') {
      const parent = await getParentOf(event.recipientIds[0])
      
      if (parent) {
        await notificationQueue.add('email', {
          to: parent.email,
          subject: `${event.senderName} sent a message to ${event.recipientName}`,
          template: 'parent-message-notification',
          data: {
            teacherName: event.senderName,
            studentName: event.recipientName,
            preview: event.contentPreview
          }
        })
      }
    }
  }
}
```

**4. UnreadCountHandler**:
```typescript
export class UnreadCountHandler implements IEventHandler<MessageSent> {
  async handle(event: MessageSent): Promise<void> {
    // Increment unread count for all recipients
    await db.conversationParticipant.updateMany({
      where: {
        conversationId: event.conversationId,
        userId: { in: event.recipientIds }
      },
      data: {
        unreadCount: { increment: 1 }
      }
    })
  }
}
```

**5. ReadReceiptHandler**:
```typescript
export class ReadReceiptHandler implements IEventHandler<MessageRead> {
  async handle(event: MessageRead): Promise<void> {
    // Create read receipt
    await db.messageReadReceipt.create({
      data: {
        messageId: event.messageId,
        userId: event.userId,
        readAt: event.readAt
      }
    })

    // Reset unread count
    await db.conversationParticipant.updateMany({
      where: {
        conversationId: event.conversationId,
        userId: event.userId
      },
      data: {
        unreadCount: 0,
        lastReadAt: event.readAt
      }
    })

    // Notify sender via WebSocket
    await websocket.sendToUser(event.senderId, {
      type: 'MESSAGE_READ',
      data: { messageId: event.messageId, userId: event.userId }
    })
  }
}
```

---

## File Attachments

### Upload Flow

```typescript
// 1. Request signed upload URL
const uploadUrl = await chatService.getUploadUrl({
  filename: 'homework.pdf',
  mimetype: 'application/pdf',
  size: 524288 // 512 KB
})

// 2. Upload directly to S3
await fetch(uploadUrl.url, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': file.type }
})

// 3. Confirm upload
await chatService.sendMessage({
  conversationId: 'conv-123',
  content: 'Here is the homework assignment',
  attachmentIds: [uploadUrl.attachmentId]
})
```

### Security

**Virus Scanning**:
```typescript
import { ClamAV } from 'clamav.js'

async function scanAttachment(file: Buffer): Promise<boolean> {
  const scanner = new ClamAV()
  const result = await scanner.scan(file)
  
  if (result.infected) {
    await eventBus.publish(new AttachmentVirusDetected({
      attachmentId,
      filename,
      virusName: result.viruses[0]
    }))
    throw new Error('File contains virus')
  }
  
  return true
}
```

**File Size Limits**:
- Students: 10 MB max
- Teachers: 50 MB max
- Admins: 100 MB max

**Allowed File Types**:
```typescript
const allowedMimeTypes = [
  // Documents
  'application/pdf',
  'application/msword',
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
  
  // Spreadsheets
  'application/vnd.ms-excel',
  'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
  
  // Images
  'image/jpeg',
  'image/png',
  'image/gif',
  
  // Archives
  'application/zip',
  'application/x-zip-compressed'
]
```

**Access Control**:
```typescript
// Only conversation participants can download attachments
async function getAttachmentUrl(attachmentId: string, userId: string): Promise<string> {
  const attachment = await db.messageAttachment.findUnique({
    where: { id: attachmentId },
    include: {
      message: {
        include: {
          conversation: {
            include: { participants: true }
          }
        }
      }
    }
  })

  const isParticipant = attachment.message.conversation.participants.some(
    p => p.userId === userId
  )

  if (!isParticipant) {
    throw new UnauthorizedError('Not a conversation participant')
  }

  // Generate time-limited signed URL (1 hour expiry)
  return s3.getSignedUrl('getObject', {
    Bucket: 'skillcore-chat-attachments',
    Key: attachment.s3Key,
    Expires: 3600
  })
}
```

---

## GraphQL API

### Queries

```graphql
type Query {
  # Get all conversations for current user
  myConversations(
    limit: Int = 20
    cursor: String
  ): ConversationConnection!

  # Get single conversation
  conversation(id: ID!): Conversation!

  # Get messages in conversation
  conversationMessages(
    conversationId: ID!
    limit: Int = 50
    cursor: String
  ): MessageConnection!

  # Search messages
  searchMessages(
    query: String!
    conversationId: ID
    limit: Int = 20
  ): [Message!]!

  # Get unread message count
  unreadMessageCount: Int!
}
```

### Mutations

```graphql
type Mutation {
  # Create conversation
  createConversation(input: CreateConversationInput!): Conversation!

  # Send message
  sendMessage(input: SendMessageInput!): Message!

  # Edit message (within 5 minutes)
  editMessage(messageId: ID!, content: String!): Message!

  # Delete message
  deleteMessage(messageId: ID!, reason: String): Boolean!

  # Mark messages as read
  markAsRead(conversationId: ID!): Boolean!

  # Add participant to group
  addParticipant(conversationId: ID!, userId: ID!): Conversation!

  # Remove participant
  removeParticipant(conversationId: ID!, userId: ID!): Conversation!

  # Mute conversation
  muteConversation(conversationId: ID!, until: DateTime): Boolean!

  # Request attachment upload URL
  requestUploadUrl(input: AttachmentUploadInput!): UploadUrl!
}
```

### Subscriptions

```graphql
type Subscription {
  # New messages in conversation
  messageReceived(conversationId: ID!): Message!

  # Typing indicators
  userTyping(conversationId: ID!): TypingIndicator!

  # Read receipts
  messageRead(messageId: ID!): ReadReceipt!

  # Conversation updated
  conversationUpdated(conversationId: ID!): Conversation!
}
```

---

## Analytics

### Message Metrics

```typescript
interface MessageAnalytics {
  totalMessages: number
  messagesByRole: {
    TEACHER: number
    STUDENT: number
    PARENT: number
    ADMIN: number
  }
  averageResponseTime: number // seconds
  flaggedMessages: number
  deletedMessages: number
  attachmentCount: number
  conversationCount: number
}
```

### Engagement Tracking

```typescript
// Track teacher-parent engagement
await db.messageAnalytics.create({
  data: {
    metric: 'TEACHER_PARENT_MESSAGES',
    value: messageCount,
    timestamp: new Date(),
    dimensions: {
      teacherId,
      parentId,
      studentId
    }
  }
})
```

---

## Related Documentation

- [Event Architecture](../architecture/events.md) - Event system
- [Notifications](./notifications.md) - Multi-channel notifications
- [Features Index](./README.md) - All features
