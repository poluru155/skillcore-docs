# AI Lesson Planner

**Last Updated**: November 17, 2025  
**Status**: Production Feature  
**Version**: 1.0

---

## Overview

AI Lesson Planner generates comprehensive, standards-aligned lesson plans from simple teacher prompts, supporting multiple pedagogical frameworks and automatically aligning to curriculum standards.

**Key Benefits**:
- ⏱️ **Time Savings**: Create lesson plans in 10-20 seconds instead of 1-2 hours
- 🎯 **Standards Alignment**: Automatic alignment to curriculum standards
- 📚 **Multiple Templates**: Support for 7+ pedagogical frameworks
- 🔄 **Differentiation**: Built-in differentiation strategies
- 📊 **Assessment Integration**: Formative and summative assessments included

---

## Features

### Supported Lesson Templates

1. **Madeline Hunter (Direct Instruction)**
   - Anticipatory Set
   - Input & Modeling
   - Guided Practice
   - Independent Practice
   - Closure

2. **5E Model (Science)**
   - Engage
   - Explore
   - Explain
   - Elaborate
   - Evaluate

3. **Understanding by Design (UbD)**
   - Desired Results
   - Assessment Evidence
   - Learning Plan

4. **Project-Based Learning**
   - Driving Question
   - Project Overview
   - Milestones
   - Final Product

5. **Inquiry-Based Learning**
   - Question Generation
   - Investigation
   - Analysis
   - Conclusion

6. **Gradual Release of Responsibility**
   - Focus Lesson
   - Guided Instruction
   - Collaborative Learning
   - Independent Learning

7. **Workshop Model**
   - Mini-Lesson
   - Independent Work Time
   - Conferencing
   - Share/Reflection

---

## How It Works

### Step 1: Teacher Prompt

Simple prompt describing the lesson:

```typescript
const prompt = `
Create a 60-minute Grade 5 Math lesson on comparing fractions with unlike denominators.
Students struggle with finding common denominators.
I want hands-on activities and visual models.
`
```

### Step 2: AI Generation

```typescript
const lessonPlanAgent = new LessonPlanGeneratorAgent({
  provider: 'openai',
  model: 'gpt-4-turbo-preview',
  vectorStore: curriculumStandardsDB,
})

const lessonPlan = await lessonPlanAgent.generateFromPrompt({
  prompt,
  template: 'madeline-hunter',
  duration: 60,
  gradeLevel: '5',
  subject: 'Mathematics',
})
```

### Step 3: Generated Lesson Plan

