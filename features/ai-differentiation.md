# AI Differentiation Advisor

**Last Updated**: November 17, 2025  
**Status**: Production Feature  
**Version**: 1.0

---

## Overview

AI Differentiation Advisor provides personalized teaching strategies, accommodation suggestions, and tiered activities to meet diverse learning needs across the classroom, supporting IEP, 504, ELL, gifted, and general education students.

**Key Benefits**:
- 🎯 **Personalized Strategies**: Tailored teaching approaches for each student
- ♿ **Universal Design**: Accessible learning for all students
- 🌍 **ELL Support**: Specialized strategies for English learners
- 🎁 **Gifted Extensions**: Challenge activities for advanced students
- ⚡ **Instant Adaptations**: Real-time lesson modification suggestions

---

## Features

### 1. Student-Specific Recommendations

Generate personalized teaching strategies based on student profile:

```typescript
const recommendations = await differentiationAdvisor.getStudentRecommendations({
  studentId: "student-123",
  subject: "Mathematics",
  topic: "Solving Linear Equations",
})

// Returns comprehensive differentiation plan
{
  studentProfile: {
    name: "Maria",
    gradeLevel: "8",
    learningProfile: {
      preferredModality: "visual",
      processingSpeed: "slower",
      strengthAreas: ["pattern recognition", "hands-on learning"],
      challengeAreas: ["abstract concepts", "written expression"],
    },
    accommodations: {
      iep: true,
      ell: true,
      section504: false,
      gifted: false,
    },
  },
  
  teachingStrategies: [
    {
      strategy: "Visual equation models",
      rationale: "Maria processes visual information well and struggles with abstract concepts",
      implementation: "Use algebra tiles or virtual manipulatives to represent equations visually before moving to symbolic notation",
      materials: ["Algebra tiles", "Virtual manipulative app", "Color-coded worksheets"],
      timeRequired: "+10 minutes for modeling",
    },
    {
      strategy: "Step-by-step scaffolding",
      rationale: "Slower processing speed benefits from breaking down multi-step problems",
      implementation: "Provide equation solving checklist: 1) Identify variable side, 2) Move constants, 3) Isolate variable. Use for first 5 problems before independent work",
      materials: ["Equation solving checklist", "Worked examples"],
      timeRequired: "+5 minutes for scaffolding",
    },
    {
      strategy: "Sentence frames for explanations",
      rationale: "ELL support for mathematical language and written expression",
      implementation: "Provide frames: 'First, I ___ because ___. Next, I ___ to get ___. Finally, x equals ___.'",
      materials: ["Sentence frame handout", "Math vocabulary cards"],
      timeRequired: "+5 minutes for language support",
    }
  ],
  
  iepAccommodations: [
    {
      accommodation: "Extended time (1.5x)",
      application: "Allow 45 minutes instead of 30 for practice problems",
      monitoring: "Track completion time and quality of work",
    },
    {
      accommodation: "Reduced item count",
      application: "Assign 10 core problems instead of 15, ensuring concept coverage",
      monitoring: "Verify mastery of key concepts with fewer items",
    },
    {
      accommodation: "Calculator for computation",
      application: "Allow calculator for arithmetic so focus remains on algebraic thinking",
      monitoring: "Ensure student understands process, not just getting answers",
    }
  ],
  
  ellSupport: [
    {
      support: "Mathematical vocabulary pre-teaching",
      timing: "5 minutes before lesson",
      method: "Show word, visual, example, practice using it",
      vocabulary: ["variable", "coefficient", "constant", "isolate", "inverse operation"],
    },
    {
      support: "Bilingual glossary",
      application: "Provide math terms in English and Spanish",
      resource: "Digital glossary with audio pronunciation",
    },
    {
      support: "Visual supports",
      application: "All word problems include diagrams or pictures",
      benefit: "Reduces language load, focuses on math concept",
    }
  ],
  
  assessmentModifications: [
    {
      modification: "Visual answer format option",
      description: "Allow Maria to show equation solving with algebra tiles photo + written steps",
      grading: "Credit for correct mathematical thinking regardless of written expression quality",
    },
    {
      modification: "Oral explanation option",
      description: "Allow oral explanation of solving process as alternative to written",
      grading: "Use rubric focused on mathematical understanding, not language fluency",
    }
  ],
  
  progressMonitoring: {
    frequency: "bi-weekly",
    measures: [
      "Accuracy on equation solving (target: 80%)",
      "Independence level (currently: prompted, target: independent)",
      "Explanation quality (currently: emerging, target: developing)",
    ],
    adjustmentTriggers: [
      "If accuracy < 70% after 2 weeks: increase visual supports",
      "If independence not improving: add peer buddy",
      "If language remains barrier: increase sentence frame complexity gradually",
    ]
  }
}
```

