# Student Portal

**Last Updated**: November 17, 2025  
**Status**: ✅ Production-Ready

---

## Overview

The Student Portal provides students with access to their grades, assignments, schedule, attendance, and learning resources. The interface is intuitive, mobile-friendly, and designed for student success.

---

## Dashboard

**View**: `/student/dashboard`

### Quick Overview

**Today's Snapshot**:
- Current schedule (what class is next)
- Today's assignments due
- Recent grades posted
- Attendance status (present, tardy, absent)
- Upcoming assessments
- Unread messages from teachers

**Performance Summary**:
- Current GPA
- Overall attendance rate
- Missing assignments count
- Latest grade trend (↗ improving, → stable, ↘ declining)

### Class Cards

Each class displays:
- Class name and teacher
- Current grade (percentage and letter)
- Trend indicator
- Next assignment due
- Recent activity
- Quick link to gradebook

**Color Indicators**:
- 🟢 A/B grades (90%+)
- 🟡 C/D grades (70-89%)
- 🔴 Failing (< 70%)

---

## Grades

**View**: `/student/grades`

### Current Grades

**All Classes View**:
- List of all enrolled classes
- Current grade in each class
- Teacher name
- Class period/schedule
- Credits earned
- Grade trend over time

**Detailed Class View**: `/student/grades/:classId`

**Assignment List**:
- Assignment title
- Category (Test, Quiz, Homework)
- Due date
- Points earned / Points possible
- Percentage
- Teacher feedback/comments
- Submission status

**Category Breakdown**:
- Pie chart showing category weights
- Score in each category
- Impact on overall grade
- "What if" calculator

### Grade History

**Term-by-Term**:
- View grades from previous terms
- Progress over school year
- Comparison to previous years
- GPA history

**Transcript View**:
- All courses taken
- Final grades
- Credits earned
- Cumulative GPA
- Class rank (if available)

### Grade Calculator

**"What If" Tool**:
- See impact of future assignments
- Calculate needed score for target grade
- Explore grade scenarios
- Plan improvement strategy

**Example**:
```
Current Grade: 82% (B)
Target Grade: 90% (A)
Remaining Assignments: 3 tests @ 100 points each

To achieve 90%:
Test 1: Need 95%
Test 2: Need 92%
Test 3: Need 88%
```

---

## Assignments

**View**: `/student/assignments`

### Assignment List

**Upcoming Assignments**:
- Due this week
- Due next week
- Future assignments

**Overdue/Missing**:
- Highlighted in red
- Late penalty information
- Last date to submit

**Completed**:
- Submission date
- Grade received
- Teacher feedback

### Assignment Details

**View Assignment**: `/student/assignments/:id`

**Assignment Info**:
- Title and description
- Class and teacher
- Category and points possible
- Due date and time
- Standards/objectives covered
- Resources/materials needed
- Submission instructions

**Submission Methods**:
- File upload (PDF, Word, images)
- Text entry
- Link submission
- In-person submission (marked as turned in)

### Assignment Submission

**Upload Work**:
1. Select file or paste text
2. Preview submission
3. Add student comments
4. Submit
5. Confirmation with timestamp

**Late Submissions**:
- Warning if past due date
- Late penalty display
- Teacher approval required
- Reason for late submission

**Resubmission**:
- If teacher allows revisions
- View previous attempts
- Compare grades
- Submit improved version

---

## Assessments

**View**: `/student/assessments`

### Upcoming Assessments

**Assessment Calendar**:
- Tests, quizzes, exams
- Due dates
- Class/subject
- Study resources
- Standards covered

**Preparation Status**:
- Days until assessment
- Suggested study time
- Practice resources
- Review materials

### Take Assessment

**Online Assessment**: `/student/assessments/:id/take`

**Assessment Interface**:
- Question navigation
- Time remaining (if timed)
- Progress indicator
- Flag questions for review
- Auto-save responses
- Submit when complete

**Question Types**:
- Multiple choice
- True/False
- Short answer
- Essay
- Matching
- File upload

### Assessment Results

**View Results**: `/student/assessments/:id/results`

**Results Display**:
- Score earned
- Percentage/grade
- Class average (optional)
- Question-by-question review
- Correct answers (if shown)
- Teacher feedback
- Standards mastery

**Retake Options**:
- If teacher allows multiple attempts
- View previous attempts
- Compare scores
- Highest score or most recent

---

## Schedule

**View**: `/student/schedule`

### Daily Schedule

**Class Schedule**:
- Period number and time
- Class name
- Teacher name
- Room number
- Type (Regular, Lab, Study Hall)

**Today's Focus**:
- Current period highlighted
- Next class countdown
- Upcoming assignments today
- Assessments today

### Week View

**Weekly Calendar**:
- All classes by day and period
- Special schedules (early dismissal, late start)
- Assembly/activity periods
- Lunch period
- Study hall/free periods

### Semester/Year View

**Full Schedule**:
- All enrolled classes
- Credits per course
- Prerequisites met
- Graduation progress

---

## Attendance

**View**: `/student/attendance`

### Attendance Record

**Calendar View**:
- Color-coded days:
  - 🟢 Present
  - 🔴 Absent
  - 🟡 Tardy
  - ⚪ Weekend/No school

**Summary Stats**:
- Total days attended
- Total absences
- Tardy count
- Attendance percentage
- Days missed by month

**Detailed Records**:
- Date
- Status (Present, Absent, Tardy, Excused)
- Class/period
- Notes/reason

### Attendance Alerts

**Warning Indicators**:
- 🟡 3+ absences in 7 days
- 🔴 Chronic absenteeism (10%+ missed)
- 🔴 Truancy risk

**Improvement Plan**:
- If intervention active
- Goals and strategies
- Progress tracking
- Weekly check-ins