```typescript
{
  title: "Comparing Fractions with Unlike Denominators Using Visual Models",
  duration: 60,
  gradeLevel: "5",
  subject: "Mathematics",
  template: "madeline-hunter",
  
  standards: [
    {
      identifier: "CCSS.MATH.CONTENT.5.NF.A.1",
      title: "Add and subtract fractions with unlike denominators",
      alignment: "major",
    },
    {
      identifier: "CCSS.MATH.CONTENT.5.NF.A.2",
      title: "Solve word problems involving addition and subtraction of fractions",
      alignment: "supporting",
    }
  ],
  
  objectives: [
    "Students will compare fractions with unlike denominators using visual models with 80% accuracy",
    "Students will find common denominators using least common multiples",
    "Students will explain their reasoning when comparing fractions",
  ],
  
  materials: [
    "Fraction strips or circles (manipulatives)",
    "Fraction comparison worksheet",
    "Whiteboard and markers",
    "Student notebooks",
    "Visual fraction chart (poster)",
  ],
  
  // Template-specific structure
  sections: [
    {
      name: "Anticipatory Set",
      duration: 5,
      description: "Hook students and activate prior knowledge",
      activities: [
        "Display two pizzas - one cut into 4 slices, one into 8 slices",
        "Ask: 'Which is more: 3/4 or 5/8 of a pizza?'",
        "Take predictions and reasoning from students",
        "Connect to today's learning: 'We'll learn to compare any fractions, even with different denominators'"
      ]
    },
    {
      name: "Input & Modeling",
      duration: 15,
      description: "Teacher demonstrates the concept",
      activities: [
        "Introduce visual fraction strips",
        "Model comparing 1/2 and 3/4 using fraction strips",
        "Think aloud: 'I need to line them up to compare. I can see 3/4 is longer than 1/2'",
        "Introduce finding common denominators: 'Sometimes we need the same-sized pieces to compare'",
        "Demonstrate converting 1/2 to 2/4 using visual model",
        "Show equivalent fractions on number line",
      ],
      teacherNotes: [
        "Emphasize visual understanding before algorithms",
        "Use student-friendly language ('same-sized pieces' vs 'common denominators')",
        "Check for understanding with thumbs up/down",
      ]
    },
    {
      name: "Guided Practice",
      duration: 15,
      description: "Students practice with teacher support",
      activities: [
        "Work through examples together: 2/3 vs 3/4",
        "Students use fraction manipulatives at desks",
        "Call on students to explain their reasoning",
        "Practice finding common denominators for 1/3 and 1/6",
        "Use number line to verify comparisons",
      ],
      differentiation: [
        "Struggling: Provide pre-marked fraction strips",
        "On-level: Use fraction circles and rectangles",
        "Advanced: Challenge with three-fraction comparisons (1/2, 2/3, 3/4)",
      ],
      checkForUnderstanding: [
        "Whiteboard check: Compare 3/8 and 1/4",
        "Partner talk: Explain how you found common denominator",
        "Exit ticket preview: Quick comparison problem",
      ]
    },
    {
      name: "Independent Practice",
      duration: 20,
      description: "Students work independently",
      activities: [
        "Distribute comparison worksheet (12 problems, varied difficulty)",
        "Students work independently using fraction manipulatives as needed",
        "Teacher circulates, provides support, checks work",
        "Early finishers: Challenge problems with mixed numbers",
      ],
      assessment: {
        formative: "Teacher observation during independent work",
        criteria: "Correct comparison in at least 10/12 problems",
        intervention: "Pull small group if multiple students struggling",
      }
    },
    {
      name: "Closure",
      duration: 5,
      description: "Summarize learning and check understanding",
      activities: [
        "Review: 'What did we learn about comparing fractions today?'",
        "Students share: One thing they learned, one question they still have",
        "Preview tomorrow: 'We'll use this skill to add fractions with unlike denominators'",
        "Exit ticket: Compare 2/5 and 3/10, explain your answer",
      ]
    }
  ],
  
  assessment: {
    formative: [
      "Thumbs up/down during modeling",
      "Whiteboard checks during guided practice",
      "Teacher observation during independent work",
      "Partner explanations",
    ],
    summative: [
      "Exit ticket (2/5 vs 3/10)",
      "Worksheet completion (10/12 accuracy target)",
    ]
  },
  
  differentiation: {
    struggling: [
      "Pre-marked fraction strips with labels",
      "Work with partner or teacher",
      "Limit to simpler fractions (halves, fourths, eighths)",
      "Provide fraction comparison chart",
    ],
    onLevel: [
      "Variety of manipulatives (strips, circles, rectangles)",
      "Mix of visual and numerical problems",
      "Partner work option",
    ],
    advanced: [
      "Three-fraction comparisons",
      "Mixed number comparisons",
      "Create own comparison problems for classmates",
      "Extension: Ordering fractions from least to greatest",
    ]
  },
  
  homeworkOptional: [
    "Practice worksheet: 10 comparison problems",
    "Real-world connection: Find fractions at home (recipe, sports stats)",
    "Parent communication: Show family how you compare fractions",
  ],
  
  accommodations: {
    IEP: [
      "Extended time on independent practice",
      "Reduced number of problems (6 instead of 12)",
      "Preferential seating near teacher",
      "Access to calculator for finding LCM",
    ],
    ELL: [
      "Visual vocabulary chart (numerator, denominator, common)",
      "Sentence frames for explanations",
      "Partner with strong English speaker",
      "Use manipulatives heavily",
    ],
    "504": [
      "Breaks as needed",
      "Minimize visual distractions on worksheet",
      "Oral instructions in addition to written",
    ]
  },
  
  aiGenerated: true,
  generatedBy: "LessonPlanGeneratorAgent",
  aiPrompt: prompt,
  confidence: 0.92,
}
```

---

## Use Cases

### Use Case 1: Weekly Lesson Planning

**Scenario**: Teacher needs 5 lessons for upcoming week

**Process**:
1. Enter 5 prompts (one per lesson)
2. AI generates all 5 in ~2 minutes
3. Teacher reviews and customizes each
4. Save to lesson plan bank
5. Share with co-teachers

**Time Saved**: 8 hours → 1 hour

### Use Case 2: Emergency Substitute Plans

**Scenario**: Teacher out sick, needs immediate sub plans

**Process**:
1. Generate 1-day substitute lesson plan
2. Include detailed instructions for non-specialist
3. Add classroom management tips
4. Include answer keys
5. Send to administrator

**Time Saved**: 45 minutes → 5 minutes (while sick!)

### Use Case 3: Unit Planning

**Scenario**: Plan complete 3-week unit on American Revolution

**Process**:
1. Generate 15 lesson outlines
2. Ensure progression and scaffolding
3. Include variety of activities
4. Align all to standards
5. Build in assessments

**Time Saved**: 20 hours → 4 hours

### Use Case 4: Differentiated Lessons

**Scenario**: Mixed-ability class with wide range

