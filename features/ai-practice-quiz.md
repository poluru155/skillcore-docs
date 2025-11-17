# AI Practice & Quiz Generation

**Last Updated**: November 17, 2025  
**Status**: Production Feature  
**Version**: 1.0

---

## Overview

AI Practice & Quiz Generation creates personalized practice sessions and assessments with adaptive difficulty, instant feedback, progressive hints, and automatic answer key generation.

**Key Benefits**:
- 🎯 **Personalized Practice**: Adaptive difficulty based on student mastery
- ⚡ **Instant Generation**: Create quizzes in 8-15 seconds
- 💡 **Smart Hints**: Progressive hint system that guides without giving answers
- 📊 **Auto-Grading**: Automatic answer keys and scoring
- 🔄 **Infinite Variations**: Never run out of practice problems

---

## Features

### 1. Practice Session Agent

**Purpose**: Generate personalized practice problems for individual students

**Capabilities**:
- Adaptive difficulty adjustment
- Targets student weak areas
- Progressive hint system
- Immediate feedback
- Tracks mastery progress

#### Example Usage

```typescript
const practiceSession = await practiceSessionAgent.generatePracticeSession({
  studentId: "student-123",
  subject: "Mathematics",
  topic: "Solving Linear Equations",
  currentMastery: "developing",
  weakAreas: ["two-step equations", "equations with variables on both sides"],
  sessionLength: 30,                  // minutes
  difficulty: "adaptive",
})

// Returns personalized practice session
{
  sessionId: "ps-789",
  estimatedTime: 30,
  problemCount: 12,
  
  problems: [
    {
      id: "prob-1",
      difficulty: "easy",              // Start easy to build confidence
      problem: "Solve for x: 2x + 5 = 13",
      answer: "x = 4",
      steps: [
        "Subtract 5 from both sides: 2x = 8",
        "Divide both sides by 2: x = 4"
      ],
      hints: [
        {
          level: 1,
          hint: "What operation can you do to both sides to isolate the term with x?"
        },
        {
          level: 2,
          hint: "Try subtracting 5 from both sides first"
        },
        {
          level: 3,
          hint: "After subtracting 5, you'll have 2x = 8. What's the next step?"
        }
      ],
      timeLimit: 120,                  // seconds
      pointValue: 5,
    },
    {
      id: "prob-2",
      difficulty: "medium",            // Gradually increase difficulty
      problem: "Solve for x: 3x - 7 = 2x + 5",
      answer: "x = 12",
      steps: [
        "Subtract 2x from both sides: x - 7 = 5",
        "Add 7 to both sides: x = 12"
      ],
      hints: [...],
      relatedToPrevious: true,         // Builds on previous concept
    },
    // ... 10 more problems with adaptive difficulty
  ],
  
  adaptiveSequence: true,              // Difficulty adjusts based on performance
  progressTracking: {
    masteryCheckpoints: [4, 8, 12],    // Check mastery at these problem numbers
    advanceThreshold: 0.8,             // 80% correct to advance difficulty
    reviewThreshold: 0.5,              // Below 50% triggers review
  }
}
```

#### Live Hint System

Students can request hints during practice:

```typescript
// Student stuck on problem, requests hint
const hint = await practiceSessionAgent.provideLiveHint({
  problemId: "prob-2",
  studentAttempt: "x = 2",             // Incorrect attempt
  hintLevel: 1,
})

// Returns contextual hint
{
  hint: "Your answer isn't quite right. Look at the left side of the equation - what operation can you perform to get all the x terms on one side?",
  encouragement: "You're on the right track! Keep thinking about isolating x.",
  revealedAnswer: false,               // Never gives away answer
  suggestedNextStep: "Try moving the 2x term to the left side"
}

// If student requests more help
const hint2 = await practiceSessionAgent.provideLiveHint({
  problemId: "prob-2",
  studentAttempt: "x = 2",
  hintLevel: 2,                        // More specific
})

{
  hint: "Subtract 2x from both sides. This will give you: (3x - 2x) - 7 = 5. Can you simplify the left side?",
  workShown: "3x - 7 = 2x + 5\n-2x      -2x\n___________\nx - 7 = 5",
  nextStep: "Now add 7 to both sides"
}
```

#### Adaptive Difficulty

System automatically adjusts difficulty based on performance:

```typescript
interface AdaptiveBehavior {
  // Student doing well (80%+ correct)
  accelerate: {
    increaseGdifficulty: true,
    skipReview: true,
    introduceAdvancedConcepts: true,
  }
  
  // Student struggling (50-80% correct)
  maintain: {
    keepDifficulty: true,
    providMoreHints: true,
    addReviewProblems: false,
  }
  
  // Student very struggling (<50% correct)
  intervene: {
    decreaseDifficulty: true,
    addReviewProblems: true,
    providWorkedExamples: true,
    suggestTeacherHelp: true,
  }
}
```