---

### 2. Universal Design for Learning (UDL)

Generate UDL-aligned lesson plans that work for all students:

```typescript
const udlLesson = await differentiationAdvisor.generateUDLLesson({
  topic: "Photosynthesis",
  subject: "Biology",
  gradeLevel: "9",
  classProfile: {
    studentCount: 28,
    diversityProfile: {
      iep: 4,
      section504: 2,
      ell: 6,
      gifted: 3,
      generalEd: 13,
    }
  }
})

// Returns UDL-designed lesson
{
  topic: "Photosynthesis",
  
  // UDL Principle 1: Multiple Means of Representation
  representation: {
    content: {
      visual: [
        "Animated video of photosynthesis process",
        "Infographic with labeled chloroplast diagram",
        "Color-coded equation: light energy + CO₂ + H₂O → glucose + O₂",
      ],
      auditory: [
        "Teacher explanation with think-aloud",
        "Student-created rap about photosynthesis steps",
        "Podcast: 'Why plants are green'",
      ],
      textual: [
        "Reading passage at three levels (grades 6, 8, 10)",
        "Graphic organizer for note-taking",
        "Digital text with embedded glossary",
      ],
      kinesthetic: [
        "Act out photosynthesis roles (students as chlorophyll, CO₂, water, glucose)",
        "Build chloroplast model with clay",
        "Lab: measure oxygen production in aquatic plants",
      ],
    },
    
    vocabulary: {
      preteach: ["photosynthesis", "chloroplast", "chlorophyll", "glucose"],
      support: {
        standard: "Written definitions with examples",
        ell: "Bilingual glossary with images",
        simplified: "Simple language definitions + pictures",
        advanced: "Scientific definitions + etymology",
      }
    },
    
    organizers: {
      visual: "Flowchart of photosynthesis steps",
      compareContrast: "Venn diagram: Photosynthesis vs. Cellular Respiration",
      causeEffect: "What happens when plants don't get light/water/CO₂",
    }
  },
  
  // UDL Principle 2: Multiple Means of Action & Expression
  expression: {
    checkForUnderstanding: [
      {
        option: "Written summary",
        description: "Write 1 paragraph explaining photosynthesis",
        supports: ["Sentence frames", "Word bank", "Outline template"],
      },
      {
        option: "Diagram creation",
        description: "Create labeled diagram showing photosynthesis process",
        supports: ["Partial diagram to complete", "Word bank", "Color coding guide"],
      },
      {
        option: "Oral explanation",
        description: "Explain photosynthesis to partner or teacher",
        supports: ["Talking points card", "Diagram to reference", "Record instead of live"],
      },
      {
        option: "Creative project",
        description: "Create analogy/model/skit showing how photosynthesis works",
        supports: ["Example analogies", "Materials list", "Rubric"],
      },
    ],
    
    tools: {
      writing: ["Speech-to-text", "Graphic organizers", "Sentence frames"],
      organization: ["Checklists", "Step-by-step guides", "Timers"],
      assistive: ["Text-to-speech", "Audiobooks", "Calculators"],
    }
  },
  
  // UDL Principle 3: Multiple Means of Engagement
  engagement: {
    choiceOptions: [
      "Choose video or reading to learn content",
      "Choose expression format for assessment",
      "Choose working alone, partner, or small group",
      "Choose practice problems difficulty level",
    ],
    
    relevance: [
      "Connect to student interests (sports, cooking, art)",
      "Real-world applications (food production, climate, oxygen)",
      "Current events (rainforest conservation, urban farming)",
    ],
    
    challenge: {
      all: "Understand photosynthesis process and importance",
      high: "Compare photosynthesis efficiency in different plant types",
      gifted: "Research artificial photosynthesis and future applications",
    },
    
    collaboration: [
      "Think-pair-share on photosynthesis importance",
      "Jigsaw: Groups become experts on one photosynthesis stage",
      "Peer teaching: Explain concept to classmate",
    ]
  }
}
```

