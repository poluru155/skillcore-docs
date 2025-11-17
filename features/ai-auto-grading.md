# AI Auto-Grading

**Last Updated**: November 17, 2025  
**Status**: Production Feature  
**Version**: 1.0

---

## Overview

AI Auto-Grading uses advanced language models to automatically grade student work with rubric-based analysis, providing instant feedback, identifying strengths and areas for improvement, and suggesting next steps for learning.

**Key Benefits**:
- ⚡ **Time Savings**: Grade essays and responses in 3-8 seconds instead of 10-15 minutes
- 🎯 **Consistency**: Apply rubrics uniformly across all student submissions
- 💬 **Rich Feedback**: Generate detailed, constructive feedback automatically
- 📊 **Standards Alignment**: Map feedback to curriculum standards
- 🔄 **Continuous Improvement**: Learn from teacher adjustments

---

## Features

### Supported Assessment Types

- ✅ **Essays & Written Responses**: Short answer to full essays
- ✅ **Project Work**: Multi-criteria project assessment
- ✅ **Lab Reports**: Science lab report evaluation
- ✅ **Creative Writing**: Narrative, descriptive, persuasive writing
- ✅ **Problem-Solving**: Math and science problem-solving work
- ✅ **Research Papers**: Citations, structure, argumentation analysis
- ✅ **Coding Assignments**: Code quality and correctness (via DeepSeek)

### Grading Capabilities

**Rubric-Based Scoring**:
```typescript
interface GradingResult {
  // Overall scoring
  score: number
  totalPoints: number
  percentage: number
  letterGrade: string
  
  // Rubric breakdown
  rubricScores: {
    criterionName: string
    points: number
    maxPoints: number
    feedback: string
  }[]
  
  // Qualitative feedback
  feedback: string                    // Overall feedback
  strengths: string[]                 // What student did well
  improvements: string[]              // Areas to improve
  nextSteps: string[]                 // Specific actions
  
  // Standards alignment
  masteryLevel: 'emerging' | 'developing' | 'proficient' | 'advanced'
  standardsAssessed: {
    standardId: string
    masteryLevel: number              // 0-1
    evidence: string[]
  }[]
  
  // AI metadata
  confidence: number                  // 0-1
  modelUsed: string
  processingTime: number
}
```

---

## How It Works

### Step 1: Teacher Setup

```typescript
// Define assignment with rubric
const assignment = {
  title: "Persuasive Essay: Environmental Conservation",
  subject: "English Language Arts",
  gradeLevel: "10",
  
  rubric: {
    criteria: [
      {
        name: "Thesis & Argumentation",
        maxPoints: 25,
        levels: [
          { points: 25, description: "Clear, compelling thesis with strong supporting arguments" },
          { points: 20, description: "Clear thesis with adequate supporting arguments" },
          { points: 15, description: "Thesis present but arguments need development" },
          { points: 10, description: "Weak or unclear thesis" },
        ]
      },
      {
        name: "Evidence & Examples",
        maxPoints: 25,
        levels: [
          { points: 25, description: "Strong, relevant evidence from credible sources" },
          { points: 20, description: "Adequate evidence, mostly relevant" },
          { points: 15, description: "Some evidence but needs more support" },
          { points: 10, description: "Limited or weak evidence" },
        ]
      },
      {
        name: "Organization & Structure",
        maxPoints: 25,
        levels: [
          { points: 25, description: "Excellent flow and logical progression" },
          { points: 20, description: "Good organization with clear structure" },
          { points: 15, description: "Basic organization, some unclear transitions" },
          { points: 10, description: "Poor organization, difficult to follow" },
        ]
      },
      {
        name: "Language & Conventions",
        maxPoints: 25,
        levels: [
          { points: 25, description: "Sophisticated language, virtually no errors" },
          { points: 20, description: "Good language use, minor errors" },
          { points: 15, description: "Adequate language, some errors" },
          { points: 10, description: "Basic language, frequent errors" },
        ]
      },
    ]
  },
  
  // Optional: Provide exemplar for AI calibration
  exemplar: {
    text: "...",
    score: 95,
    rationale: "This essay demonstrates..."
  }
}
```

### Step 2: Student Submits Work