---

### 2. Quiz Generator Agent

**Purpose**: Rapidly create quizzes and tests with varied question types

**Capabilities**:
- Multiple question types (MC, short answer, essay, matching, T/F)
- Bloom's Taxonomy distribution
- Automatic answer keys
- Standards alignment
- Difficulty balancing

#### Example Usage

```typescript
const quiz = await quizGeneratorAgent.generateQuickQuiz({
  topic: "Photosynthesis",
  subject: "Biology",
  gradeLevel: "9",
  questionCount: 15,
  timeLimit: 30,                       // minutes
  
  questionTypes: [
    { type: "multiple-choice", count: 8 },
    { type: "short-answer", count: 4 },
    { type: "diagram-label", count: 2 },
    { type: "essay", count: 1 },
  ],
  
  difficulty: "mixed",                 // easy, medium, hard, mixed
  
  bloomsDistribution: {
    "remember": 20,                    // % of questions
    "understand": 30,
    "apply": 25,
    "analyze": 15,
    "evaluate": 10,
  },
  
  includeAnswerKey: true,
  includeRubric: true,
})

// Returns complete quiz
{
  quizId: "quiz-456",
  title: "Photosynthesis Quiz",
  totalPoints: 100,
  estimatedTime: 30,
  
  questions: [
    {
      id: "q1",
      type: "multiple-choice",
      question: "What is the primary purpose of photosynthesis?",
      options: [
        "A) To produce oxygen for animals",
        "B) To convert light energy into chemical energy",
        "C) To remove carbon dioxide from the atmosphere",
        "D) To create chlorophyll"
      ],
      correctAnswer: "B",
      explanation: "Photosynthesis converts light energy (from the sun) into chemical energy stored in glucose molecules. While oxygen is produced as a byproduct, the primary purpose is energy conversion.",
      points: 5,
      bloomsLevel: "understand",
      standardId: "HS-LS1-5",
      difficulty: "medium",
    },
    {
      id: "q2",
      type: "short-answer",
      question: "Explain the role of chlorophyll in photosynthesis.",
      sampleAnswer: "Chlorophyll is a green pigment in plant cells that absorbs light energy, primarily blue and red wavelengths. This absorbed energy is used to power the chemical reactions that convert carbon dioxide and water into glucose and oxygen.",
      rubric: {
        "5 points": "Complete explanation including absorption of light, specific wavelengths, and energy conversion",
        "3-4 points": "Mentions chlorophyll's role in light absorption and some connection to energy",
        "1-2 points": "Vague reference to chlorophyll and light",
        "0 points": "Incorrect or no answer"
      },
      points: 5,
      bloomsLevel: "understand",
      difficulty: "medium",
    },
    {
      id: "q3",
      type: "diagram-label",
      question: "Label the parts of the chloroplast involved in photosynthesis",
      diagram: "[Image of chloroplast]",
      labels: [
        { id: "A", correctAnswer: "Thylakoid", location: "stacked disk structures" },
        { id: "B", correctAnswer: "Stroma", location: "fluid-filled space" },
        { id: "C", correctAnswer: "Granum", location: "stack of thylakoids" },
      ],
      points: 6,
      bloomsLevel: "remember",
      difficulty: "easy",
    },
    {
      id: "q15",
      type: "essay",
      question: "Compare and contrast the light-dependent and light-independent reactions of photosynthesis. In your answer, discuss the location, inputs, outputs, and purpose of each set of reactions.",
      rubric: {
        "20-25 points": "Comprehensive comparison covering all aspects (location, inputs, outputs, purpose) with accurate scientific terminology and clear organization",
        "15-19 points": "Good comparison covering most aspects with minor omissions or errors",
        "10-14 points": "Basic comparison with significant omissions or misconceptions",
        "5-9 points": "Limited understanding, major errors or omissions",
        "0-4 points": "Minimal or no understanding demonstrated"
      },
      points: 25,
      bloomsLevel: "analyze",
      difficulty: "hard",
    }
  ],
  
  answerKey: {
    multipleChoice: [
      { question: 1, answer: "B" },
      { question: 2, answer: "C" },
      // ...
    ],
    shortAnswer: [
      { question: 2, sampleAnswer: "...", rubric: {...} },
      // ...
    ],
    essay: [
      { question: 15, rubric: {...}, exemplar: "..." },
    ]
  },
  
  standards: [
    { id: "HS-LS1-5", title: "Use a model to illustrate how photosynthesis transforms light energy into stored chemical energy" },
    { id: "HS-LS1-6", title: "Construct explanation based on evidence for how carbon, hydrogen, and oxygen cycle among biosphere and atmosphere" },
  ],
  
  difficultyDistribution: {
    easy: 4,
    medium: 8,
    hard: 3,
  },
  
  bloomsDistribution: {
    remember: 3,
    understand: 5,
    apply: 4,
    analyze: 2,
    evaluate: 1,
  }
}
```

