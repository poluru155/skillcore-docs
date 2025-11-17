# Intervention System (MTSS/RTI)

**Last Updated**: November 17, 2025  
**Status**: ✅ Production-Ready

---

## Overview

The Intervention System implements the Multi-Tiered System of Supports (MTSS) and Response to Intervention (RTI) frameworks for systematic, data-driven student support across academic, behavioral, attendance, and social-emotional domains.

---

## MTSS/RTI Framework

### Three-Tier Model

**Tier 1 - Universal Supports** (80% of students):
- High-quality core instruction
- Differentiation for all students
- Universal screening
- School-wide positive behavior supports (PBIS)
- Progress monitoring for at-risk students

**Tier 2 - Targeted Interventions** (15% of students):
- Small group instruction (3-5 students)
- Evidence-based interventions
- Progress monitoring (weekly/biweekly)
- Increased intensity/frequency
- Parent communication

**Tier 3 - Intensive Supports** (5% of students):
- Individual or very small group (1-2 students)
- Highly specialized interventions
- Frequent progress monitoring (2-3x/week)
- Comprehensive evaluation
- Possible special education referral

---

## Intervention Types

### Academic Interventions

**Reading/Literacy**:
```typescript
const readingInterventions = {
  tier1: [
    'Differentiated reading groups',
    'Guided reading',
    'Independent reading time',
    'Literacy centers'
  ],
  tier2: [
    'Leveled Literacy Intervention (LLI)',
    'Reading Mastery',
    'Corrective Reading',
    'Wilson Reading System',
    'Orton-Gillingham approach'
  ],
  tier3: [
    'One-on-one tutoring',
    'Intensive phonics instruction',
    'Reading specialist services',
    'Assistive technology (text-to-speech)'
  ]
}
```

**Math**:
```typescript
const mathInterventions = {
  tier1: [
    'Math workshop model',
    'Number talks',
    'Math games and manipulatives',
    'Problem-solving strategies'
  ],
  tier2: [
    'Number Worlds',
    'Math Recovery',
    'TouchMath',
    'Do The Math (Marilyn Burns)',
    'Small group targeted instruction'
  ],
  tier3: [
    'Individualized math tutoring',
    'Concrete-Representational-Abstract (CRA)',
    'Computer-assisted instruction',
    'Math specialist services'
  ]
}
```

**Writing**:
- Self-Regulated Strategy Development (SRSD)
- Step Up to Writing
- 6+1 Trait Writing
- Graphic organizers

### Behavioral Interventions

**Tier 1** (School-Wide):
- PBIS matrix (expectations in all settings)
- Positive reinforcement systems
- Classroom management strategies
- Social-emotional learning curriculum

**Tier 2** (Small Group):
- Check-In/Check-Out (CICO)
- Social skills groups
- Behavior contracts
- Mentoring programs
- Self-monitoring strategies

**Tier 3** (Individual):
- Functional Behavior Assessment (FBA)
- Behavior Intervention Plan (BIP)
- Individual counseling
- Wraparound services
- Alternative placement

### Attendance Interventions

**Tier 1**:
- Attendance tracking and reporting
- Parent communication
- Incentive programs
- School climate improvements

**Tier 2**:
- Attendance contracts
- Daily check-ins
- Mentoring
- Targeted parent outreach
- Community agency referrals

**Tier 3**:
- Home visits
- Truancy court involvement
- Intensive case management
- Removal of barriers (transportation, food, clothing)

### Social-Emotional Interventions

**Tier 1**:
- SEL curriculum (CASEL framework)
- Mindfulness practices
- Classroom community building
- Conflict resolution training

**Tier 2**:
- Lunch bunch groups
- Friendship skills groups
- Anger management groups
- Anxiety/stress management groups

**Tier 3**:
- Individual therapy
- Crisis intervention
- Safety planning
- External mental health referrals

---

## Intervention Workflow

### 1. Identification

**Universal Screening**:
```typescript
interface UniversalScreening {
  subject: 'READING' | 'MATH' | 'BEHAVIOR' | 'SEL'
  frequency: 'FALL' | 'WINTER' | 'SPRING'  // 3x per year
  tool: string  // DIBELS, AIMSweb, BASC, etc.
  cutScore: number
  students: {
    studentId: string
    score: number
    atRisk: boolean
  }[]
}
```

**Referral Sources**:
- Universal screening results
- Teacher concern
- Parent request
- Counselor referral
- Grade/attendance triggers
- Behavioral incidents

### 2. Team Formation

**Intervention Team Members**:
```typescript
interface InterventionTeam {
  interventionId: string
  members: {
    userId: string
    role: 'COORDINATOR' | 'TEACHER' | 'PARENT' | 'SPECIALIST' | 'ADMINISTRATOR'
    responsibilities: string[]
  }[]
}
```

**Typical Team**:
- Classroom teacher (implements intervention)
- Intervention specialist/coordinator
- Parent/guardian
- School counselor
- Special education teacher (if applicable)
- Principal/assistant principal

### 3. Problem Definition