Student submits written response through the platform.

### Step 3: AI Grading

```typescript
const autoGradeAgent = new AutoGradingAgent({
  provider: 'anthropic',              // Claude 3.5 Sonnet for best quality
  model: 'claude-3-sonnet',
  temperature: 0.3,                   // Lower for consistency
})

const result = await autoGradeAgent.grade({
  studentWork: studentSubmission.text,
  rubric: assignment.rubric,
  assignmentType: 'essay',
  subject: 'English Language Arts',
  gradeLevel: '10',
  exemplar: assignment.exemplar?.text,
})
```

### Step 4: Teacher Review & Adjustment

```typescript
// Teacher reviews AI grading
{
  score: 87,
  letterGrade: "B+",
  
  rubricScores: [
    {
      criterionName: "Thesis & Argumentation",
      points: 23,
      maxPoints: 25,
      feedback: "Strong thesis about plastic pollution reduction. Arguments are well-developed with clear reasoning about economic and environmental impacts."
    },
    {
      criterionName: "Evidence & Examples",
      points: 22,
      maxPoints: 25,
      feedback: "Good use of statistics from EPA and recent ocean cleanup initiatives. Could strengthen with more diverse source types (interviews, case studies)."
    },
    // ... other criteria
  ],
  
  feedback: "This is a well-written persuasive essay with a clear stance on environmental conservation. Your argument about plastic pollution is particularly compelling, and you've used strong evidence to support your points. To improve further, consider addressing potential counterarguments more thoroughly and incorporating a wider variety of source types.",
  
  strengths: [
    "Clear, compelling thesis statement",
    "Strong use of statistical evidence",
    "Logical organization with smooth transitions",
    "Engaging introduction that hooks the reader",
  ],
  
  improvements: [
    "Address counterarguments more thoroughly",
    "Diversify source types beyond statistics",
    "Conclusion could be more powerful with a call to action",
  ],
  
  nextSteps: [
    "Research opposing viewpoints to strengthen your counterargument section",
    "Find 1-2 expert interviews or case studies to add depth",
    "Revise conclusion to include a specific, actionable call to readers",
  ]
}
```

Teacher can:
- ✅ Accept AI grading as-is
- ✏️ Adjust scores for any criterion
- 💬 Edit or enhance feedback
- ➕ Add additional comments
- 🎓 Override final grade if needed

### Step 5: Student Receives Feedback

Student sees:
- Overall grade and rubric breakdown
- Specific feedback for each criterion
- Strengths to celebrate
- Areas for improvement with actionable steps
- Standards mastery levels

---

## Use Cases

### Use Case 1: Daily Formative Assessment

**Scenario**: Teacher assigns daily reading comprehension questions

**Process**:
1. Students submit 2-3 paragraph responses
2. AI grades all 30 submissions in ~2 minutes
3. Teacher reviews flagged responses (low scores or low AI confidence)
4. Students receive immediate feedback
5. Teacher identifies common misconceptions for next lesson

**Time Saved**: 90 minutes → 15 minutes (83% reduction)

### Use Case 2: Major Essay Grading

**Scenario**: End-of-unit persuasive essays (120 students across 4 classes)

**Process**:
1. AI grades all essays overnight
2. Teacher reviews sample of 10-15 essays to verify AI accuracy
3. Spot-checks any grade that seems off
4. Makes final adjustments
5. Releases grades with detailed feedback

**Time Saved**: 40 hours → 8 hours (80% reduction)

### Use Case 3: Project-Based Assessment

**Scenario**: Science fair project reports (multimodal assessment)

**Process**:
1. Students submit written reports (text)
2. AI assesses: hypothesis clarity, methodology, data analysis, conclusions
3. Teacher adds scores for: presentation, visual display, innovation
4. Combined grade with comprehensive feedback

**Benefit**: Consistent written component grading + teacher expertise for creative elements

### Use Case 4: Differentiated Feedback

**Scenario**: Mixed-ability class with IEP/504 students

**Process**:
1. AI grades all submissions with same rubric
2. AI generates differentiated feedback based on student level
3. Struggling students get more scaffolded next steps
4. Advanced students get extension challenges
5. Teacher reviews and personalizes further

**Benefit**: Every student gets appropriate, personalized feedback

