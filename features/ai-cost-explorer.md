# AI Cost Explorer

**Last Updated**: November 17, 2025  
**Status**: Production Feature  
**Version**: 1.0

---

## Overview

The AI Cost Explorer provides comprehensive visibility into AI usage costs, predictions, and optimization opportunities across the SkillCore platform. It enables administrators, teachers, and users to monitor spending, compare providers, predict future costs, and optimize AI usage for maximum value.

**Key Features**:
- Real-time cost tracking across all AI providers
- Cost prediction engine with 30-day historical analysis
- Provider cost comparison and recommendations
- Agent-specific cost breakdowns
- Budget alerts and limits
- Usage analytics and optimization insights

---

## Table of Contents

- [Cost Tracking Architecture](#cost-tracking-architecture)
- [Provider Cost Analysis](#provider-cost-analysis)
- [Agent Cost Breakdown](#agent-cost-breakdown)
- [Cost Prediction](#cost-prediction)
- [Budget Management](#budget-management)
- [Optimization Strategies](#optimization-strategies)
- [Cost Explorer UI](#cost-explorer-ui)
- [API Reference](#api-reference)

---

## Cost Tracking Architecture

### Usage Recording

Every AI request is tracked with detailed cost information:

```typescript
interface AIUsageRecord {
  // Identity
  id: string
  userId: string
  userRole: UserRole
  sessionId: string
  
  // AI Configuration
  agentName: string
  provider: AIProviderType
  model: string
  
  // Usage Metrics
  promptTokens: number
  completionTokens: number
  totalTokens: number
  
  // Cost Calculation
  inputCost: number                   // Prompt tokens cost
  outputCost: number                  // Completion tokens cost
  cost: number                        // Total cost in USD
  
  // Performance
  latency: number                     // Response time in ms
  success: boolean
  error?: string
  
  // Context
  timestamp: Date
  schoolId: string
  districtId: string
  
  // Metadata
  requestMetadata?: Record<string, any>
}
```

### Cost Calculation

```typescript
export class CostCalculator {
  /**
   * Calculate cost for AI request based on provider pricing
   */
  calculateCost(params: {
    provider: AIProviderType
    model: string
    promptTokens: number
    completionTokens: number
  }): number {
    const config = this.getProviderConfig(params.provider)
    const modelPricing = this.getModelPricing(params.provider, params.model)
    
    const inputCost = (params.promptTokens / 1000) * modelPricing.inputPer1k
    const outputCost = (params.completionTokens / 1000) * modelPricing.outputPer1k
    
    return inputCost + outputCost
  }
  
  /**
   * Get provider-specific pricing
   */
  private getModelPricing(provider: AIProviderType, model: string): {
    inputPer1k: number
    outputPer1k: number
  } {
    const pricingMap = {
      [AIProviderType.OPENAI]: {
        'gpt-4-turbo-preview': { inputPer1k: 0.01, outputPer1k: 0.03 },
        'gpt-4': { inputPer1k: 0.03, outputPer1k: 0.06 },
        'gpt-3.5-turbo': { inputPer1k: 0.0005, outputPer1k: 0.0015 },
      },
      [AIProviderType.ANTHROPIC]: {
        'claude-3-opus': { inputPer1k: 0.015, outputPer1k: 0.075 },
        'claude-3-sonnet': { inputPer1k: 0.003, outputPer1k: 0.015 },
        'claude-3-haiku': { inputPer1k: 0.00025, outputPer1k: 0.00125 },
      },
      [AIProviderType.GOOGLE]: {
        'gemini-pro': { inputPer1k: 0.00035, outputPer1k: 0.0014 },
        'gemini-ultra': { inputPer1k: 0.001, outputPer1k: 0.004 },
      },
      [AIProviderType.DEEPSEEK]: {
        'deepseek-chat': { inputPer1k: 0.0014, outputPer1k: 0.0028 },
        'deepseek-coder': { inputPer1k: 0.0014, outputPer1k: 0.0028 },
      },
    }
    
    return pricingMap[provider][model]
  }
}
```

---

## Provider Cost Analysis

### Provider Comparison Table

Real-time cost comparison across all providers:

```typescript
interface ProviderCostComparison {
  provider: AIProviderType
  
  // Usage Statistics
  totalRequests: number
  totalTokens: number
  
  // Cost Statistics
  totalCost: number
  avgCostPerRequest: number
  avgCostPerToken: number
  
  // Performance Statistics
  avgLatency: number
  successRate: number
  errorRate: number
  
  // Efficiency
  costEfficiency: number              // Cost per successful request
  performanceScore: number            // Combined metric (0-100)
  
  // Recommendation
  recommended: boolean
  recommendationReason: string
}

export class ProviderAnalyzer {
  async compareProviders(
    timeframe: 'day' | 'week' | 'month',
    userId?: string
  ): Promise<ProviderCostComparison[]> {
    const providers = [
      AIProviderType.OPENAI,
      AIProviderType.ANTHROPIC,
      AIProviderType.GOOGLE,
      AIProviderType.DEEPSEEK,
    ]
    
    const comparisons = await Promise.all(
      providers.map(provider => this.analyzeProvider(provider, timeframe, userId))
    )
    
    // Sort by cost efficiency
    return comparisons.sort((a, b) => a.costEfficiency - b.costEfficiency)
  }
  
  private async analyzeProvider(
    provider: AIProviderType,
    timeframe: string,
    userId?: string
  ): Promise<ProviderCostComparison> {
    const startDate = this.getStartDate(timeframe)
    
    const stats = await prisma.aIUsageLog.aggregate({
      where: {
        provider,
        timestamp: { gte: startDate },
        ...(userId && { userId }),
      },
      _sum: {
        totalTokens: true,
        cost: true,
      },
      _count: true,
      _avg: {
        latency: true,
        cost: true,
      },
    })
    
    const successRate = await this.calculateSuccessRate(provider, startDate, userId)
    
    return {
      provider,
      totalRequests: stats._count,
      totalTokens: stats._sum.totalTokens || 0,
      totalCost: stats._sum.cost || 0,
      avgCostPerRequest: stats._avg.cost || 0,
      avgCostPerToken: (stats._sum.cost || 0) / (stats._sum.totalTokens || 1),
      avgLatency: stats._avg.latency || 0,
      successRate,
      errorRate: 1 - successRate,
      costEfficiency: (stats._sum.cost || 0) / (stats._count * successRate || 1),
      performanceScore: this.calculatePerformanceScore({
        successRate,
        latency: stats._avg.latency || 0,
        cost: stats._avg.cost || 0,
      }),
      recommended: this.isRecommended(provider, stats, successRate),
      recommendationReason: this.getRecommendationReason(provider, stats),
    }
  }
}
```

### Provider Cost Dashboard

Visual comparison of providers:

```typescript
interface ProviderDashboardData {
  // Overview
  totalSpend: number
  totalRequests: number
  avgCostPerRequest: number
  
  // By Provider
  providers: {
    provider: AIProviderType
    percentage: number                // % of total spend
    cost: number
    requests: number
    trend: 'increasing' | 'stable' | 'decreasing'
  }[]
  
  // Recommendations
  costOptimizationOpportunities: {
    currentProvider: AIProviderType
    suggestedProvider: AIProviderType
    potentialSavings: number
    reason: string
  }[]
}
```

---

## Agent Cost Breakdown

### Agent Usage Analysis

Cost breakdown by AI agent:

```typescript
interface AgentCostAnalysis {
  agentName: string
  
  // Usage
  totalRequests: number
  totalTokens: number
  
  // Cost
  totalCost: number
  avgCostPerRequest: number
  percentageOfTotal: number
  
  // Provider Distribution
  providers: {
    provider: AIProviderType
    requests: number
    cost: number
  }[]
  
  // Performance
  avgLatency: number
  successRate: number
  
  // Optimization
  optimizationSuggestions: string[]
}

export class AgentCostAnalyzer {
  async analyzeAgentCosts(
    timeframe: 'day' | 'week' | 'month',
    userId?: string
  ): Promise<AgentCostAnalysis[]> {
    const agents = [
      'AutoGradingAgent',
      'LessonPlanGeneratorAgent',
      'PracticeSessionAgent',
      'QuizOrTestAgent',
      'StudentAnalyzerAgent',
      'DifferentiationAdvisorAgent',
      'TestOrganizerAgent',
    ]
    
    const analyses = await Promise.all(
      agents.map(agent => this.analyzeAgent(agent, timeframe, userId))
    )
    
    return analyses.sort((a, b) => b.totalCost - a.totalCost)
  }
  
  private async analyzeAgent(
    agentName: string,
    timeframe: string,
    userId?: string
  ): Promise<AgentCostAnalysis> {
    const startDate = this.getStartDate(timeframe)
    
    // Get aggregate stats
    const stats = await prisma.aIUsageLog.aggregate({
      where: {
        agentName,
        timestamp: { gte: startDate },
        ...(userId && { userId }),
      },
      _sum: { totalTokens: true, cost: true },
      _count: true,
      _avg: { latency: true, cost: true },
    })
    
    // Get provider distribution
    const providerBreakdown = await this.getProviderBreakdown(
      agentName, 
      startDate, 
      userId
    )
    
    // Calculate total spend across all agents for percentage
    const totalSpend = await this.getTotalSpend(startDate, userId)
    
    return {
      agentName,
      totalRequests: stats._count,
      totalTokens: stats._sum.totalTokens || 0,
      totalCost: stats._sum.cost || 0,
      avgCostPerRequest: stats._avg.cost || 0,
      percentageOfTotal: ((stats._sum.cost || 0) / totalSpend) * 100,
      providers: providerBreakdown,
      avgLatency: stats._avg.latency || 0,
      successRate: await this.calculateSuccessRate(agentName, startDate, userId),
      optimizationSuggestions: this.generateOptimizationSuggestions(
        agentName, 
        stats, 
        providerBreakdown
      ),
    }
  }
}
```

### Agent Cost Comparison

Compare costs for different agent combinations:

```typescript
interface AgentCombinationAnalysis {
  combination: string[]
  
  // Historical Performance
  totalRequests: number
  totalCost: number
  avgCostPerCombination: number
  
  // Provider Usage
  providerDistribution: Record<AIProviderType, number>
  
  // Optimization Potential
  currentCost: number
  optimizedCost: number
  potentialSavings: number
  optimizationSteps: string[]
}

export class AgentCombinationAnalyzer {
  /**
   * Analyze cost efficiency of common agent combinations
   * Example: Teacher using AutoGrading + LessonPlanner together
   */
  async analyzeCombinations(
    timeframe: 'day' | 'week' | 'month'
  ): Promise<AgentCombinationAnalysis[]> {
    // Common combinations based on user role
    const combinations = [
      // Teacher workflow
      ['AutoGradingAgent', 'LessonPlanGeneratorAgent'],
      ['QuizOrTestAgent', 'AutoGradingAgent'],
      
      // Student support workflow
      ['PracticeSessionAgent', 'StudentAnalyzerAgent'],
      
      // Counselor workflow
      ['StudentAnalyzerAgent', 'DifferentiationAdvisorAgent'],
      
      // Full teaching workflow
      ['LessonPlanGeneratorAgent', 'QuizOrTestAgent', 'AutoGradingAgent'],
    ]
    
    return await Promise.all(
      combinations.map(combo => this.analyzeCombination(combo, timeframe))
    )
  }
}
```

---

## Cost Prediction

### Prediction Engine

AI-powered cost forecasting based on historical trends:

```typescript
interface CostPrediction {
  // Prediction Period
  period: 'day' | 'week' | 'month'
  
  // Provider-Specific Predictions
  provider: AIProviderType
  
  // Historical Basis
  basedOnLastNDays: number
  historicalAvgDailyCost: number
  
  // Predictions
  estimatedDailyCost: number
  estimatedWeeklyCost: number
  estimatedMonthlyCost: number
  
  // Trend Analysis
  trend: 'increasing' | 'stable' | 'decreasing'
  trendPercentage: number             // % change
  
  // Confidence
  confidence: number                  // 0-100%
  confidenceLevel: 'low' | 'medium' | 'high'
  
  // Agent Breakdown
  breakdown: {
    agentName: string
    percentage: number
    estimatedCost: number
  }[]
}

export class CostPredictionEngine {
  async predictCosts(
    userId: string,
    providers: AIProviderType[]
  ): Promise<CostPrediction[]> {
    const predictions = await Promise.all(
      providers.map(provider => this.predictProviderCost(userId, provider))
    )
    
    return predictions.sort((a, b) => b.estimatedMonthlyCost - a.estimatedMonthlyCost)
  }
  
  private async predictProviderCost(
    userId: string,
    provider: AIProviderType
  ): Promise<CostPrediction> {
    // Get 30 days of historical data
    const historicalData = await this.getHistoricalUsage(userId, provider, 30)
    
    if (historicalData.length === 0) {
      return this.getZeroPrediction(provider)
    }
    
    // Calculate daily costs
    const dailyCosts = this.aggregateDailyCosts(historicalData)
    
    // Time series analysis
    const avgDailyCost = this.calculateAverage(dailyCosts)
    const trend = this.calculateTrend(dailyCosts)
    const trendPercentage = this.calculateTrendPercentage(dailyCosts)
    
    // Project forward with trend
    const estimatedDailyCost = avgDailyCost * (1 + trend)
    
    // Agent breakdown
    const breakdown = this.getAgentBreakdown(historicalData, estimatedDailyCost)
    
    return {
      period: 'month',
      provider,
      basedOnLastNDays: 30,
      historicalAvgDailyCost: avgDailyCost,
      estimatedDailyCost,
      estimatedWeeklyCost: estimatedDailyCost * 7,
      estimatedMonthlyCost: estimatedDailyCost * 30,
      trend: this.getTrendDirection(trend),
      trendPercentage,
      confidence: this.calculateConfidence(dailyCosts),
      confidenceLevel: this.getConfidenceLevel(this.calculateConfidence(dailyCosts)),
      breakdown,
    }
  }
  
  private calculateTrend(dailyCosts: number[]): number {
    // Linear regression to detect trend
    const n = dailyCosts.length
    if (n < 7) return 0  // Need at least 7 days for trend
    
    const x = Array.from({ length: n }, (_, i) => i)
    const y = dailyCosts
    
    const sumX = x.reduce((a, b) => a + b, 0)
    const sumY = y.reduce((a, b) => a + b, 0)
    const sumXY = x.reduce((sum, xi, i) => sum + xi * y[i], 0)
    const sumXX = x.reduce((sum, xi) => sum + xi * xi, 0)
    
    const slope = (n * sumXY - sumX * sumY) / (n * sumXX - sumX * sumX)
    
    return slope
  }
  
  private calculateConfidence(dailyCosts: number[]): number {
    if (dailyCosts.length < 7) return 30
    if (dailyCosts.length < 14) return 60
    
    // Calculate coefficient of variation
    const mean = this.calculateAverage(dailyCosts)
    const stdDev = this.calculateStdDev(dailyCosts)
    const cv = (stdDev / mean) * 100
    
    // Lower CV = higher confidence
    if (cv < 20) return 95
    if (cv < 40) return 80
    if (cv < 60) return 65
    return 50
  }
}
```

### Prediction Scenarios

What-if analysis for cost planning:

```typescript
interface PredictionScenario {
  name: string
  description: string
  
  // Assumptions
  requestsPerDay: number
  agentDistribution: Record<string, number>
  providerPreference: AIProviderType
  
  // Predictions
  estimatedDailyCost: number
  estimatedMonthlyCost: number
  
  // Comparison
  vsCurrentCost: number
  savingsOpportunity: number
}

export class ScenarioAnalyzer {
  /**
   * Generate cost predictions for different usage scenarios
   */
  async generateScenarios(userId: string): Promise<PredictionScenario[]> {
    const currentUsage = await this.getCurrentUsage(userId)
    
    return [
      // Scenario 1: Light usage
      await this.predictScenario({
        name: 'Light Usage',
        description: 'Minimal AI assistance (10 requests/day)',
        requestsPerDay: 10,
        agentDistribution: {
          'AutoGradingAgent': 0.4,
          'LessonPlanGeneratorAgent': 0.3,
          'QuizOrTestAgent': 0.3,
        },
        providerPreference: AIProviderType.DEEPSEEK,
      }),
      
      // Scenario 2: Moderate usage
      await this.predictScenario({
        name: 'Moderate Usage',
        description: 'Regular AI assistance (25 requests/day)',
        requestsPerDay: 25,
        agentDistribution: {
          'AutoGradingAgent': 0.35,
          'LessonPlanGeneratorAgent': 0.25,
          'QuizOrTestAgent': 0.2,
          'PracticeSessionAgent': 0.2,
        },
        providerPreference: AIProviderType.GOOGLE,
      }),
      
      // Scenario 3: Heavy usage
      await this.predictScenario({
        name: 'Heavy Usage',
        description: 'Extensive AI integration (50+ requests/day)',
        requestsPerDay: 50,
        agentDistribution: {
          'AutoGradingAgent': 0.3,
          'LessonPlanGeneratorAgent': 0.2,
          'QuizOrTestAgent': 0.15,
          'PracticeSessionAgent': 0.15,
          'StudentAnalyzerAgent': 0.2,
        },
        providerPreference: AIProviderType.ANTHROPIC,
      }),
    ]
  }
}
```

---

## Budget Management

### Budget Configuration

Set spending limits at various levels:

```typescript
interface BudgetConfiguration {
  // Budget Levels
  dailyLimit?: number                 // USD per day
  weeklyLimit?: number                // USD per week
  monthlyLimit?: number               // USD per month
  
  // Scope
  scope: 'user' | 'school' | 'district'
  scopeId: string
  
  // Alert Thresholds
  alertAt50Percent: boolean
  alertAt75Percent: boolean
  alertAt90Percent: boolean
  
  // Actions
  hardStop: boolean                   // Stop AI requests when limit reached
  notifyUsers: string[]               // User IDs to notify
  
  // Metadata
  createdAt: Date
  updatedAt: Date
  createdBy: string
}

export class BudgetManager {
  async setBudget(config: BudgetConfiguration): Promise<void> {
    await prisma.aIBudget.upsert({
      where: {
        scope_scopeId: {
          scope: config.scope,
          scopeId: config.scopeId,
        },
      },
      create: config,
      update: config,
    })
  }
  
  async checkBudget(
    userId: string,
    estimatedCost: number
  ): Promise<{
    allowed: boolean
    reason?: string
    currentSpend: number
    limit: number
    remaining: number
  }> {
    // Check user-level budget
    const userBudget = await this.getUserBudget(userId)
    if (userBudget) {
      const allowed = await this.checkBudgetLimit(
        userBudget,
        userId,
        estimatedCost
      )
      if (!allowed.allowed) return allowed
    }
    
    // Check school-level budget
    const user = await this.getUser(userId)
    const schoolBudget = await this.getSchoolBudget(user.schoolId)
    if (schoolBudget) {
      const allowed = await this.checkBudgetLimit(
        schoolBudget,
        user.schoolId,
        estimatedCost
      )
      if (!allowed.allowed) return allowed
    }
    
    // Check district-level budget
    const districtBudget = await this.getDistrictBudget(user.districtId)
    if (districtBudget) {
      return await this.checkBudgetLimit(
        districtBudget,
        user.districtId,
        estimatedCost
      )
    }
    
    return { allowed: true, currentSpend: 0, limit: Infinity, remaining: Infinity }
  }
}
```

### Budget Alerts

Automated alerts when approaching limits:

```typescript
export class BudgetAlertService {
  async checkAndAlert(
    scopeId: string,
    newCost: number
  ): Promise<void> {
    const budget = await this.getBudget(scopeId)
    if (!budget) return
    
    const currentSpend = await this.getCurrentSpend(scopeId, 'month')
    const newTotal = currentSpend + newCost
    const percentUsed = (newTotal / budget.monthlyLimit) * 100
    
    // Check thresholds
    if (percentUsed >= 90 && budget.alertAt90Percent) {
      await this.sendAlert({
        level: 'critical',
        scopeId,
        percentUsed,
        currentSpend: newTotal,
        limit: budget.monthlyLimit,
        message: `90% of monthly AI budget reached`,
      })
    } else if (percentUsed >= 75 && budget.alertAt75Percent) {
      await this.sendAlert({
        level: 'warning',
        scopeId,
        percentUsed,
        currentSpend: newTotal,
        limit: budget.monthlyLimit,
        message: `75% of monthly AI budget reached`,
      })
    } else if (percentUsed >= 50 && budget.alertAt50Percent) {
      await this.sendAlert({
        level: 'info',
        scopeId,
        percentUsed,
        currentSpend: newTotal,
        limit: budget.monthlyLimit,
        message: `50% of monthly AI budget reached`,
      })
    }
  }
}
```

---

## Optimization Strategies

### Cost Optimization Recommendations

AI-powered recommendations to reduce costs:

```typescript
interface OptimizationRecommendation {
  type: 'provider-switch' | 'agent-optimization' | 'usage-pattern'
  priority: 'high' | 'medium' | 'low'
  
  // Current State
  currentCost: number
  
  // Optimized State
  optimizedCost: number
  potentialSavings: number
  savingsPercentage: number
  
  // Recommendation
  title: string
  description: string
  steps: string[]
  
  // Impact
  impact: {
    qualityChange: 'none' | 'minimal' | 'moderate'
    latencyChange: number               // ms difference
    successRateChange: number           // % difference
  }
}

export class CostOptimizer {
  async generateRecommendations(
    userId: string
  ): Promise<OptimizationRecommendation[]> {
    const recommendations: OptimizationRecommendation[] = []
    
    // 1. Provider optimization
    const providerRec = await this.analyzeProviderOptimization(userId)
    if (providerRec) recommendations.push(providerRec)
    
    // 2. Agent usage optimization
    const agentRec = await this.analyzeAgentOptimization(userId)
    if (agentRec) recommendations.push(agentRec)
    
    // 3. Usage pattern optimization
    const patternRec = await this.analyzeUsagePatterns(userId)
    if (patternRec) recommendations.push(patternRec)
    
    return recommendations.sort((a, b) => b.potentialSavings - a.potentialSavings)
  }
  
  private async analyzeProviderOptimization(
    userId: string
  ): Promise<OptimizationRecommendation | null> {
    // Analyze current provider usage
    const usage = await this.getProviderUsage(userId, 30)
    
    // Find cost-effective alternatives
    const currentProvider = this.getMostUsedProvider(usage)
    const alternatives = await this.findCheaperAlternatives(currentProvider, usage)
    
    if (alternatives.length === 0) return null
    
    const best = alternatives[0]
    
    return {
      type: 'provider-switch',
      priority: best.savingsPercentage > 30 ? 'high' : 'medium',
      currentCost: usage.totalCost,
      optimizedCost: best.estimatedCost,
      potentialSavings: usage.totalCost - best.estimatedCost,
      savingsPercentage: ((usage.totalCost - best.estimatedCost) / usage.totalCost) * 100,
      title: `Switch from ${currentProvider} to ${best.provider}`,
      description: `Save ${best.savingsPercentage.toFixed(0)}% by switching to ${best.provider} for similar quality`,
      steps: [
        'Review quality comparison',
        'Update agent configurations',
        'Monitor first week for quality',
      ],
      impact: {
        qualityChange: 'minimal',
        latencyChange: best.latencyDiff,
        successRateChange: best.successRateDiff,
      },
    }
  }
}
```

### Usage Pattern Analysis

Identify inefficient usage patterns:

```typescript
export class UsagePatternAnalyzer {
  async analyzePatterns(userId: string): Promise<{
    inefficientPatterns: {
      pattern: string
      frequency: number
      wastedCost: number
      suggestion: string
    }[]
    optimizationOpportunities: number
  }> {
    const usage = await this.getUsageHistory(userId, 30)
    
    const patterns = [
      // Pattern 1: Repeated similar requests
      await this.detectRepeatedRequests(usage),
      
      // Pattern 2: Over-prompting (too many tokens)
      await this.detectOverPrompting(usage),
      
      // Pattern 3: Using expensive models for simple tasks
      await this.detectModelMisuse(usage),
      
      // Pattern 4: Redundant agent calls
      await this.detectRedundantCalls(usage),
    ]
    
    const inefficientPatterns = patterns.filter(p => p.wastedCost > 1)
    const totalSavings = inefficientPatterns.reduce((sum, p) => sum + p.wastedCost, 0)
    
    return {
      inefficientPatterns,
      optimizationOpportunities: totalSavings,
    }
  }
}
```

---

## Cost Explorer UI

### Dashboard Components

```typescript
// Main Cost Explorer Dashboard
interface CostExplorerDashboard {
  // Overview Cards
  overview: {
    todaySpend: number
    monthSpend: number
    projectedMonthSpend: number
    vsLastMonth: number                 // % change
  }
  
  // Provider Breakdown Chart
  providerChart: {
    type: 'pie' | 'bar'
    data: {
      provider: AIProviderType
      cost: number
      percentage: number
    }[]
  }
  
  // Agent Breakdown Chart
  agentChart: {
    type: 'bar' | 'treemap'
    data: {
      agent: string
      cost: number
      requests: number
    }[]
  }
  
  // Trend Chart
  trendChart: {
    type: 'line'
    period: 'day' | 'week' | 'month'
    data: {
      date: Date
      cost: number
      requests: number
    }[]
  }
  
  // Predictions
  predictions: CostPrediction[]
  
  // Recommendations
  recommendations: OptimizationRecommendation[]
}
```

### Interactive Features

- **Time Range Selector**: Day, Week, Month, Custom
- **Provider Filter**: Toggle providers on/off
- **Agent Filter**: Toggle agents on/off
- **Cost/Usage Toggle**: Switch between cost view and usage view
- **Export**: CSV, PDF reports
- **Budget Status**: Visual budget consumption bar

---

## API Reference

### GraphQL Queries

```graphql
type Query {
  # Cost tracking
  aiUsageStats(
    userId: ID
    schoolId: ID
    districtId: ID
    timeframe: Timeframe!
  ): AIUsageStats!
  
  # Cost predictions
  aiCostPrediction(
    userId: ID!
    providers: [AIProvider!]!
  ): [CostPrediction!]!
  
  # Provider comparison
  aiProviderComparison(
    timeframe: Timeframe!
    userId: ID
  ): [ProviderComparison!]!
  
  # Agent analysis
  aiAgentCostBreakdown(
    timeframe: Timeframe!
    userId: ID
  ): [AgentCostAnalysis!]!
  
  # Optimization
  aiCostOptimization(userId: ID!): [OptimizationRecommendation!]!
  
  # Budget
  aiBudgetStatus(
    scope: BudgetScope!
    scopeId: ID!
  ): BudgetStatus!
}

type Mutation {
  # Budget management
  setAIBudget(input: BudgetInput!): Budget!
  updateAIBudget(id: ID!, input: BudgetInput!): Budget!
  
  # Alerts
  configureAIAlerts(input: AlertConfigInput!): AlertConfig!
}

type Subscription {
  # Real-time cost updates
  aiCostUpdates(userId: ID!): CostUpdate!
  
  # Budget alerts
  aiBudgetAlert(scopeId: ID!): BudgetAlert!
}
```

### REST API Endpoints

```typescript
// Cost tracking
GET /api/ai/costs/stats?timeframe=month&userId=123
GET /api/ai/costs/predictions?userId=123&providers=openai,anthropic

// Provider analysis
GET /api/ai/providers/comparison?timeframe=month
GET /api/ai/providers/:provider/stats

// Agent analysis
GET /api/ai/agents/costs?timeframe=month
GET /api/ai/agents/:agent/analysis

// Optimization
GET /api/ai/optimization/recommendations?userId=123
GET /api/ai/optimization/scenarios

// Budget
GET /api/ai/budget/:scopeId
POST /api/ai/budget
PUT /api/ai/budget/:id
```

---

## Related Documentation

- [AI Integration Architecture](../architecture/ai-integration.md)
- [Budget Management](./budget-management.md)
- [Analytics Dashboard](./analytics.md)
- [Admin Features](./admin-features.md)

---

**Cost Explorer Status**: ✅ Production Feature  
**Real-time Tracking**: Enabled  
**Prediction Engine**: Active  
**Budget Alerts**: Configured  
**Next Steps**: Dashboard UI implementation
