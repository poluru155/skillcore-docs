# Teacher Portal

**Last Updated**: November 17, 2025  
**Status**: ✅ Production-Ready

---

## Overview

The Teacher Portal provides a comprehensive interface for educators to manage classes, grades, attendance, assessments, and student communication. All features are designed with educational workflows in mind and comply with FERPA requirements.

---

## Dashboard

### Quick Stats Overview

**Daily Snapshot**:
- Today's schedule with period times
- Classes with unmarked attendance (highlighted)
- Pending grade entries
- Unread parent messages
- Upcoming assignments/assessments
- Student alerts (absences, failing grades, interventions)

**Week-at-a-Glance**:
- Upcoming deadlines
- Scheduled assessments
- Parent-teacher conferences
- Professional development sessions

### Class Performance Cards

Each class shows:
- Class name and period
- Current average grade
- Attendance rate (last 7 days)
- Number of missing assignments
- Students needing attention (color-coded)
- Quick action buttons (Take Attendance, Enter Grades, Post Announcement)

**Visual Indicators**:
- 🟢 Green: Class average > 80%, good attendance
- 🟡 Yellow: Class average 70-80% or attendance concerns
- 🔴 Red: Class average < 70%, intervention needed

---

## Classes & Roster Management

### Class Overview

**Available via**: `/teacher/classes`

Each class shows:
- Class title (e.g., "Algebra I - Period 3")
- Academic session
- Meeting schedule (days and times)
- Room number
- Student count
- Current term/grading period

### Student Roster

**View**: `/teacher/classes/:classId/roster`

**Student List Features**:
- Student name, photo, student ID
- Current grade in class
- Attendance rate
- Active alerts (IEP, 504, intervention)
- Quick actions (Message parent, View profile, Grade history)

**Bulk Actions**:
- Send group message
- Export roster to CSV
- Print class list
- Copy emails to clipboard

### Individual Student Profile

**View**: `/teacher/classes/:classId/students/:studentId`

**Comprehensive Student View**:
- Demographics and contact info
- Current grades across all classes
- Attendance history
- Special programs (IEP, 504, ESL, Gifted)
- Active interventions
- Parent/guardian contact info
- Assignment history
- Behavioral notes
- Accommodation needs

---

## Attendance Management

**View**: `/teacher/attendance`

### Quick Entry Mode

**Homeroom Attendance** (Daily):
- Student roster with checkboxes
- "Mark All Present" button
- Quick status buttons (P, A, T, EA, ET)
- Keyboard shortcuts (P key = Present, A key = Absent, T key = Tardy)
- Real-time save (no submit button needed)
- Auto-notifications to parents on absence

**Period-by-Period**:
- Tab interface for each class period
- Copy attendance from previous period
- Bulk actions menu
- Visual indicators for students with attendance concerns

### Attendance Codes

| Code | Meaning | Notifies Parent |
|------|---------|-----------------|
| P | Present | No |
| A | Absent | Yes (immediate) |
| T | Tardy | Yes (daily digest) |
| EA | Excused Absent | Yes |
| ET | Excused Tardy | No |
| HO | Health Office | No |
| FT | Field Trip | No |
| S | Suspension | Yes |

### Attendance Alerts

**Automatic Notifications**:
- 🔴 3+ absences in 7 days → Counselor notified
- 🔴 Chronic absenteeism (10%+) → Admin + counselor
- 🟡 3 consecutive absences → Tier 1 intervention created

**Historical View**:
- Calendar view of student attendance
- Color-coded days
- Filter by status
- Export attendance reports

---

## Gradebook

**View**: `/teacher/gradebook`

### Spreadsheet Interface

**Layout**:
- Students in rows (frozen left column)
- Assignments in columns
- Inline editing with keyboard navigation
- Auto-save on cell blur
- Color-coded cells:
  - Green: 90-100%
  - Yellow: 70-89%
  - Red: < 70%
  - Gray: Not graded yet

**Keyboard Navigation**:
- Tab: Next cell
- Shift+Tab: Previous cell
- Enter: Edit cell
- Arrow keys: Navigate
- Esc: Cancel edit

### Grade Categories

**Setup**: `/teacher/classes/:classId/settings/categories`

**Common Configurations**:
```
Tests: 40%
Quizzes: 25%
Homework: 20%
Participation: 15%
```

**Advanced Options**:
- Drop lowest N scores
- Extra credit weighting
- Standards-based grading
- Custom calculation formulas

### Grade Entry Workflows

**Quick Entry**:
1. Select assignment column
2. Enter scores sequentially
3. Auto-advance to next student
4. Real-time calculation of averages

**Rubric-Based Entry**:
1. Create rubric with criteria
2. Grade each criterion
3. Auto-calculate total score
4. Save rubric for future use

**Import from CSV**:
1. Download template
2. Fill in scores
3. Upload CSV file
4. Review and confirm

### Grade Posting

**Draft Mode** (Default):
- Grades entered but not visible to students/parents
- Make corrections as needed
- Review before posting

**Post Grades**:
- Confirmation dialog
- Automatic parent notifications
- Email with grade summary
- SMS alert (if enabled)
- Push notification