---

## Quality Assurance

### AI Confidence Scoring

```typescript
interface ConfidenceAnalysis {
  overallConfidence: number           // 0-1
  
  factors: {
    rubricClarity: number             // How clear the rubric is
    studentWorkClarity: number        // How clear the student work is
    alignmentToRubric: number         // How well work maps to rubric
    exemplarAvailability: boolean     // Whether exemplar was provided
  }
  
  recommendation: 'auto-accept' | 'review-suggested' | 'manual-review-required'
}
```

**Auto-Accept**: Confidence > 0.9 (90%+)
- Clear rubric
- Well-structured student work
- Strong alignment to criteria
- Teacher can review but likely accurate

**Review Suggested**: Confidence 0.7-0.9
- Minor ambiguities in work or rubric
- AI highlights specific areas of uncertainty
- Teacher should review flagged sections

**Manual Review Required**: Confidence < 0.7
- Ambiguous rubric criteria
- Unclear student work
- Poor alignment to rubric
- Teacher must review entire grading

### Teacher Calibration

System learns from teacher adjustments:

```typescript
interface CalibrationData {
  assignmentId: string
  rubricId: string
  
  adjustments: {
    criterionName: string
    aiScore: number
    teacherScore: number
    difference: number
    teacherRationale?: string
  }[]
  
  patterns: {
    criterionName: string
    avgAdjustment: number             // AI tends to score +/- this amount
    direction: 'too-lenient' | 'too-strict' | 'accurate'
  }[]
}
```

AI adjusts future grading based on teacher preferences:
- Learns grading strictness/leniency
- Adapts to specific rubric interpretation
- Improves over time with more feedback

---

## Configuration

### Grading Settings

```typescript
interface AutoGradingSettings {
  // Provider selection
  preferredProvider: 'anthropic' | 'openai' | 'google'
  fallbackProvider?: 'anthropic' | 'openai' | 'google'
  
  // Quality settings
  temperature: number                 // 0.0-1.0 (lower = more consistent)
  confidenceThreshold: number         // Minimum confidence to auto-accept
  
  // Teacher controls
  requireReviewBeforeRelease: boolean
  autoReleaseHighConfidence: boolean
  flagLowConfidence: boolean
  
  // Feedback settings
  feedbackTone: 'encouraging' | 'neutral' | 'academic'
  includeNextSteps: boolean
  includeStandardsAlignment: boolean
  
  // Cost controls
  maxCostPerGrading: number           // USD
  budgetAlert: number                 // Alert when approaching limit
}
```

### Rubric Best Practices

**✅ Good Rubrics for AI**:
- Clear, specific criteria
- Distinct performance levels
- Objective descriptors
- Aligned to standards
- 3-5 criteria (not too many)

**❌ Challenging Rubrics for AI**:
- Vague criteria ("Good effort")
- Overlapping levels
- Highly subjective (personal style)
- Too many criteria (10+)
- Culturally specific references

---

## Privacy & Security

### Student Data Protection

- ✅ Student work processed with encryption
- ✅ No student data stored by AI providers (per contract)
- ✅ Audit logs for all AI grading operations
- ✅ FERPA-compliant data handling
- ✅ Consent required for AI grading (configurable)

### Teacher Controls

- ✅ Teachers can disable AI grading for specific assignments
- ✅ Teachers can disable AI grading for specific students
- ✅ AI suggestions only - teacher has final authority
- ✅ All AI grades are provisional until teacher approval

---

## Costs

### Typical Costs per Assignment Type

| Assignment Type | Avg Tokens | Estimated Cost | Provider |
|----------------|------------|----------------|----------|
| Short Answer (2-3 paragraphs) | 500-800 | $0.01-0.02 | Gemini/DeepSeek |
| Essay (3-5 pages) | 2000-4000 | $0.05-0.15 | Claude/GPT-4 |
| Long Essay (8-10 pages) | 5000-8000 | $0.15-0.30 | Claude Sonnet |
| Project Report | 3000-6000 | $0.08-0.20 | Claude/GPT-4 |
| Lab Report | 1500-3000 | $0.04-0.10 | GPT-4/Claude |

**Monthly Teacher Estimate**: $15-25 for active use (50-100 gradings)