**Process**:
1. Generate base lesson
2. Request differentiation strategies
3. Get tiered activities
4. Include scaffolds and extensions
5. Customize for specific students

**Time Saved**: 3 hours → 30 minutes

---

## Advanced Features

### Standards-Aligned Generation

AI automatically finds relevant standards using vector search:

```typescript
// AI searches curriculum standards database
const relevantStandards = await vectorStore.similaritySearch(
  "comparing fractions with unlike denominators grade 5",
  5,
  { subject: 'Mathematics', gradeLevel: '5' }
)

// Includes in lesson plan with alignment strength
standards: [
  { id: "5.NF.A.1", alignment: "major" },
  { id: "5.NF.A.2", alignment: "supporting" },
  { id: "MP.3", alignment: "minor" },  // Math Practice standard
]
```

### Vocabulary Integration

AI identifies key vocabulary and suggests teaching strategies:

```typescript
vocabulary: [
  {
    term: "numerator",
    definition: "The top number in a fraction, showing how many parts you have",
    studentFriendly: "The counting number on top",
    teachingStrategy: "Point to numerator repeatedly, use hand gestures (top)",
    usage: "In the fraction 3/4, 3 is the numerator",
  },
  {
    term: "common denominator",
    definition: "Denominators that are the same in two or more fractions",
    studentFriendly: "Making the bottom numbers match",
    teachingStrategy: "Use visual models showing same-sized pieces",
    usage: "To compare 1/2 and 1/4, we can use 4 as the common denominator",
  }
]
```

### Real-World Connections

AI suggests real-world applications:

```typescript
realWorldConnections: [
  "Pizza slices and sharing with friends",
  "Cooking recipes (1/2 cup vs 3/4 cup)",
  "Sports statistics (batting averages, completion percentages)",
  "Money (quarters, dimes, nickels as fractions of dollar)",
  "Time (1/4 hour = 15 minutes, 1/2 hour = 30 minutes)",
]
```

### Question Bank Generation

AI creates discussion questions and checks for understanding:

```typescript
questions: {
  openingQuestions: [
    "Have you ever had to share something equally? How did you decide who got what?",
    "Why do you think we need to compare fractions?",
  ],
  
  guidingQuestions: [
    "How can we tell which fraction is larger if the denominators are different?",
    "What does it mean to have a 'common' denominator?",
    "Can you explain why 2/4 is the same as 1/2?",
  ],
  
  checkForUnderstanding: [
    "Which is larger: 3/8 or 1/4? How do you know?",
    "Explain to your partner how you would compare 2/3 and 3/4",
    "Create a fraction that is larger than 1/2 but smaller than 3/4",
  ],
  
  higherOrderThinking: [
    "Why is finding a common denominator helpful for comparing fractions?",
    "Can you create a rule for comparing fractions with the same numerator?",
    "How would you explain comparing fractions to a younger student?",
  ],
}
```

---

## Configuration

### Lesson Plan Settings

```typescript
interface LessonPlanSettings {
  // Template preferences
  defaultTemplate: LessonTemplate
  allowTemplateModification: boolean
  
  // Generation settings
  includeStandards: boolean
  includeVocabulary: boolean
  includeRealWorldConnections: boolean
  includeHomework: boolean
  includeDifferentiation: boolean
  includeAccommodations: boolean
  
  // Style preferences
  detailLevel: 'brief' | 'moderate' | 'detailed'
  languageStyle: 'formal' | 'conversational'
  includeTeacherNotes: boolean
  
  // Standards settings
  standardsFramework: 'Common Core' | 'Ontario' | 'IB' | 'state-specific'
  autoAlignStandards: boolean
  
  // AI settings
  provider: 'openai' | 'anthropic' | 'google'
  model: string
  temperature: number
  
  // Cost controls
  maxCostPerPlan: number
}
```

---

## Integration with Curriculum

### Unit Planning

Generate complete unit with multiple lessons:

```typescript
const unitPlan = await lessonPlanAgent.generateUnit({
  topic: "American Revolution",
  duration: "3 weeks (15 lessons)",
  gradeLevel: "8",
  subject: "Social Studies",
  
  learningGoals: [
    "Understand causes of American Revolution",
    "Analyze key battles and turning points",
    "Evaluate impact on different groups (colonists, British, Native Americans, enslaved people)",
  ],
  
  assessments: [
    { type: "formative", frequency: "daily" },
    { type: "project", description: "Historical newspaper" },
    { type: "test", description: "Unit test on key concepts" },
  ]
})

// Returns 15 lessons with progression
{
  unitTitle: "The American Revolution: From Protest to Independence",
  lessons: [
    { day: 1, title: "Causes of Revolution: Taxation Without Representation" },
    { day: 2, title: "The Boston Tea Party: Protest or Vandalism?" },
    // ... 13 more lessons
  ],
  progression: "Each lesson builds on previous, scaffolded complexity",
  assessments: [...],
  resources: [...]
}
```

