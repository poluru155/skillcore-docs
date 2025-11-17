# Counselor Portal

**Last Updated**: November 17, 2025  
**Status**: ✅ Production-Ready

---

## Overview

The Counselor Portal provides comprehensive tools for student support, intervention management, special programs coordination, caseload management, and social-emotional learning support. Designed for school counselors, social workers, and student support specialists.

---

## Dashboard

**View**: `/counselor/dashboard`

### Caseload Overview

**Quick Stats**:
- Total students in caseload
- Active interventions count
- Pending IEP/504 reviews
- Crisis alerts (last 24 hours)
- Scheduled appointments today
- Overdue tasks

**Priority Alerts**:
- 🔴 Critical: Self-harm indicators, crisis situations
- 🟡 Urgent: Truancy, failing multiple classes, behavioral referrals
- 🟢 Routine: Scheduled check-ins, routine appointments

### Student Cards

**At-Risk Students** (Auto-filtered):
- Students with 3+ risk factors
- Chronic absenteeism (10%+)
- Failing 2+ classes
- Multiple behavioral incidents
- Social-emotional concerns

Each card shows:
- Student photo and name
- Grade level
- Risk factors
- Active interventions
- Last contact date
- Next scheduled meeting

---

## Caseload Management

**View**: `/counselor/caseload`

### Student Assignment

**Caseload Distribution**:
- Students assigned by grade level
- Alphabetical assignment
- Balanced caseload numbers
- Special program assignments (IEP, 504, ESL)

**Student List**:
- All students in caseload
- Filter by: Grade, Risk level, Program, Intervention tier
- Sort by: Name, Last contact, Risk score
- Search by name or student ID

### Student Profile

**View Profile**: `/counselor/students/:studentId`

**Comprehensive View**:
- Demographics and contact info
- Academic performance (all classes)
- Attendance history
- Behavioral records
- Social-emotional assessments
- Family information
- Medical/health notes (if applicable)
- Support services history
- Contact log

**Quick Actions**:
- Schedule appointment
- Create intervention
- Send message to parent/teacher
- Add contact note
- Refer to services

---

## Interventions (MTSS/RTI)

**View**: `/counselor/interventions`

### Intervention Dashboard

**Active Interventions**:
- Tier 1: Universal supports
- Tier 2: Targeted interventions
- Tier 3: Intensive supports

**Filter Options**:
- By tier
- By intervention type (academic, behavioral, attendance)
- By progress status (on-track, needs adjustment, not working)
- By review date

### Create Intervention

**New Intervention**: `/counselor/interventions/new`

**Intervention Plan Setup**:
1. **Select Student(s)**: Individual or group
2. **Identify Concern**: Academic, behavioral, attendance, social-emotional
3. **Baseline Data**: Current performance metrics
4. **Set Goal**: SMART goal (Specific, Measurable, Achievable, Relevant, Time-bound)
5. **Choose Tier**: Based on severity and previous interventions
6. **Select Strategies**: Evidence-based interventions
7. **Assign Team**: Teachers, specialists, parents involved
8. **Set Review Date**: Typically 4-6 weeks

**Example Goal**:
```
Student: John Doe
Concern: Reading below grade level
Baseline: Reading at 3rd grade level (currently in 5th grade)
Goal: Increase reading level by 1 grade level in 8 weeks
Tier: 2 (Targeted)
Strategies:
  - Small group reading intervention 3x/week
  - Leveled reading materials
  - Parent reading practice nightly
Review Date: January 15, 2026
```

### Intervention Strategies Library

**Academic Interventions**:
- Reading (phonics, comprehension, fluency)
- Math (number sense, operations, problem-solving)
- Writing (organization, mechanics, content)
- Study skills

**Behavioral Interventions**:
- Check-In/Check-Out (CICO)
- Behavior contracts
- Social skills groups
- Restorative practices
- Conflict resolution

**Attendance Interventions**:
- Attendance contracts
- Incentive systems
- Mentoring programs
- Community agency referrals
- Home visits

**Social-Emotional Interventions**:
- Individual counseling
- Group counseling
- Mindfulness training
- Peer support groups
- Crisis intervention

### Progress Monitoring

**Track Progress**: `/counselor/interventions/:id/progress`

**Data Collection**:
- Weekly progress notes
- Quantitative data (grades, attendance %, behavior incidents)
- Qualitative observations
- Student self-reports
- Teacher input

**Progress Visualization**:
- Line graphs showing trend toward goal
- Comparison to baseline
- Benchmark targets
- Goal attainment prediction

**Team Meetings**:
- Schedule intervention team meetings
- Review data with team
- Make data-driven decisions:
  - Continue intervention (working)
  - Modify intervention (needs adjustment)
  - Increase tier (not working, need more support)
  - Exit intervention (goal achieved)

### Intervention Reporting

**Fidelity Tracking**:
- Was intervention implemented as designed?
- Frequency (how many times per week?)
- Duration (minutes per session)
- Quality rating
- Barriers to implementation