---

## Messages

**View**: `/student/messages`

### Teacher Messages

**Inbox**:
- Messages from teachers
- Read/unread status
- Date received
- Subject line
- Quick reply

**Compose Message**:
- Select teacher
- Subject and message body
- Attach files
- Send

**Message Guidelines**:
- Professional communication
- Content filtering (profanity blocked)
- Parent notification (for younger students)

### Announcements

**Class Announcements**:
- Posted by teacher
- Class-specific updates
- Homework reminders
- Test notifications
- General updates

**School Announcements**:
- School-wide updates
- Event notifications
- Schedule changes
- Important dates

---

## Resources

**View**: `/student/resources`

### Learning Resources

**By Class**:
- Study guides
- Video tutorials
- Practice problems
- Reference materials
- Helpful links

**Resource Library**:
- Search by subject
- Filter by grade level
- Recently viewed
- Favorites/bookmarks

**Resource Types**:
- Videos
- Documents (PDF, Word)
- Interactive activities
- External links
- eBooks

### Academic Support

**Tutoring**:
- Schedule tutoring session
- Available tutors by subject
- Peer tutoring options
- After-school programs

**Academic Help**:
- Study skills resources
- Time management tips
- Test-taking strategies
- Organization tools

---

## Progress & Analytics

**View**: `/student/analytics`

### Performance Trends

**Grade Trends**:
- Line graph showing grades over time
- By class or overall
- Identify patterns
- Celebrate improvements

**Standards Mastery**:
- Radar chart by subject
- Proficiency levels
- Areas of strength
- Areas needing improvement

### Goal Setting

**Academic Goals**:
- Set target GPA
- Track progress toward goal
- Milestones and checkpoints
- Celebrate achievements

**Goal Types**:
- Grade improvement
- Attendance improvement
- Assignment completion
- Test score targets

### Achievements

**Badges & Awards**:
- Honor roll
- Perfect attendance
- Most improved
- Academic excellence
- Citizenship awards

---

## College & Career

**View**: `/student/college-career` (High School)

### College Planning

**College Research**:
- Saved colleges
- Application deadlines
- Required tests (SAT, ACT)
- GPA requirements
- Scholarship opportunities

**Application Tracker**:
- Applications in progress
- Submitted applications
- Acceptance/rejection status
- Financial aid awarded

### Career Exploration

**Career Assessments**:
- Interest inventories
- Skills assessments
- Personality tests
- Career matches

**Career Resources**:
- Career profiles
- Salary information
- Education requirements
- Job outlook

### Course Planning

**4-Year Plan**:
- Course requirements for graduation
- College prep track
- CTE pathways
- AP/Honors courses
- Elective choices

---

## Special Programs

### IEP/504 (If Applicable)

**View Accommodations**: `/student/special-programs/accommodations`

**My Accommodations**:
- Extended time on tests
- Preferential seating
- Read-aloud support
- Calculator use
- Other supports

**Self-Advocacy**:
- Request accommodations
- Communicate needs to teachers
- Progress on IEP goals

### ESL Support

**English Proficiency**:
- Current level (1-6)
- Progress over time
- Language support services
- Translation tools

### Gifted & Talented

**Enrichment Opportunities**:
- Advanced coursework
- Independent projects
- Competitions
- Mentorship programs

---

## Wellness & Support

**View**: `/student/wellness`

### Counseling Services

**Schedule Appointment**:
- School counselor
- Social worker
- Psychologist
- Crisis support

**Topics**:
- Academic concerns
- Social-emotional support
- College/career planning
- Personal issues

### Interventions

**My Support Plan**:
- If receiving academic/behavioral support
- Goals and strategies
- Progress tracking
- Team members

**Check-Ins**:
- Daily/weekly check-in schedule
- Reflection prompts
- Self-reporting tools

---

## Calendar

**View**: `/student/calendar`

### Personal Calendar

**Events**:
- Assignment due dates
- Assessment dates
- School events
- Extracurricular activities
- Personal appointments

**Sync Options**:
- Export to Google Calendar
- iCal integration
- Mobile calendar sync

---

## Settings

**View**: `/student/settings`

### Notification Preferences

**Email Notifications**:
- New grades posted
- Assignment reminders
- Teacher messages
- Schedule changes

**Push/SMS Notifications**:
- Due date reminders (1 day before)
- Grade posted alerts
- Urgent messages

### Display Preferences

**Theme**:
- Light mode
- Dark mode
- High contrast

**Language**:
- English
- Spanish
- Other supported languages

### Privacy

**Profile Visibility**:
- Photo display
- Contact information
- Activity status

**Data Sharing**:
- Share progress with parents
- Participation in research studies

---

## Mobile App

**Student Mobile App Features**:
- Quick grade check
- Assignment due dates
- Daily schedule
- Attendance record
- Push notifications
- Message teachers
- Submit assignments
- Take assessments

**Offline Access**:
- View cached grades
- Read downloaded resources
- Work on offline assignments
- Sync when online

---

## Parent Access

**Parent Portal Link**: `/parent/dashboard`

Students can:
- See what parents see
- Understand shared information
- Request privacy for certain data (age-appropriate)

---

## Gamification (Optional)

### Points & Rewards

**Earn Points**:
- Submit assignments on time
- Improve grades
- Perfect attendance
- Participate in class
- Help peers

**Redeem Rewards**:
- School store items
- Privilege passes
- Recognition certificates
- Special events

### Leaderboards

**Class Rankings** (Optional):
- Top performers
- Most improved
- Best attendance
- Citizenship leaders

---

## Related Documentation

- [Gradebook System](../features/gradebook.md)
- [Assessment System](../features/assessment.md)
- [Attendance System](../features/attendance.md)
- [Chat System](../features/chat.md)