### Cost Optimization

- Use **Gemini Pro** or **DeepSeek** for short responses (10x cheaper)
- Use **Claude Sonnet** for essays (good quality, reasonable cost)
- Use **GPT-4** only for complex rubrics requiring deep reasoning
- Batch grading to reduce overhead

---

## Analytics

### Grading Analytics Dashboard

```typescript
interface GradingAnalytics {
  // Usage metrics
  totalGradings: number
  averageConfidence: number
  teacherAdjustmentRate: number       // % of grades adjusted
  
  // Time savings
  estimatedManualTime: number         // Hours
  actualTimeSpent: number             // Hours (with AI)
  timeSaved: number                   // Hours
  timeSavingsPercentage: number
  
  // Quality metrics
  averageScoreDifference: number      // AI vs teacher final score
  consistencyScore: number            // How consistent AI is
  
  // Cost metrics
  totalCost: number
  costPerGrading: number
  costSavingsVsManual: number         // Assuming hourly rate
  
  // Feedback metrics
  averageFeedbackLength: number       // Words
  feedbackQualityScore: number        // Teacher-rated
}
```

---

## Integration

### GraphQL API

```graphql
type Mutation {
  # Grade student submission
  autoGradeSubmission(input: AutoGradeInput!): AutoGradeResult!
  
  # Adjust AI grading
  adjustAutoGrade(
    gradingId: ID!
    adjustments: GradeAdjustments!
  ): GradingResult!
  
  # Accept AI grading
  acceptAutoGrade(gradingId: ID!): GradingResult!
}

type Query {
  # Get AI grading result
  autoGradingResult(gradingId: ID!): AutoGradeResult!
  
  # Get grading analytics
  autoGradingAnalytics(
    teacherId: ID!
    timeframe: Timeframe!
  ): GradingAnalytics!
}

input AutoGradeInput {
  submissionId: ID!
  rubricId: ID!
  exemplarId: ID
  settings: AutoGradingSettings
}

type AutoGradeResult {
  id: ID!
  score: Float!
  letterGrade: String!
  rubricScores: [RubricScore!]!
  feedback: String!
  strengths: [String!]!
  improvements: [String!]!
  nextSteps: [String!]!
  confidence: Float!
  status: GradingStatus!
  createdAt: DateTime!
}
```

---

## Best Practices

### For Teachers

1. **Start Small**: Begin with formative assessments, not high-stakes tests
2. **Provide Exemplars**: AI performs better with example work
3. **Review Initially**: Review first 5-10 gradings to verify quality
4. **Calibrate**: Make adjustments and let AI learn your preferences
5. **Use High Confidence**: Trust high-confidence gradings, review low
6. **Personalize**: Add personal touches to AI-generated feedback
7. **Monitor Costs**: Set budget alerts to avoid surprises

### For Administrators

1. **Pilot Program**: Start with volunteer teachers before rollout
2. **Set Budgets**: Configure school/district-wide budget limits
3. **Train Teachers**: Provide training on effective rubric design
4. **Monitor Quality**: Track adjustment rates and teacher satisfaction
5. **Measure Impact**: Track time savings and teacher workload reduction
6. **Review Privacy**: Ensure all consent forms and policies are in place

---

## Support & Troubleshooting

### Common Issues

**Issue**: AI scores consistently too high/low  
**Solution**: Adjust temperature setting or provide calibration feedback

**Issue**: Low confidence scores  
**Solution**: Improve rubric clarity and provide exemplar work

**Issue**: Feedback too generic  
**Solution**: Add more specific rubric descriptors and exemplars

**Issue**: High costs  
**Solution**: Switch to more cost-effective providers for simple assignments

---

## Related Documentation

- [AI Integration Architecture](../architecture/ai-integration.md)
- [AI Cost Explorer](./ai-cost-explorer.md)
- [Assessment System](./assessment.md)
- [Gradebook](./gradebook.md)

---

**Auto-Grading Status**: ✅ Production Feature  
**Supported Providers**: Claude, GPT-4, Gemini, DeepSeek  
**Average Time Savings**: 80%  
**Teacher Satisfaction**: High  
**Next Steps**: Continuous improvement through calibration