**Lock Gradebook**:
- Prevent further edits
- End-of-term finalization
- Generate report cards

### Grade Analytics

**Class-Level**:
- Grade distribution chart (histogram)
- Average, median, mode
- Standard deviation
- Comparison to previous terms

**Student-Level**:
- Trend analysis (improving/declining)
- Missing assignments list
- Category breakdown
- Standards mastery heatmap

---

## Assignments & Assessments

**View**: `/teacher/assignments`

### Assignment Creation

**Create Assignment**: `/teacher/assignments/new`

**Required Fields**:
- Title
- Description
- Class/section
- Category (Test, Quiz, Homework, etc.)
- Total points
- Due date
- Assigned date

**Optional Fields**:
- Allow late submissions (Yes/No)
- Late penalty (% per day)
- Rubric attachment
- Standards alignment
- Resource links
- File attachments

### Assignment Types

**1. Traditional Assignment**:
- Point-based grading
- Submit via platform or paper
- Due date with late policy

**2. Standards-Based Assessment**:
- Proficiency levels (1-4)
- Multiple attempts allowed
- Mastery tracking

**3. Project/Performance Task**:
- Rubric-based grading
- Multiple criteria
- Weight by importance

**4. Participation/Effort**:
- Pass/Fail or point-based
- Daily/weekly tracking

### Assignment Library

**Reusable Templates**:
- Save assignments for future use
- Share with department
- Import from colleagues
- Curriculum-aligned resources

### Student Submissions

**View Submissions**: `/teacher/assignments/:id/submissions`

**Features**:
- List view: All students, submission status
- Filter: Submitted, Not submitted, Late, Graded
- Bulk download all submissions
- Side-by-side grading interface
- Add comments and feedback
- Return for revision

---

## Assessments (Formative & Summative)

**View**: `/teacher/assessments`

### Assessment Builder

**Create Assessment**: `/teacher/assessments/new`

**Assessment Types**:
- Quiz (5-10 questions)
- Test (10+ questions)
- Exit Ticket (1-3 questions)
- Observation Checklist
- Performance Task

**Question Types**:
- Multiple Choice
- True/False
- Short Answer
- Essay
- Matching
- Fill in the Blank
- File Upload

**Advanced Options**:
- Question bank integration
- Randomize questions
- Time limit
- Auto-grade (MC, T/F)
- Standards alignment
- Difficulty tagging

### Assessment Administration

**In-Class Administration**:
- Start session
- Student roster check-in
- Live progress monitoring
- Time remaining countdown
- Submit all at end

**Take-Home/Online**:
- Assign to class
- Set availability window
- Track completion
- Prevent multiple submissions

### Results & Analysis

**Individual Results**:
- Student scores
- Question-by-question breakdown
- Time spent
- Attempt history (if multiple attempts allowed)

**Class Analytics**:
- Score distribution
- Question difficulty analysis
- Most missed questions
- Standards mastery by student
- Item analysis (discrimination index)

**Data-Driven Instruction**:
- Identify learning gaps
- Reteach recommendations
- Small group formation
- Intervention triggers

---

## Communication

### Parent Messaging

**View**: `/teacher/messages`

**Features**:
- 1-on-1 conversations with parents
- Group messages (all parents in class)
- Message templates
- Attachment support
- Read receipts
- Translation (for multilingual families)

**Common Messages**:
- Grade concerns
- Behavioral updates
- Positive reinforcement
- Conference scheduling
- Missing work reminders

### Class Announcements

**Post Announcement**: `/teacher/classes/:classId/announcements`

**Announcement Types**:
- General updates
- Homework reminders
- Test/quiz notifications
- Schedule changes
- Resource sharing

**Delivery Options**:
- Post to class feed
- Email to all parents
- Push notification
- SMS alert

### Parent-Teacher Conferences

**Schedule Conferences**: `/teacher/conferences`

**Features**:
- Calendar integration
- Time slot availability
- Parent self-scheduling
- Automatic reminders
- Virtual meeting links (Zoom/Teams)
- Agenda templates
- Note-taking interface
- Action items tracking

---

## Student Interventions

**View**: `/teacher/interventions`

### Intervention Types

**Academic Interventions**:
- Below grade level in subject
- Failing grades (< 70%)
- Missing assignments
- Test score decline

**Behavioral Interventions**:
- Attendance concerns
- Classroom disruptions
- Homework completion
- Participation issues

**Social-Emotional**:
- Peer relationship issues
- Motivation concerns
- Anxiety/stress indicators

### Intervention Workflow

**1. Identify Student**:
- Automatic triggers (grades, attendance)
- Teacher referral
- Counselor referral

**2. Create Intervention Plan**:
- Select tier (1, 2, or 3)
- Define goal (SMART format)
- Choose strategies
- Set review date
- Assign team members

**3. Monitor Progress**:
- Weekly check-ins
- Progress notes
- Data collection
- Graph progress toward goal

**4. Review & Adjust**:
- Team meeting
- Data analysis
- Continue, modify, or exit intervention

### Teacher Responsibilities

**Tier 1 (Universal)**:
- Implement classroom strategies
- Collect data weekly
- Communicate with parents
- Request support as needed