### Standards Coverage Tracking

```typescript
const coverageAnalysis = await lessonPlanAgent.analyzeStandardsCoverage({
  lessonsPlanned: teacherLessons,
  gradeLevel: "5",
  subject: "Mathematics",
  timeframe: "semester",
})

// Returns standards coverage report
{
  totalStandards: 24,
  standardsCovered: 20,
  coveragePercentage: 83,
  
  missingStandards: [
    { id: "5.NBT.A.1", title: "Understand place value..." },
    { id: "5.MD.C.3", title: "Recognize volume..." },
    // ...
  ],
  
  recommendations: [
    "Add lesson on place value to thousands place",
    "Include volume activity in measurement unit",
  ]
}
```

---

## Quality Assurance

### Lesson Plan Review Checklist

AI-generated plans include quality indicators:

```typescript
qualityMetrics: {
  standardsAlignment: "strong",         // weak/moderate/strong
  ageAppropriateness: "appropriate",    // inappropriate/appropriate
  timeAllocation: "realistic",          // unrealistic/tight/realistic/generous
  differentiationQuality: "excellent",  // minimal/adequate/excellent
  assessmentIntegration: "strong",      // weak/moderate/strong
  
  suggestions: [
    "Consider adding more hands-on activities for kinesthetic learners",
    "Closure activity could be extended by 2-3 minutes for deeper reflection",
  ],
  
  overallQuality: 92                    // 0-100 score
}
```

### Teacher Feedback Loop

```typescript
// Teachers rate and improve plans
await lessonPlanAgent.submitFeedback({
  lessonPlanId: "lp-12345",
  rating: 4,                            // 1-5 stars
  usedInClass: true,
  effectiveness: "very-effective",
  
  improvements: [
    "I added a gallery walk activity that worked great",
    "Extended guided practice by 5 minutes - students needed it",
    "Used different manipulatives that students liked better",
  ],
  
  wouldRecommend: true,
})

// AI learns from feedback to improve future generation
```

---

## Cost Management

### Typical Costs

| Lesson Type | Complexity | Tokens | Cost | Provider |
|-------------|-----------|--------|------|----------|
| Simple Lesson | Low | 2000-3000 | $0.06-0.10 | Gemini |
| Standard Lesson | Medium | 4000-6000 | $0.12-0.20 | GPT-4 |
| Detailed Lesson | High | 6000-10000 | $0.20-0.35 | GPT-4 |
| Unit Plan (15 lessons) | Very High | 15000-25000 | $0.50-0.80 | GPT-4 |

**Monthly Teacher Estimate**: $10-20 (weekly planning)

### Cost Optimization

- Use **Gemini Pro** for simple lesson outlines
- Use **GPT-4 Turbo** for detailed, standards-aligned plans
- Generate outline first, then add details in second pass
- Reuse and adapt existing plans when possible

---

## Analytics

### Planning Analytics

```typescript
interface PlanningAnalytics {
  // Usage
  totalPlansGenerated: number
  plansUsedInClass: number
  usageRate: number
  
  // Quality
  averageRating: number
  averageEffectiveness: number
  teacherSatisfaction: number
  
  // Time savings
  estimatedPlanningTime: number         // Hours without AI
  actualTimeSpent: number               // Hours with AI
  timeSaved: number
  
  // Standards coverage
  standardsCoveredCount: number
  standardsCoveragePercentage: number
  
  // Popular templates
  templateUsage: {
    template: string
    count: number
    percentage: number
  }[]
}
```

---

## Best Practices

### For Teachers

1. **Start with Clear Prompts**: Be specific about grade, topic, duration, student needs
2. **Review and Customize**: AI generates draft - add your personal touch
3. **Use Exemplars**: Save your favorite plans as templates
4. **Iterate**: Generate multiple versions and combine best parts
5. **Provide Feedback**: Rate plans to improve future generation
6. **Build Library**: Save and organize plans for reuse

### For Department Heads

1. **Share Plans**: Create shared lesson plan library
2. **Collaborate**: Teachers can build on each other's AI-generated plans
3. **Standardize**: Align planning templates across department
4. **Quality Review**: Review sample AI plans for quality
5. **Professional Development**: Train teachers on effective prompting

---

## Related Documentation

- [AI Integration Architecture](../architecture/ai-integration.md)
- [Curriculum Management](./curriculum.md)
- [Standards Alignment](./standards.md)
- [Assessment Integration](./assessment.md)

---

**Lesson Planner Status**: ✅ Production Feature  
**Supported Templates**: 7 pedagogical frameworks  
**Standards Frameworks**: Common Core, Ontario, IB, State-specific  
**Average Time Savings**: 85%  
**Teacher Satisfaction**: Very High