---

### 3. Test Organizer Agent

**Purpose**: Create comprehensive tests with blueprints and full alignment

**Capabilities**:
- Test blueprinting
- Standards-based test design
- Question bank management
- Difficulty balancing
- Time allocation

#### Example Usage

```typescript
const test = await testOrganizerAgent.organizeTest({
  subject: "Algebra 1",
  gradeLevel: "9",
  
  standards: [
    { id: "A-REI.B.3", weight: 25 },   // Solving linear equations: 25% of test
    { id: "A-REI.D.10", weight: 20 },  // Graphing equations: 20%
    { id: "A-CED.A.2", weight: 30 },   // Creating equations: 30%
    { id: "A-SSE.A.1", weight: 25 },   // Interpreting expressions: 25%
  ],
  
  testLength: 90,                      // minutes
  totalPoints: 100,
  
  bloomsDistribution: {
    "remember": 10,
    "understand": 20,
    "apply": 40,
    "analyze": 20,
    "evaluate": 10,
  },
  
  questionTypes: {
    "multiple-choice": 40,             // % of points
    "short-answer": 30,
    "problem-solving": 30,
  }
})

// Returns comprehensive test with blueprint
{
  testBlueprint: {
    totalQuestions: 25,
    totalPoints: 100,
    estimatedTime: 90,
    
    sectionBreakdown: [
      {
        section: "Multiple Choice",
        questionCount: 15,
        points: 40,
        estimatedTime: 30,
        standards: [
          { id: "A-REI.B.3", questions: 4, points: 10 },
          { id: "A-REI.D.10", questions: 3, points: 8 },
          { id: "A-CED.A.2", questions: 5, points: 12 },
          { id: "A-SSE.A.1", questions: 3, points: 10 },
        ]
      },
      {
        section: "Short Answer",
        questionCount: 6,
        points: 30,
        estimatedTime: 30,
        // ...
      },
      {
        section: "Problem Solving",
        questionCount: 4,
        points: 30,
        estimatedTime: 30,
        // ...
      }
    ]
  },
  
  questions: [...],                    // Full question set
  answerKey: {...},
  rubrics: {...},
  
  standardsAlignment: {
    "A-REI.B.3": {
      questionIds: ["q1", "q2", "q8", "q16"],
      totalPoints: 25,
      percentOfTest: 25,
    },
    // ... other standards
  },
  
  difficultyBalance: {
    easy: { count: 8, points: 20, percentage: 20 },
    medium: { count: 12, points: 50, percentage: 50 },
    hard: { count: 5, points: 30, percentage: 30 },
  }
}
```

---

## Adaptive Features

### Intelligent Problem Selection

AI selects problems based on student history:

```typescript
interface ProblemSelectionStrategy {
  // Student profile
  studentMastery: number              // 0-1
  recentPerformance: number[]         // Last 10 problems
  weakAreas: string[]
  strongAreas: string[]
  learningStyle: string
  
  // Selection rules
  startEasy: boolean                  // Build confidence
  targetWeaknesses: boolean           // Focus on gaps
  spiralReview: boolean               // Include past topics
  incrementalDifficulty: boolean      // Gradual progression
  
  // Adaptive behavior
  if (recentPerformance.avg > 0.8) {
    difficulty = "increase"
    skipReview = true
  } else if (recentPerformance.avg < 0.5) {
    difficulty = "decrease"
    addWorkedExamples = true
    provideMoreHints = true
  }
}
```

### Progress Tracking

Track mastery in real-time:

```typescript
interface PracticeProgress {
  sessionId: string
  studentId: string
  topic: string
  
  // Performance metrics
  problemsAttempted: number
  problemsCorrect: number
  accuracy: number
  
  // Time metrics
  averageTimePerProblem: number
  totalTimeSpent: number
  
  // Mastery tracking
  startingMastery: number
  currentMastery: number
  masteryGain: number
  
  // Adaptive tracking
  difficultyProgression: number[]     // Difficulty of each problem
  hintUsage: number                   // Hints requested
  
  // Recommendations
  suggestedNextTopic: string
  readyForAssessment: boolean
  needsTeacherHelp: boolean
}
```

---

## Question Types

### Supported Question Types

1. **Multiple Choice**
   - Single correct answer
   - "All that apply" style
   - Plausible distractors
   - Explanation for correct answer

2. **Short Answer**
   - Text-based responses
   - AI grading with sample answer
   - Rubric provided