**Tier 2 (Targeted)**:
- Collaborate with intervention team
- Provide progress monitoring data
- Attend team meetings
- Adjust instruction

**Tier 3 (Intensive)**:
- Participate in comprehensive evaluation
- Implement individualized strategies
- Frequent data collection
- Coordinate with specialists

---

## Special Programs

### IEP Students

**View IEP**: `/teacher/special-programs/iep/:studentId`

**Teacher Access**:
- Goals and objectives
- Accommodations required
- Modifications needed
- Service minutes
- Progress monitoring schedule

**Accommodations**:
- Extended time on tests
- Preferential seating
- Read-aloud support
- Reduced assignments
- Calculator use
- Separate testing location

**Progress Reporting**:
- Quarterly progress on IEP goals
- Narrative or data-based
- Submitted to case manager
- Shared with parents

### Section 504 Plans

**View 504 Plan**: `/teacher/special-programs/504/:studentId`

**Accommodations Catalog**:
- Physical/environmental
- Testing/assignments
- Behavioral supports
- Medical needs

**Implementation Tracking**:
- Daily/weekly checklist
- Effectiveness notes
- Suggested adjustments

### ESL Students

**View ESL Profile**: `/teacher/special-programs/esl/:studentId`

**Student Info**:
- English proficiency level (1-6)
- Native language
- Service model (push-in, pull-out)
- WIDA scores

**Instructional Strategies**:
- Scaffolding recommendations
- Visual supports
- Vocabulary pre-teaching
- Simplified language

### Gifted & Talented

**View Gifted Profile**: `/teacher/special-programs/gifted/:studentId`

**Differentiation Plan**:
- Areas of giftedness
- Acceleration options
- Enrichment activities
- Independent projects

---

## Analytics & Reports

**View**: `/teacher/analytics`

### Class Performance

**Overview Metrics**:
- Class average (trend over time)
- Attendance rate
- Assignment completion rate
- Assessment averages

**Comparison Views**:
- This class vs. other sections
- This year vs. last year
- By grading period

### Student Groups

**At-Risk Students**:
- Failing (< 60%)
- Borderline (60-69%)
- Chronic absenteeism
- Multiple missing assignments

**High Performers**:
- Honor roll candidates
- Improvement leaders
- Perfect attendance

### Standards Mastery

**Standards Tracking**:
- Heatmap by student and standard
- Class-wide mastery percentage
- Priority standards needing reteach
- Students needing remediation

### Export Options

**Report Formats**:
- PDF: Printable reports
- Excel: Data analysis
- CSV: Import to other systems
- Google Sheets: Collaboration

---

## Calendar & Schedule

**View**: `/teacher/calendar`

### Schedule View

**Daily Schedule**:
- Period times
- Class names
- Room numbers
- Duty assignments

**Week View**:
- All classes at a glance
- Planning periods highlighted
- Meetings/PD sessions
- After-school commitments

**Month View**:
- Assessment dates
- Field trips
- Conferences
- Professional development

### Lesson Planning

**Lesson Plan Builder**:
- Template library
- Standards alignment
- Learning objectives
- Materials needed
- Assessment methods
- Differentiation strategies

**Unit Planning**:
- Multi-week units
- Pacing guides
- Resource collections
- Collaborative planning

---

## Professional Development

**View**: `/teacher/professional-development`

### PD Tracking

**Completed Training**:
- Course title
- Date completed
- Hours earned
- Certificate download

**Required Training**:
- District mandates
- State certification
- Compliance training
- Due dates

**Recommended Training**:
- Based on subject area
- Based on student data
- Peer recommendations

### Learning Communities

**PLCs (Professional Learning Communities)**:
- Department/grade-level groups
- Discussion forums
- Resource sharing
- Lesson study

---

## Settings & Preferences

**View**: `/teacher/settings`

### Notification Preferences

**Email Notifications**:
- New parent messages (immediate/daily digest)
- Grade posting reminders
- Attendance alerts
- Intervention updates

**SMS/Push Notifications**:
- Urgent messages only
- Daily summary
- Custom schedule

### Gradebook Preferences

**Display Options**:
- Show percentages or letter grades
- Color-coded cells
- Sort order (name, grade)

**Calculation Options**:
- Rounding rules
- Standards scoring method (most recent vs. highest)
- Extra credit handling

### Privacy & Security

**Account Security**:
- Two-factor authentication
- Password change
- Active sessions
- Login history

**Data Privacy**:
- FERPA training completion
- Data sharing preferences
- Student photo permissions

---

## Mobile App Features

**Teacher Mobile App**:
- Quick attendance entry
- Grade entry on-the-go
- Parent message responses
- View schedule
- Student profile lookup
- Intervention notes
- Push notifications

**Offline Mode**:
- Cache roster data
- Enter attendance offline
- Sync when online

---

## Related Documentation

- [Gradebook System](../features/gradebook.md)
- [Attendance System](../features/attendance.md)
- [Assessment System](../features/assessment.md)
- [Special Programs](../features/special-programs.md)
- [Interventions](../features/interventions.md)
- [Chat System](../features/chat.md)