**Problem Statement**:
```typescript
interface ProblemDefinition {
  studentId: string
  area: 'ACADEMIC' | 'BEHAVIORAL' | 'ATTENDANCE' | 'SOCIAL_EMOTIONAL'
  specificConcern: string
  baselineData: {
    metric: string
    value: number
    date: Date
    comparisonBenchmark?: number
  }
  gapAnalysis: string  // How far below expectation
}
```

**Example**:
```
Student: John Doe
Area: Academic - Reading
Specific Concern: Reading fluency below grade level
Baseline Data:
  - Current ORF (Oral Reading Fluency): 45 words/minute
  - Grade 3 benchmark: 90 words/minute
  - Gap: 45 words below benchmark (50% of expected)
  - Measured: September 15, 2025
```

### 4. Goal Setting (SMART Goals)

**SMART Goal Structure**:
```typescript
interface SMARTGoal {
  specific: string      // What exactly will be achieved
  measurable: string    // How will we measure success
  achievable: boolean   // Is this realistic given resources
  relevant: string      // Why is this important
  timeBound: Date       // When will goal be achieved
}
```

**Example**:
```
Specific: John will increase reading fluency
Measurable: From 45 WPM to 70 WPM (25 word gain)
Achievable: Yes, with daily 20-minute intervention
Relevant: Reading fluency impacts comprehension and confidence
Time-Bound: By December 15, 2025 (12 weeks)
```

### 5. Intervention Selection

**Evidence-Based Criteria**:
```typescript
interface InterventionStrategy {
  id: string
  name: string
  tier: 1 | 2 | 3
  targetArea: string
  researchBasis: string  // Citation to research studies
  successRate: number    // Percentage of students who respond
  duration: string       // Typical intervention length
  frequency: string      // How often (daily, 3x/week)
  sessionLength: number  // Minutes per session
  groupSize: number      // 1 (individual), 3-5 (small group), 20+ (class)
  materials: string[]
  training Required: boolean
}
```

**Intervention Plan**:
```typescript
interface InterventionPlan {
  id: string
  studentId: string
  tier: 1 | 2 | 3
  startDate: Date
  reviewDate: Date  // Typically 4-8 weeks
  goal: SMARTGoal
  strategies: InterventionStrategy[]
  schedule: {
    frequency: string  // "Daily", "3x per week"
    duration: number   // Minutes per session
    time: string       // "9:00 AM - 9:20 AM"
    location: string   // "Resource room", "Classroom"
    provider: string   // Teacher implementing
  }
  progressMonitoring: {
    tool: string
    frequency: string  // Weekly, biweekly
    day: string        // "Every Monday"
  }
}
```

### 6. Implementation

**Fidelity Tracking**:
```typescript
interface FidelityCheck {
  interventionId: string
  date: Date
  implementedAsPlanned: boolean
  frequency: number      // How many sessions this week
  duration: number       // Average minutes per session
  quality: 1 | 2 | 3 | 4 | 5  // Observer rating
  barriers: string[]
  notes: string
}
```

**Implementation Checklist**:
- [ ] Materials prepared
- [ ] Schedule communicated to all team members
- [ ] Student understands intervention purpose
- [ ] Parent notified of intervention start
- [ ] Progress monitoring tool ready
- [ ] Data collection system in place

### 7. Progress Monitoring

**Data Collection**:
```typescript
interface ProgressMonitoringData {
  interventionId: string
  date: Date
  measure: string        // What was measured
  score: number
  notes: string
  collectedBy: string
}
```

**Progress Monitoring Schedule**:
- **Tier 1**: Monthly or quarterly (universal screening)
- **Tier 2**: Weekly or biweekly
- **Tier 3**: 2-3 times per week

**Progress Analysis**:
```typescript
interface ProgressAnalysis {
  trendDirection: 'POSITIVE' | 'FLAT' | 'NEGATIVE'
  rateOfImprovement: number  // Points gained per week
  onTrackToGoal: boolean
  projectedEndDate: Date
  recommendation: 'CONTINUE' | 'MODIFY' | 'INTENSIFY' | 'EXIT'
}
```

### 8. Team Review Meeting

**Meeting Agenda**:
1. Review baseline data
2. Review progress monitoring data
3. Analyze trend (improving, flat, declining)
4. Discuss fidelity of implementation
5. Make data-driven decision

**Decision Rules**:
```typescript
// After 6-8 weeks of intervention
if (student.achievedGoal) {
  return 'EXIT_INTERVENTION'
} else if (student.makingAdequateProgress) {
  return 'CONTINUE_INTERVENTION'
} else if (student.makingMinimalProgress) {
  return 'MODIFY_INTERVENTION'
} else if (student.makingNoProgress) {
  return 'INTENSIFY_INTERVENTION'  // Move to next tier
}
```

**Possible Outcomes**:
- **Continue**: Keep intervention as is, goal not yet met but making progress
- **Modify**: Change strategy, frequency, or provider
- **Intensify**: Move to next tier (Tier 1 → Tier 2 → Tier 3)
- **Exit**: Goal achieved, return to universal supports
- **Refer**: Special education evaluation (if Tier 3 ineffective)