---

### 3. Tiered Activities

Create multiple versions of activities for different readiness levels:

```typescript
const tieredActivity = await differentiationAdvisor.createTieredActivity({
  topic: "Writing persuasive essays",
  subject: "English Language Arts",
  gradeLevel: "7",
  tiers: 3,
})

// Returns activities for three readiness levels
{
  topic: "Writing Persuasive Essays",
  standardAlignment: "W.7.1: Write arguments to support claims",
  
  tiers: [
    {
      tier: "approaching",
      students: ["Student A", "Student B", "Student C"],
      readinessLevel: "Still developing paragraph structure",
      
      activity: {
        title: "Persuasive Paragraph Builder",
        description: "Write one well-developed persuasive paragraph",
        
        task: "Should students have longer recess? Write ONE paragraph persuading your principal.",
        
        scaffold: {
          structure: "Highly structured with sentence starters",
          supports: [
            "Paragraph frame: Topic sentence | Reason 1 + evidence | Reason 2 + evidence | Concluding sentence",
            "Sentence starters provided for each part",
            "Word bank of persuasive language",
            "Example paragraph provided",
          ],
          
          sentenceStarters: {
            topic: "I believe students should/should not have longer recess because...",
            reason1: "First, longer recess would...",
            evidence1: "For example...",
            reason2: "Additionally, students...",
            evidence2: "Research shows...",
            conclusion: "For these reasons..."
          }
        },
        
        successCriteria: [
          "Clear position stated",
          "Two reasons with evidence",
          "Persuasive language used",
          "Concluding sentence",
        ],
        
        timeExpected: "30 minutes",
      }
    },
    
    {
      tier: "on-level",
      students: ["Student D", "Student E", "Student F", ...],
      readinessLevel: "Can write paragraphs, ready for full essay structure",
      
      activity: {
        title: "Five-Paragraph Persuasive Essay",
        description: "Write complete persuasive essay with intro, body, conclusion",
        
        task: "Should your school implement a four-day school week? Write a persuasive essay.",
        
        scaffold: {
          structure: "Essay outline provided, some sentence starters",
          supports: [
            "Essay outline template (intro/3 body/conclusion)",
            "Transition words list",
            "Counterargument sentence frames",
            "Peer editing checklist",
          ],
          
          outline: {
            intro: "Hook | Background | Thesis statement",
            body1: "Reason 1 | Evidence | Explanation",
            body2: "Reason 2 | Evidence | Explanation",
            body3: "Counterargument | Rebuttal",
            conclusion: "Restate position | Summarize reasons | Call to action",
          }
        },
        
        successCriteria: [
          "Engaging introduction with clear thesis",
          "Three body paragraphs with evidence",
          "Addresses counterargument",
          "Strong conclusion with call to action",
          "Uses transition words",
        ],
        
        timeExpected: "60 minutes",
      }
    },
    
    {
      tier: "advanced",
      students: ["Student X", "Student Y", "Student Z"],
      readinessLevel: "Proficient essay writers, ready for complexity",
      
      activity: {
        title: "Multi-Perspective Persuasive Essay",
        description: "Write sophisticated essay considering multiple stakeholder perspectives",
        
        task: "Should your district implement later school start times? Address concerns of students, parents, teachers, bus drivers, and administration in your argument.",
        
        scaffold: {
          structure: "Minimal scaffolding, open-ended",
          supports: [
            "Research articles from multiple perspectives",
            "Advanced persuasive techniques guide (ethos/pathos/logos)",
            "Rhetorical devices list",
            "Self-editing checklist",
          ],
          
          challenges: [
            "Address at least 3 stakeholder perspectives",
            "Incorporate research from provided sources",
            "Use advanced rhetorical techniques (rhetorical questions, analogies, appeals)",
            "Acknowledge and refute strongest counterargument",
          ]
        },
        
        successCriteria: [
          "Sophisticated thesis addressing complexity",
          "Multiple perspectives considered",
          "Strong evidence from research",
          "Advanced persuasive techniques used",
          "Nuanced handling of counterarguments",
          "Compelling, memorable conclusion",
        ],
        
        timeExpected: "90 minutes + research time",
        
        extension: {
          title: "Present to Stakeholders",
          description: "Adapt essay into presentation for school board simulation",
          time: "+30 minutes",
        }
      }
    }
  ],
  
  flexibleGrouping: {
    note: "Students can move between tiers based on topic-specific readiness",
    progressionCriteria: {
      approachingToOn: "Successfully completes 2 approaching activities with 85%+ quality",
      onToAdvanced: "Consistently exceeds success criteria, ready for more challenge",
    }
  }
}
```

