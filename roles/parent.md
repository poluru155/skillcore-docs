# Parent Portal

**Last Updated**: November 17, 2025  
**Status**: ✅ Production-Ready

---

## Overview

The Parent Portal provides comprehensive access to student academic performance, attendance, behavior, communication with teachers, and school updates. Designed for busy parents with multi-child households and mobile-first access.

---

## Dashboard

**View**: `/parent/dashboard`

### Multi-Child Overview

**Family Dashboard**:
- All children displayed with photo tiles
- Quick stats for each child:
  - Current GPA
  - Attendance rate (last 30 days)
  - Missing assignments count
  - Unread teacher messages
  - Recent alerts

**Alert Summary**:
- 🔴 Urgent: Failing grades, excessive absences, discipline issues
- 🟡 Attention: Declining grades, missing assignments
- 🟢 Positive: Honor roll, improved grades, perfect attendance

### Individual Child View

Click on child to see detailed dashboard:
- Current grades in all classes
- Today's schedule
- Upcoming assignments
- Recent attendance
- Teacher messages
- Behavioral notes

---

## Grades & Academic Progress

**View**: `/parent/grades`

### Current Grades

**Grade Overview**:
- List all classes by child
- Current percentage and letter grade
- Teacher name
- Class period
- Trend indicator (↗ ↘ →)
- Last updated timestamp

**Class Details**: `/parent/grades/:studentId/:classId`

**Detailed Breakdown**:
- Assignment list with scores
- Category weights and performance
- Missing assignments highlighted
- Teacher comments
- Standards mastery (if applicable)

**Grade Trends**:
- Line chart showing grade over time
- Term-by-term comparison
- Historical performance

### Grade Notifications

**Automatic Alerts**:
- New grade posted (real-time or daily digest)
- Failing grade (< 60%)
- Significant drop (> 10% decrease)
- Missing assignment
- Late submission

**Notification Channels**:
- Email (customizable frequency)
- SMS text alerts
- Push notifications (mobile app)
- Weekly summary email

### Report Cards

**Access Report Cards**:
- Current term report card
- Previous terms
- Final grades
- GPA history
- Credits earned
- Teacher comments

**Download Options**:
- PDF for printing
- Email to self
- Share with family members

---

## Attendance

**View**: `/parent/attendance`

### Attendance Overview

**Multi-Child View**:
- Attendance rate for each child
- Recent absences/tardies
- Comparison to school average
- Alert indicators

**Individual Attendance**: `/parent/attendance/:studentId`

**Calendar View**:
- Color-coded attendance:
  - 🟢 Present
  - 🔴 Absent
  - 🟡 Tardy
  - 🔵 Excused
  - ⚪ Weekend/Holiday

**Attendance Summary**:
- Total days attended
- Total absences (excused/unexcused)
- Tardy count
- Attendance percentage
- Comparison to previous year

### Absence Notifications

**Real-Time Alerts**:
- Daily absence notification (sent by 10 AM)
- Tardy notification
- Early dismissal confirmation

**Notification Content**:
```
SkillCore Alert: [Child Name] was marked absent today.

Date: November 17, 2025
Status: Unexcused Absent
Period: Homeroom

Current Attendance Rate: 94.5%

If this is an error or needs to be excused, please contact the school office.

Reply HELP for assistance.
```

### Excused Absence Process