**Outcome Reports**:
- Students who achieved goals
- Students who increased tier
- Students who exited interventions
- Average time to goal achievement
- Most effective strategies

---

## Special Programs Coordination

**View**: `/counselor/special-programs`

### IEP Management

**IEP Caseload**: `/counselor/special-programs/iep`

**Counselor Responsibilities**:
- Participate in IEP team meetings
- Provide social-emotional goals/services
- Transition planning (for students 14+)
- Coordinate related services
- Monitor progress on counseling goals

**IEP Review Calendar**:
- Annual reviews due (60/90 days out)
- Triennial evaluations due
- Transition plan updates
- Progress report deadlines

**Transition Services** (Ages 14-22):
- Career assessments
- College planning
- Independent living skills
- Community resources
- Post-secondary goals
- Agency coordination

### Section 504 Plans

**504 Caseload**: `/counselor/special-programs/504`

**Counselor Role**:
- Coordinate 504 evaluations
- Chair 504 team meetings
- Develop accommodation plans
- Monitor implementation
- Annual reviews
- Parent communication

**Common Accommodations**:
- ADHD supports
- Medical conditions
- Mental health accommodations
- Behavioral supports
- Testing accommodations

### ESL/ELL Services

**ESL Students**: `/counselor/special-programs/esl`

**Counselor Support**:
- Cultural transition support
- Family engagement
- Translation services
- Social integration
- Academic counseling
- Language development monitoring

### Gifted & Talented

**Gifted Caseload**: `/counselor/special-programs/gifted`

**Counselor Services**:
- Social-emotional support (perfectionism, anxiety)
- Underachievement counseling
- Differentiation advocacy
- College/career planning
- Dual exceptionality (gifted + disability)

---

## Crisis Management

**View**: `/counselor/crisis`

### Crisis Response

**Crisis Types**:
- Self-harm/suicidal ideation
- Threats of violence
- Abuse/neglect reports
- Substance abuse
- Severe mental health crisis
- Grief/loss
- Trauma response

**Crisis Protocol**:
1. **Immediate Safety**: Ensure student safety
2. **Assessment**: Risk assessment protocol
3. **Notification**: Parents, admin, emergency contacts
4. **Intervention**: Crisis counseling, external referral
5. **Follow-Up**: Safety plan, ongoing monitoring
6. **Documentation**: Detailed crisis notes

**Risk Assessment Tools**:
- Suicide risk assessment (Columbia Protocol)
- Threat assessment
- Substance abuse screening
- Mental health screening

### Safety Plans

**Create Safety Plan**: `/counselor/crisis/safety-plan/new`

**Safety Plan Components**:
- Warning signs (triggers)
- Coping strategies (what helps)
- Support people (who to call)
- Crisis resources (hotlines, emergency services)
- Means restriction (remove access to harm)
- Follow-up schedule

**Monitoring**:
- Daily check-ins (high risk)
- Weekly counseling sessions
- Parent communication
- Teacher observation
- Gradual reduction as risk decreases

---

## Counseling Sessions

**View**: `/counselor/sessions`

### Appointment Scheduling

**Calendar View**:
- Daily/weekly/monthly views
- Time blocks for appointments
- Color-coded by appointment type
- Double-booking prevention
- Reminder notifications

**Appointment Types**:
- Individual counseling (30-45 minutes)
- Group counseling (45-60 minutes)
- Crisis intervention (immediate)
- Parent meetings
- Teacher consultations
- Team meetings

### Session Documentation

**SOAP Notes**:
- **S**ubjective: What student reported
- **O**bjective: Counselor observations
- **A**ssessment: Clinical impressions
- **P**lan: Next steps, interventions

**Note Templates**:
- Initial intake
- Ongoing counseling
- Crisis intervention
- Termination summary

**Confidentiality**:
- FERPA-compliant storage
- Limited access (counselor only)
- Mandatory reporting exceptions
- Parent access restrictions (age-appropriate)

### Group Counseling

**Counseling Groups**: `/counselor/groups`

**Group Types**:
- Social skills training
- Grief/loss support
- Divorce/family changes
- Anger management
- Anxiety management
- Friendship skills
- Study skills

**Group Management**:
- Recruit members
- Schedule sessions
- Track attendance
- Document progress
- Pre/post assessments

---

## College & Career Planning

**View**: `/counselor/college-career`

### College Counseling (High School)

**Senior Timeline**:
- College application deadlines
- Recommendation letter requests
- Transcript requests
- FAFSA completion
- Scholarship applications
- Enrollment deposits

**College Planning Tools**:
- College search (fit, cost, programs)
- Application tracker
- Essay review
- Interview preparation
- Campus visit scheduling

**Naviance Integration** (if applicable):
- Student accounts
- College research
- Career assessments
- Application management

### Career Development

**Career Assessments**:
- Interest inventories (Strong, Holland)
- Skills assessments
- Personality tests (MBTI, Big Five)
- Values clarification