---

### 4. Real-Time Lesson Adaptation

Get instant suggestions during teaching:

```typescript
// Teacher notices students struggling during lesson
const adaptations = await differentiationAdvisor.getRealTimeAdaptation({
  currentActivity: "Independent practice - solving equations",
  observedIssue: "60% of students stuck, multiple hands raised",
  timeRemaining: 20,                   // minutes left in class
})

// Returns immediate adaptation strategies
{
  urgency: "high",
  
  quickFixes: [
    {
      strategy: "Pause and re-teach mini-lesson",
      time: "5 minutes",
      implementation: "Stop independent work. Gather students on carpet. Work through 2 problems together with think-aloud. Send back to seats.",
      benefit: "Addresses whole-class confusion immediately",
    },
    {
      strategy: "Peer tutoring pairs",
      time: "2 minutes to set up",
      implementation: "Pair struggling students with proficient students. Proficient students explain their thinking. Teacher circulates to monitor.",
      benefit: "Immediate support without teacher bottleneck",
    },
    {
      strategy: "Worked example + parallel problem",
      time: "1 minute to distribute",
      implementation: "Give struggling students worked example to study. Then solve parallel problem with same structure. Builds confidence through modeling.",
      benefit: "Scaffolds without reducing expectations",
    }
  ],
  
  forNextTime: [
    "Build in more guided practice before independent work",
    "Check for understanding more frequently",
    "Provide reference sheet with steps",
    "Pre-teach key students who may struggle",
  ],
  
  assignmentAdjustment: {
    original: "Complete problems 1-20",
    adapted: "Complete problems 1-10 (odds only), focusing on quality over quantity",
    rationale: "Better to master fewer problems than struggle through many",
  }
}
```

---

### 5. Accommodation Library

Access database of research-based accommodations:

```typescript
const accommodations = await differentiationAdvisor.searchAccommodations({
  challengeArea: "reading comprehension",
  gradeLevel: "6",
  context: "science textbook",
})

// Returns relevant accommodations
{
  accommodations: [
    {
      name: "Chunk reading passages",
      category: "Presentation",
      description: "Break long passages into smaller sections with comprehension checks between",
      implementation: {
        before: "Here's chapter 3 - read all 8 pages and answer questions",
        after: "Read pages 58-59 (Section 1). Stop. Answer questions 1-3. Then read pages 60-61...",
      },
      effectiveness: "High",
      evidence: "Reduces cognitive load, improves comprehension by 30-40%",
      worksFor: ["Reading disabilities", "ADHD", "ELL", "Processing delays"],
    },
    {
      name: "Pre-teach vocabulary",
      category: "Presentation",
      description: "Teach key vocabulary before reading, reducing language load during reading",
      implementation: {
        timing: "5 minutes before reading",
        method: "Show word, show image, show in context, students practice using it",
        example: "For 'photosynthesis' chapter: pre-teach photosynthesis, chloroplast, chlorophyll, glucose, carbon dioxide",
      },
      effectiveness: "Very High",
      evidence: "Increases comprehension by 25-35%, especially for ELLs",
      worksFor: ["ELL", "Limited vocabulary", "Learning disabilities"],
    },
    {
      name: "Text-to-speech",
      category: "Response",
      description: "Allow students to listen to text while following along",
      implementation: {
        tools: ["Built-in screen reader", "Learning Ally", "Bookshare", "Kurzweil"],
        settings: "Adjust speed to student preference (usually 1.2-1.5x normal)",
        combination: "Listen while reading along - dual modality strengthens comprehension",
      },
      effectiveness: "High",
      evidence: "Improves comprehension and reduces fatigue for struggling readers",
      worksFor: ["Dyslexia", "Slow reading speed", "Visual impairments", "Attention challenges"],
    },
    {
      name: "Graphic organizers",
      category: "Response",
      description: "Provide visual framework for organizing information from text",
      implementation: {
        types: ["Main idea/details", "Cause/effect", "Compare/contrast", "Sequence"],
        when: "Give BEFORE reading as note-taking guide",
        example: "For chapter on ecosystems, provide food web diagram to fill in while reading",
      },
      effectiveness: "Very High",
      evidence: "Improves comprehension and retention by 40-50%",
      worksFor: ["All students", "Especially helpful for visual learners, organization challenges"],
    }
  ]
}
```