---

## Special Education Referral

### When to Refer

**Criteria for Referral**:
- Significant gap between performance and peers (1.5-2 years behind)
- Inadequate progress after intensive Tier 3 intervention (minimum 12-16 weeks)
- Suspected disability (learning disability, ADHD, autism, etc.)
- Parent request for evaluation

**Referral Process**:
1. Document all intervention efforts (Tier 1, 2, 3)
2. Compile progress monitoring data
3. Hold team meeting with parent
4. Obtain parent consent for evaluation
5. Conduct comprehensive evaluation (60 days)
6. Hold eligibility meeting
7. Develop IEP if eligible

---

## Event System

### Domain Events

```typescript
export class InterventionCreated extends DomainEvent {
  interventionId: string
  studentId: string
  tier: 1 | 2 | 3
  area: string
  goal: string
  createdBy: string
}

export class InterventionProgressUpdated extends DomainEvent {
  interventionId: string
  studentId: string
  date: Date
  score: number
  trend: 'POSITIVE' | 'FLAT' | 'NEGATIVE'
}

export class InterventionReviewScheduled extends DomainEvent {
  interventionId: string
  reviewDate: Date
  teamMembers: string[]
}

export class InterventionExited extends DomainEvent {
  interventionId: string
  studentId: string
  exitReason: 'GOAL_ACHIEVED' | 'MOVED_TO_HIGHER_TIER' | 'REFERRED_TO_SPED' | 'OTHER'
  finalData: ProgressMonitoringData
}

export class InterventionModified extends DomainEvent {
  interventionId: string
  changesMade: string[]
  modifiedBy: string
}
```

### Event Handlers

**1. InterventionNotificationHandler**:
```typescript
export class InterventionNotificationHandler implements IEventHandler<InterventionCreated> {
  async handle(event: InterventionCreated): Promise<void> {
    const student = await getStudent(event.studentId)
    const parent = await getParentOf(event.studentId)

    // Notify parent
    await notificationQueue.add('email', {
      to: parent.email,
      subject: `Support Plan Created for ${student.name}`,
      template: 'intervention-created',
      data: {
        studentName: student.name,
        tier: event.tier,
        area: event.area,
        goal: event.goal
      }
    })

    // Notify intervention team
    const team = await getInterventionTeam(event.interventionId)
    for (const member of team.members) {
      await notificationQueue.add('email', {
        to: member.email,
        subject: `New Intervention: ${student.name}`,
        template: 'team-member-notification'
      })
    }
  }
}
```

**2. ProgressAlertHandler**:
```typescript
export class ProgressAlertHandler implements IEventHandler<InterventionProgressUpdated> {
  async handle(event: InterventionProgressUpdated): Promise<void> {
    const intervention = await getIntervention(event.interventionId)
    
    // Alert if no progress after 4 weeks
    const weeksSinceStart = getWeeksBetween(intervention.startDate, event.date)
    if (weeksSinceStart >= 4 && event.trend === 'FLAT') {
      await notificationService.send({
        recipientRole: 'COUNSELOR',
        title: 'Intervention Not Working',
        body: `${event.studentName}'s intervention showing no progress after 4 weeks`,
        priority: 'HIGH',
        link: `/interventions/${event.interventionId}`
      })
    }
  }
}
```

---

## GraphQL API

```graphql
type Query {
  # Get all interventions for student
  studentInterventions(studentId: ID!): [Intervention!]!

  # Get intervention details
  intervention(id: ID!): Intervention!

  # Get progress monitoring data
  interventionProgress(interventionId: ID!): [ProgressData!]!

  # Get intervention strategies library
  interventionStrategies(
    tier: Int
    area: String
  ): [InterventionStrategy!]!
}

type Mutation {
  # Create intervention
  createIntervention(input: CreateInterventionInput!): Intervention!

  # Update intervention
  updateIntervention(id: ID!, input: UpdateInterventionInput!): Intervention!

  # Add progress data
  addProgressData(input: ProgressDataInput!): ProgressData!

  # Schedule review meeting
  scheduleReview(interventionId: ID!, date: DateTime!): Meeting!

  # Exit intervention
  exitIntervention(id: ID!, reason: String!): Intervention!

  # Modify intervention
  modifyIntervention(id: ID!, changes: String!): Intervention!
}
```

---

## Reporting

### Intervention Reports

**Student Intervention Report**:
- All interventions (current and past)
- Progress graphs
- Tier history
- Outcome summary

**School-Wide Report**:
- Total students in each tier
- Percentage responding to intervention
- Average time to goal achievement
- Most effective strategies

**Equity Analysis**:
- Disproportionality by race/ethnicity
- Special education referral rates
- Exit rates by demographic group

---

## Related Documentation

- [Special Programs](./special-programs.md)
- [Gradebook](./gradebook.md)
- [Attendance](./attendance.md)
- [Counselor Portal](../roles/counselor.md)