**Career Exploration**:
- Career research
- Job shadowing opportunities
- Internship placement
- Guest speakers
- College major exploration

**Work-Based Learning**:
- Career and Technical Education (CTE) pathways
- Apprenticeships
- Cooperative education
- Youth employment programs

### Academic Advising

**Course Planning**:
- 4-year plans
- Graduation requirements
- College prep track
- AP/Honors recommendations
- Elective selection
- Schedule changes

**Academic Recovery**:
- Credit recovery options
- Summer school planning
- Online course enrollment
- Alternative pathways to graduation

---

## Parent & Family Support

**View**: `/counselor/family-support`

### Parent Consultation

**Parent Meetings**:
- Schedule appointments
- Progress updates
- Intervention planning
- Crisis consultations
- Special education meetings

**Parent Education**:
- Workshops (anxiety, ADHD, study skills)
- Resource library
- Community referrals
- Parenting strategies

### Family Resources

**Community Connections**:
- Mental health services
- Substance abuse treatment
- Food assistance
- Housing support
- Medical/dental services
- Clothing/supplies
- Tutoring services
- After-school programs

**Referral Management**:
- Track referrals made
- Follow-up on connections
- Service coordination
- Release of information forms

---

## Social-Emotional Learning (SEL)

**View**: `/counselor/sel`

### SEL Programs

**Core Competencies** (CASEL Framework):
1. Self-Awareness
2. Self-Management
3. Social Awareness
4. Relationship Skills
5. Responsible Decision-Making

**Classroom Lessons**:
- Lesson library by grade level
- Delivery schedule
- Teacher collaboration
- Student activities
- Assessment tools

**School-Wide Initiatives**:
- Character education
- Bullying prevention
- Kindness campaigns
- Peer mediation
- Restorative justice

### Student Assessments

**SEL Screening**:
- Universal screening (all students)
- Risk identification
- Progress monitoring
- Outcome evaluation

**Assessment Tools**:
- BASC (Behavior Assessment System)
- SAEBRS (Social, Academic, and Emotional Behavior Risk Screener)
- Student self-reports
- Teacher ratings

---

## Behavioral Support

**View**: `/counselor/behavior`

### Behavioral Incidents

**Incident Reports**:
- View all incidents for caseload students
- Analyze patterns (time, location, antecedent)
- Collaborate with administrators
- Develop behavior intervention plans

**Restorative Practices**:
- Mediation sessions
- Restorative circles
- Conflict resolution
- Relationship repair

### Behavior Intervention Plans (BIP)

**Create BIP**: `/counselor/behavior/bip/new`

**BIP Components**:
- Target behaviors (operational definitions)
- Functional behavior assessment (FBA)
- Replacement behaviors
- Antecedent strategies (prevention)
- Teaching strategies
- Consequence strategies
- Data collection plan

**Positive Behavior Supports**:
- Reinforce appropriate behavior
- Token economy systems
- Behavior contracts
- Social skills instruction

---

## Data & Analytics

**View**: `/counselor/analytics`

### Caseload Analytics

**Student Outcomes**:
- Intervention success rates
- Students exiting interventions
- Grade improvement trends
- Attendance improvement
- Behavioral incident reduction

**Equity Analysis**:
- Disproportionality review
- Demographic breakdowns
- Special education referrals
- Discipline data
- Gifted identification

### Counseling Program Evaluation

**Program Metrics**:
- Number of students served
- Types of services provided
- Parent contacts
- Teacher consultations
- Crisis interventions
- Group counseling participation

**ASCA National Model Alignment**:
- Student competencies
- Program planning
- Program implementation
- Program evaluation

---

## Professional Development

**View**: `/counselor/professional-development`

### Continuing Education

**Required Training**:
- Suicide prevention (annually)
- Mandated reporter training
- Crisis intervention
- FERPA compliance

**Professional Growth**:
- Workshops and conferences
- Webinars
- Certification programs (NCC, NCSC)
- Specialized training (trauma, ADHD, autism)

### Supervision & Consultation

**Clinical Supervision**:
- Weekly/biweekly meetings
- Case consultations
- Professional development goals
- Ethical dilemmas
- Self-care

---

## Settings & Tools

**View**: `/counselor/settings`

### Notification Preferences

**Alert Types**:
- Crisis alerts (immediate)
- Intervention due dates
- Appointment reminders
- Report deadlines
- Student check-ins

### Tools & Resources

**Assessment Library**:
- Risk assessments
- Career assessments
- SEL screeners
- Mental health screenings

**Document Templates**:
- SOAP notes
- Safety plans
- Intervention plans
- Parent letters
- Recommendation letters

---

## Mobile App

**Counselor Mobile App**:
- Crisis response on-the-go
- Appointment scheduling
- Quick session notes
- Student profile lookup
- Intervention updates
- Push notifications for alerts

---

## Related Documentation

- [Interventions](../features/interventions.md)
- [Special Programs](../features/special-programs.md)
- [Chat System](../features/chat.md)
- [Attendance System](../features/attendance.md)