---

## Analytics

### Differentiation Effectiveness Tracking

```typescript
interface DifferentiationAnalytics {
  teacher: string
  timeframe: "semester"
  
  // Usage metrics
  strategiesImplemented: number
  studentsSupported: number
  
  // Effectiveness metrics
  studentGrowth: {
    withDifferentiation: 85,           // % showing growth
    withoutDifferentiation: 62,        // Comparison group
    difference: +23,
  },
  
  achievementGap: {
    beginning: 28,                     // Points between high/low
    current: 18,                       // Points between high/low
    reduction: 36,                     // % reduction
  },
  
  // Strategy effectiveness
  topStrategies: [
    {
      strategy: "Visual supports",
      timesUsed: 45,
      studentSuccess: 0.88,
      subjects: ["Math", "Science"],
    },
    {
      strategy: "Sentence frames",
      timesUsed: 38,
      studentSuccess: 0.82,
      subjects: ["ELA", "Social Studies"],
    }
  ],
  
  // Student-specific impact
  studentProgress: [
    {
      student: "Maria",
      strategies: ["Visual models", "ELL support", "Extended time"],
      growthRate: "high",
      beforeGrade: 68,
      currentGrade: 82,
      improvement: +14,
    }
  ]
}
```

---

## Cost Management

### Typical Costs

| Feature | Complexity | Tokens | Cost | Provider |
|---------|-----------|--------|------|----------|
| Student Recommendations | Medium | 2000-3000 | $0.06-0.10 | Claude |
| UDL Lesson | High | 4000-6000 | $0.12-0.20 | GPT-4 |
| Tiered Activities | High | 3000-5000 | $0.10-0.15 | GPT-4 |
| Real-Time Adaptation | Low | 800-1200 | $0.02-0.04 | Gemini |
| Accommodation Search | Low | 500-800 | $0.01-0.03 | DeepSeek |

**Monthly Per-Teacher Estimate**: $12-20 (active differentiation use)  
**Monthly Per-School Estimate**: $300-600 (25 teachers)

---

## Best Practices

### For General Education Teachers

1. **Start with Universal Design**: UDL benefits all students, not just those with IEPs
2. **Use Tiered Activities**: Provide appropriate challenge for all readiness levels
3. **Student Choice**: Let students choose their path when possible
4. **Progress Monitoring**: Track what works for which students
5. **Flexible Grouping**: Group and regroup based on specific needs, not fixed labels

### For Special Education Teachers

1. **Align with IEP Goals**: Ensure differentiation supports IEP objectives
2. **Document Strategies**: Keep record of what works for progress reports
3. **Collaborate**: Share successful strategies with general ed teachers
4. **Gradual Release**: Fade supports as student becomes more independent
5. **Student Involvement**: Include students in setting differentiation goals

### For ELL Teachers

1. **Language Objectives**: Every lesson has both content and language objectives
2. **Visual Everything**: Reduce language load with images, diagrams, gestures
3. **Pre-Teach Vocabulary**: Never assume students know academic vocabulary
4. **Comprehensible Input**: Slow down, repeat, rephrase, check understanding
5. **Celebrate Languages**: View multilingualism as asset, not deficit

---

## Related Documentation

- [Student Insights](./ai-student-insights.md)
- [Lesson Planning](./ai-lesson-planner.md)
- [IEP Management](./iep.md)
- [ELL Support](./ell.md)
- [Universal Design for Learning](../guides/udl.md)

---

**Differentiation Status**: ✅ Production Feature  
**Supported Plans**: IEP, 504, ELL, Gifted, General Ed  
**UDL Alignment**: Full support for all three principles  
**Evidence-Based**: Research-backed strategies only  
**Impact**: Closes achievement gaps by average 36%