**Submit Excuse**:
1. Select absence date
2. Choose reason (illness, appointment, family emergency)
3. Upload documentation (doctor's note, etc.)
4. Submit for approval

**Status Tracking**:
- Pending review
- Approved (changed to excused)
- Denied (remains unexcused)

### Attendance Interventions

**If Child Has Attendance Issues**:
- View intervention plan
- Goals and strategies
- Progress updates
- Team meeting dates
- How to support at home

---

## Assignments & Homework

**View**: `/parent/assignments`

### Assignment Tracking

**Upcoming Assignments**:
- All children's upcoming work
- Due dates and times
- Class and teacher
- Points possible
- Submission status

**Missing/Overdue**:
- Highlighted in red
- Days overdue
- Impact on grade
- Late submission policy

**Completed**:
- Submission date
- Grade received (if posted)
- Teacher feedback

### Assignment Details

**View Assignment**: `/parent/assignments/:id`

**Information Available**:
- Assignment description
- Due date
- Class and teacher
- Points possible
- Resources/materials needed
- Submission requirements
- Student submission status
- Grade and feedback (if graded)

### Homework Support

**Help Your Child**:
- View assignment instructions
- Access learning resources
- Contact teacher with questions
- Monitor completion progress

**Homework Checklist**:
- Generate daily/weekly homework list
- Print for home use
- Check off completed items
- Track completion rate

---

## Communication

**View**: `/parent/messages`

### Teacher Communication

**Message Inbox**:
- Messages from all teachers
- Organized by child
- Read/unread status
- Priority indicators
- Quick reply

**Start Conversation**:
1. Select child
2. Select teacher
3. Choose topic (grades, behavior, attendance, general)
4. Write message
5. Send

**Message Features**:
- File attachments
- Translation (multilingual support)
- Read receipts
- Message history

### Conference Scheduling

**Parent-Teacher Conferences**: `/parent/conferences`

**Schedule Conference**:
1. Select teacher
2. View available time slots
3. Choose date/time
4. Select virtual or in-person
5. Add agenda topics
6. Confirm appointment

**Conference Management**:
- Automatic reminders (email, SMS)
- Virtual meeting link (Zoom, Teams)
- Reschedule/cancel option
- Post-conference notes from teacher

### School Communications

**Announcements**:
- School-wide updates
- Classroom newsletters
- District news
- Event notifications

**Emergency Alerts**:
- School closures
- Safety notifications
- Weather delays
- Urgent updates

---

## Behavior & Discipline

**View**: `/parent/behavior`

### Behavioral Reports

**Positive Behaviors**:
- Recognition awards
- Citizenship highlights
- Improvement notes
- Teacher commendations

**Concerns**:
- Disciplinary incidents
- Office referrals
- Detentions
- Suspensions

**Incident Details**:
- Date and time
- Location
- Description
- Action taken
- Teacher/admin notes
- Parent signature required

### Behavior Interventions

**Support Plans**:
- If child receiving behavioral support
- Goals and strategies
- Progress monitoring
- Parent involvement activities
- Resources for home

---

## Special Programs

**View**: `/parent/special-programs`

### IEP (Individualized Education Program)

**View IEP**: `/parent/special-programs/iep/:studentId`

**Parent Access**:
- Current IEP document
- Goals and objectives
- Accommodations and modifications
- Service minutes
- Progress reports (quarterly)
- Annual review date
- Team member contacts

**Parent Rights**:
- FERPA rights information
- Procedural safeguards
- Request IEP meeting
- Request evaluations
- Dispute resolution

### Section 504 Plan

**View 504 Plan**: `/parent/special-programs/504/:studentId`

**Plan Details**:
- Accommodations
- Review schedule
- Implementation notes
- Contact coordinator

### ESL/ELL Services

**View ESL Profile**: `/parent/special-programs/esl/:studentId`

**Information**:
- English proficiency level
- WIDA scores
- Service model (push-in, pull-out)
- Progress toward fluency
- Home language support

### Gifted & Talented

**View Gifted Services**: `/parent/special-programs/gifted/:studentId`

**Program Details**:
- Areas of giftedness
- Differentiation plan
- Enrichment opportunities
- Advanced coursework
- Competitions and events

---

## Assessments & Testing

**View**: `/parent/assessments`

### Test Calendar

**Upcoming Assessments**:
- Class tests and quizzes
- Standardized tests (state testing)
- Benchmark assessments
- AP/IB exams (high school)

**Test Preparation**:
- Study resources
- Practice tests
- Recommended study time
- Testing accommodations

### Test Results

**Class Assessments**:
- Recent test scores
- Class averages (for context)
- Standards assessed
- Areas of strength/weakness

**Standardized Test Results**:
- State assessment scores
- Percentile rankings
- Proficiency levels
- Historical comparison
- District/state averages

---

## Student Wellness

**View**: `/parent/wellness`

### Health Information

**Health Records**:
- Immunization records
- Medical conditions
- Allergies
- Medications administered at school
- Health office visits

**Update Health Info**:
- Submit new documentation
- Update emergency contacts
- Report new allergies/conditions

### Counseling Services

**Student Support**:
- Counselor contact information
- Schedule appointment
- Support services available
- Crisis resources

**Intervention Programs**:
- Academic interventions
- Behavioral supports
- Social-emotional learning
- Peer mentoring

---

## School Calendar & Events

**View**: `/parent/calendar`

### School Calendar

**Important Dates**:
- School holidays
- Early dismissal days
- Professional development (no school)
- Parent-teacher conference days
- Report card distribution
- Testing windows

**Events**:
- School events (concerts, plays, games)
- Parent involvement opportunities
- Fundraisers
- Field trips

### Family Engagement

**Volunteer Opportunities**:
- Classroom volunteer
- Field trip chaperone
- Event support
- PTA/PTO involvement

**Parent Education**:
- Workshops (helping with homework, college prep)
- Technology training
- Parent university courses

---

## Payments & Forms

**View**: `/parent/payments`

### School Fees

**Fee Management**:
- View outstanding balances
- Payment history
- Pay online (credit card, ACH)
- Payment plans

**Fee Types**:
- Lunch account
- Activity fees
- Athletic fees
- Field trip costs
- School supplies

### Forms & Documents

**Required Forms**:
- Emergency contact updates
- Field trip permissions
- Medical authorization
- Photo/media release
- Technology acceptable use

**Form Status**:
- Pending signature
- Submitted
- Approved/Denied
- Renewal needed

**E-Signature**:
- Sign forms electronically
- Track submission
- Receive confirmation

---

## Transportation

**View**: `/parent/transportation` (if applicable)

### Bus Information

**Bus Details**:
- Bus number
- Driver name
- Route number
- Pickup time and location
- Drop-off time and location

**Real-Time Tracking**:
- Bus location (GPS)
- Estimated arrival time
- Delay notifications

**Transportation Changes**:
- Request change in transportation
- Add emergency contact pickup
- Report bus issues

---

## Lunch & Nutrition

**View**: `/parent/lunch`

### Lunch Account

**Account Management**:
- Current balance
- Add funds online
- Set low balance alerts
- Transaction history

**Meal Program**:
- Free/reduced lunch status
- Apply for meal benefits
- Dietary restrictions on file

### Lunch Menu

**Daily Menu**:
- View weekly menu
- Nutritional information
- Allergen information
- Pre-order meals (if available)

---

## College & Career (High School)

**View**: `/parent/college-career`

### College Planning

**College Readiness**:
- GPA and class rank
- Course rigor (AP, Honors)
- Standardized test scores (SAT, ACT, PSAT)
- College application timeline

**Financial Aid**:
- FAFSA information
- Scholarship opportunities
- Financial aid workshops
- Net price calculators

### Career Exploration

**Career Planning**:
- Career assessments
- Career pathways
- Work-based learning opportunities
- Apprenticeships

---

## Analytics & Insights

**View**: `/parent/analytics`

### Student Progress

**Academic Trends**:
- Grade trends over time
- GPA progression
- Subject strengths/weaknesses
- Comparison to previous years

**Engagement Metrics**:
- Assignment completion rate
- Assessment performance
- Participation indicators
- Growth metrics

### Predictive Insights

**At-Risk Indicators**:
- Early warning system
- Grades dropping
- Attendance concerns
- Missing assignments
- Recommended actions

**Success Indicators**:
- Honor roll projection
- Grade improvement
- Consistent performance
- Positive trends

---

## Settings & Preferences

**View**: `/parent/settings`

### Notification Preferences

**By Child**:
- Customize notifications per child
- Email vs. SMS vs. Push
- Frequency (real-time, daily digest, weekly)
- Quiet hours (no notifications)

**Notification Types**:
- Grades posted
- Attendance alerts
- Teacher messages
- Behavioral notes
- Assignment reminders
- School announcements

### Account Settings

**Profile**:
- Update contact information
- Add/remove email addresses
- Phone numbers (work, home, mobile)
- Preferred communication method

**Language Preference**:
- English
- Spanish
- Other supported languages

**Privacy Settings**:
- Data sharing preferences
- Third-party access
- Research participation

### Multi-User Access

**Family Sharing**:
- Add co-parent access
- Grandparent access
- Legal guardian access
- Permission levels

**Student Visibility**:
- Age-appropriate data sharing
- Allow student to see parent view
- Privacy controls for older students

---

## Mobile App

**Parent Mobile App Features**:
- Dashboard with all children
- Grade notifications
- Attendance alerts
- Teacher messages
- Quick pay (lunch, fees)
- Conference scheduling
- Push notifications
- Offline access to cached data

**App Benefits**:
- Stay connected on-the-go
- Real-time updates
- Respond to teachers quickly
- Monitor multiple children easily

---

## Help & Support

**View**: `/parent/help`

### Help Resources

**FAQs**:
- How to check grades
- How to excuse an absence
- How to message teachers
- How to pay fees
- How to update contact info

**Video Tutorials**:
- Portal navigation
- Mobile app setup
- Notification settings
- Conference scheduling

**Contact Support**:
- School office phone/email
- Technical support
- District help desk
- Live chat (if available)

---

## Related Documentation

- [Gradebook System](../features/gradebook.md)
- [Attendance System](../features/attendance.md)
- [Chat System](../features/chat.md)
- [Special Programs](../features/special-programs.md)
- [Notifications](../features/notifications.md)