3. **Essay**
   - Extended response
   - Detailed rubric
   - Exemplar answer

4. **True/False**
   - With explanation requirement
   - Correcting false statements

5. **Matching**
   - Terms to definitions
   - Concepts to examples

6. **Fill in the Blank**
   - Vocabulary
   - Sentence completion

7. **Problem Solving (Math/Science)**
   - Show work required
   - Step-by-step solution
   - Partial credit rubric

8. **Diagram Labeling**
   - Image-based questions
   - Label parts
   - Visual learning

9. **Ordering/Sequencing**
   - Steps in process
   - Timeline events
   - Procedural knowledge

---

## Configuration

### Practice Session Settings

```typescript
interface PracticeSettings {
  // Session parameters
  defaultDuration: number             // minutes
  problemsPerSession: number
  
  // Difficulty settings
  startingDifficulty: 'easy' | 'medium' | 'hard' | 'adaptive'
  allowDifficultyAdjustment: boolean
  difficultyAdjustmentThreshold: number
  
  // Hint settings
  hintsAvailable: boolean
  maxHintsPerProblem: number
  hintPenalty: number                 // Points deducted per hint
  
  // Feedback settings
  immediateFeedback: boolean
  showCorrectAnswer: boolean          // After incorrect attempt
  showExplanation: boolean
  
  // Progress settings
  trackMastery: boolean
  updateStudentProfile: boolean
  notifyTeacher: boolean              // On low performance
}
```

### Quiz Generation Settings

```typescript
interface QuizSettings {
  // Question generation
  defaultQuestionCount: number
  allowedQuestionTypes: QuestionType[]
  
  // Standards settings
  alignToStandards: boolean
  standardsFramework: string
  
  // Blooms settings
  enforceBloomsDistribution: boolean
  defaultBloomsDistribution: Record<BloomsLevel, number>
  
  // Difficulty settings
  difficultyDistribution: 'easy-heavy' | 'balanced' | 'hard-heavy'
  
  // Answer key settings
  includeExplanations: boolean
  includeRubrics: boolean
  generateExemplars: boolean
}
```

---

## Analytics

### Practice Analytics

```typescript
interface PracticeAnalytics {
  studentId: string
  timeframe: string
  
  // Usage
  totalSessions: number
  totalProblemsAttempted: number
  totalTimeSpent: number
  
  // Performance
  overallAccuracy: number
  accuracyByTopic: Record<string, number>
  accuracyTrend: number[]             // Over time
  
  // Mastery progress
  topicsMastered: string[]
  topicsInProgress: string[]
  topicsStruggling: string[]
  
  // Engagement
  averageSessionLength: number
  completionRate: number
  hintUsageRate: number
  
  // Recommendations
  suggestedTopics: string[]
  readyForAssessments: string[]
  needsInterventions: string[]
}
```

---

## Cost Management

### Typical Costs

| Feature | Complexity | Tokens | Cost | Provider |
|---------|-----------|--------|------|----------|
| Practice Session (10 problems) | Medium | 1500-2500 | $0.03-0.08 | DeepSeek/Gemini |
| Quick Quiz (15 questions) | Medium | 3000-5000 | $0.08-0.15 | GPT-4 |
| Comprehensive Test (50 questions) | High | 8000-15000 | $0.25-0.50 | GPT-4 |
| Live Hint | Low | 200-400 | $0.005-0.01 | Gemini |

**Monthly Student Estimate**: $3-8 (regular practice use)  
**Monthly Teacher Estimate**: $8-15 (quiz/test generation)

---

## Best Practices

### For Teachers

1. **Start Small**: Begin with practice sessions before full quizzes
2. **Review First Generation**: Check first quiz for quality
3. **Customize**: Edit questions to match your teaching style
4. **Build Question Bank**: Save good questions for reuse
5. **Monitor Student Practice**: Check progress analytics
6. **Adjust Difficulty**: Ensure questions match student level

### For Students

1. **Use Hints Wisely**: Try problem first before requesting hints
2. **Show Your Work**: Practice showing work even when not required
3. **Review Explanations**: Read explanations for incorrect answers
4. **Track Progress**: Monitor your mastery growth
5. **Practice Regularly**: Short, frequent sessions beat cramming

---

## Related Documentation

- [AI Auto-Grading](./ai-auto-grading.md)
- [Assessment System](./assessment.md)
- [Student Analytics](./analytics.md)
- [AI Integration Architecture](../architecture/ai-integration.md)

---

**Practice & Quiz Status**: ✅ Production Feature  
**Question Types**: 9 types supported  
**Adaptive Difficulty**: Real-time adjustment  
**Cost Efficiency**: High (DeepSeek/Gemini for most)  
**Student Engagement**: Very High
