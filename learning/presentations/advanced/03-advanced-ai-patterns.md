# Advanced 03: Advanced AI Patterns

**Duration**: 45-50 minutes
**Level**: 🚀 Advanced
**Goal**: Master sophisticated AI orchestration patterns, ensemble techniques, and cutting-edge prompt engineering strategies used in production AI systems

---

## 🎯 What You'll Learn

By the end of this tutorial, you'll:
- Implement multi-model orchestration and ensemble techniques
- Master advanced prompt engineering patterns (Chain-of-Thought, Tree-of-Thought, ReAct)
- Design adaptive AI workflows with model switching
- Build sophisticated agent orchestration systems
- Understand streaming response management and content filtering

---

## 📋 Prerequisites

- ✅ Expert-level understanding of LLM architectures and limitations
- ✅ Mastery of concurrent programming patterns in Go
- ✅ Deep knowledge of the Crush AI integration system
- ✅ Experience with production AI systems and their challenges
- ✅ Understanding of AI safety and alignment principles

---

## 🧠 Multi-Model Orchestration Architecture

Crush implements sophisticated AI orchestration that goes far beyond simple model switching:

```
┌─────────────────────────────────────────────────────────────┐
│                AI Orchestration Engine                     │
│  ┌─────────────────────┬─────────────────────┬─────────────┐ │
│  │ Model Coordinator   │ Response Synthesizer │ Quality Gate│ │
│  │ - Model selection   │ - Ensemble voting    │ - Confidence│ │
│  │ - Load balancing    │ - Response merging   │ - Validation│ │
│  │ - Fallback chains   │ - Conflict resolution│ - Filtering │ │
│  └─────────────────────┴─────────────────────┴─────────────┘ │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                Provider Ecosystem                           │
│  ┌─────────┬─────────┬─────────┬─────────┬─────────────────┐ │
│  │ OpenAI  │Anthropic│ Groq    │ Local   │ Specialized     │ │
│  │ Models  │ Claude  │ Models  │ Models  │ Models          │ │
│  │ GPT-4o  │ Sonnet  │ Llama   │ Ollama  │ Code/Math/etc   │ │
│  └─────────┴─────────┴─────────┴─────────┴─────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎭 Ensemble AI Patterns

### Response Ensemble and Voting

```bash
# Examine the ensemble implementation
grep -A 20 "EnsembleResponse" internal/llm/provider/
```

```go
// Advanced ensemble response system
type EnsembleOrchestrator struct {
    providers map[string]Provider
    strategies map[string]EnsembleStrategy

    // Quality metrics
    qualityThreshold float64
    confidenceModel  *ConfidencePredictor

    // Performance tracking
    modelPerformance map[string]*PerformanceMetrics

    // Response filtering
    contentFilter *ContentFilter
    safetyChecker *SafetyChecker
}

type EnsembleStrategy interface {
    CombineResponses(ctx context.Context, responses []*Response) (*Response, error)
    GetRequiredModels() []string
    GetConfidenceThreshold() float64
}

// Voting-based ensemble
type VotingEnsemble struct {
    votingMethod VotingMethod
    weights      map[string]float64
}

type VotingMethod int

const (
    MajorityVoting VotingMethod = iota
    WeightedVoting
    BayesianVoting
    ExpertCommittee
)

func (v *VotingEnsemble) CombineResponses(ctx context.Context, responses []*Response) (*Response, error) {
    if len(responses) < 2 {
        return nil, fmt.Errorf("need at least 2 responses for ensemble")
    }

    switch v.votingMethod {
    case MajorityVoting:
        return v.majorityVote(responses)
    case WeightedVoting:
        return v.weightedVote(responses)
    case BayesianVoting:
        return v.bayesianVote(responses)
    case ExpertCommittee:
        return v.expertCommitteeVote(responses)
    default:
        return nil, fmt.Errorf("unknown voting method")
    }
}

func (v *VotingEnsemble) majorityVote(responses []*Response) (*Response, error) {
    // Group similar responses
    groups := v.groupSimilarResponses(responses)

    // Find the largest group
    var largestGroup []*Response
    for _, group := range groups {
        if len(group) > len(largestGroup) {
            largestGroup = group
        }
    }

    if len(largestGroup) == 0 {
        return responses[0], nil // Fallback to first response
    }

    // Select the best response from the majority group
    return v.selectBestFromGroup(largestGroup), nil
}

func (v *VotingEnsemble) weightedVote(responses []*Response) (*Response, error) {
    // Calculate weighted scores for each response
    scores := make(map[*Response]float64)

    for _, response := range responses {
        weight := v.weights[response.Provider]
        if weight == 0 {
            weight = 1.0 // Default weight
        }

        // Factor in response quality metrics
        qualityScore := v.calculateQualityScore(response)
        scores[response] = weight * qualityScore
    }

    // Find highest scoring response
    var bestResponse *Response
    var highestScore float64

    for response, score := range scores {
        if score > highestScore {
            highestScore = score
            bestResponse = response
        }
    }

    return bestResponse, nil
}

func (v *VotingEnsemble) bayesianVote(responses []*Response) (*Response, error) {
    // Implement Bayesian model combination
    // Prior probabilities based on historical performance
    priors := v.calculatePriors(responses)

    // Likelihood of each response being correct
    likelihoods := v.calculateLikelihoods(responses)

    // Posterior probabilities using Bayes' theorem
    posteriors := make(map[*Response]float64)

    for response, prior := range priors {
        likelihood := likelihoods[response]
        posteriors[response] = prior * likelihood
    }

    // Normalize posteriors
    sum := 0.0
    for _, posterior := range posteriors {
        sum += posterior
    }

    for response := range posteriors {
        posteriors[response] /= sum
    }

    // Select response with highest posterior probability
    var bestResponse *Response
    var highestPosterior float64

    for response, posterior := range posteriors {
        if posterior > highestPosterior {
            highestPosterior = posterior
            bestResponse = response
        }
    }

    return bestResponse, nil
}

// Expert committee uses specialized models for different aspects
func (v *VotingEnsemble) expertCommitteeVote(responses []*Response) (*Response, error) {
    // Assign expert roles to different models
    experts := map[string][]string{
        "technical_accuracy": {"gpt-4", "claude-3-sonnet"},
        "clarity":           {"gpt-4o", "claude-3-haiku"},
        "creativity":        {"gpt-4", "claude-3-opus"},
        "safety":           {"claude-3-sonnet", "gpt-4"},
    }

    // Score each response by expert committee
    expertScores := make(map[*Response]map[string]float64)

    for _, response := range responses {
        expertScores[response] = make(map[string]float64)

        for aspect, expertModels := range experts {
            // Check if this response came from an expert model for this aspect
            if slices.Contains(expertModels, response.Provider) {
                score := v.evaluateAspect(response, aspect)
                expertScores[response][aspect] = score
            }
        }
    }

    // Aggregate expert scores
    finalScores := make(map[*Response]float64)
    for response, aspects := range expertScores {
        totalScore := 0.0
        for _, score := range aspects {
            totalScore += score
        }
        finalScores[response] = totalScore / float64(len(aspects))
    }

    // Select highest scoring response
    var bestResponse *Response
    var highestScore float64

    for response, score := range finalScores {
        if score > highestScore {
            highestScore = score
            bestResponse = response
        }
    }

    return bestResponse, nil
}
```

### Adaptive Model Selection

```go
// Intelligent model router based on context analysis
type AdaptiveModelRouter struct {
    // Model capabilities matrix
    capabilities map[string]ModelCapabilities

    // Performance history
    performanceDB *PerformanceDatabase

    // Context analyzer
    contextAnalyzer *ContextAnalyzer

    // Cost optimizer
    costOptimizer *CostOptimizer
}

type ModelCapabilities struct {
    // Core capabilities
    MaxContextLength int
    SupportsCode     bool
    SupportsVision   bool
    SupportsFunction bool

    // Performance characteristics
    Speed           float64 // tokens/second
    Accuracy        float64 // 0-1 score
    Reasoning       float64 // reasoning capability score
    Creativity      float64 // creative task performance

    // Cost metrics
    InputCostPer1K  float64
    OutputCostPer1K float64

    // Specializations
    Specializations []string // "code", "math", "creative", "analysis"
}

func (r *AdaptiveModelRouter) SelectOptimalModel(ctx context.Context, task *Task) (string, error) {
    // Analyze task context
    context := r.contextAnalyzer.AnalyzeTask(task)

    // Get candidate models
    candidates := r.getCandidateModels(context)

    // Score each candidate
    scores := make(map[string]float64)

    for _, modelName := range candidates {
        score := r.scoreModel(modelName, context)
        scores[modelName] = score
    }

    // Apply cost constraints if specified
    if context.CostConstraint != nil {
        scores = r.costOptimizer.ApplyConstraints(scores, context.CostConstraint)
    }

    // Select highest scoring model
    bestModel := r.selectBestModel(scores)

    // Log selection reasoning for analysis
    r.logModelSelection(task, bestModel, scores, context)

    return bestModel, nil
}

func (r *AdaptiveModelRouter) scoreModel(modelName string, context *TaskContext) float64 {
    capabilities := r.capabilities[modelName]

    score := 0.0

    // Context length requirements
    if context.EstimatedTokens > capabilities.MaxContextLength {
        return 0.0 // Hard constraint
    }

    // Task type matching
    switch context.TaskType {
    case TaskTypeCode:
        if capabilities.SupportsCode {
            score += capabilities.Accuracy * 0.4
            score += capabilities.Speed * 0.3
            score += r.getCodeSpecializationBonus(modelName) * 0.3
        }
    case TaskTypeAnalysis:
        score += capabilities.Reasoning * 0.5
        score += capabilities.Accuracy * 0.3
        score += capabilities.Speed * 0.2
    case TaskTypeCreative:
        score += capabilities.Creativity * 0.6
        score += capabilities.Reasoning * 0.2
        score += capabilities.Speed * 0.2
    case TaskTypeMath:
        score += r.getMathPerformance(modelName) * 0.7
        score += capabilities.Reasoning * 0.3
    }

    // Historical performance adjustment
    historicalScore := r.performanceDB.GetAverageScore(modelName, context.TaskType)
    score = score*0.7 + historicalScore*0.3

    // Recency bias - prefer models with recent good performance
    recentPerformance := r.performanceDB.GetRecentPerformance(modelName, 24*time.Hour)
    if recentPerformance > 0 {
        score *= (1.0 + recentPerformance*0.1)
    }

    return score
}

// Dynamic fallback chains
type FallbackChain struct {
    primary   string
    fallbacks []FallbackRule
}

type FallbackRule struct {
    Model     string
    Condition FallbackCondition
    Timeout   time.Duration
}

type FallbackCondition interface {
    ShouldFallback(response *Response, err error) bool
}

// Error-based fallback
type ErrorFallback struct {
    ErrorTypes []error
}

func (e *ErrorFallback) ShouldFallback(response *Response, err error) bool {
    if err == nil {
        return false
    }

    for _, errorType := range e.ErrorTypes {
        if errors.Is(err, errorType) {
            return true
        }
    }

    return false
}

// Quality-based fallback
type QualityFallback struct {
    MinQuality     float64
    QualityChecker func(*Response) float64
}

func (q *QualityFallback) ShouldFallback(response *Response, err error) bool {
    if err != nil || response == nil {
        return true
    }

    quality := q.QualityChecker(response)
    return quality < q.MinQuality
}

func (r *AdaptiveModelRouter) ExecuteWithFallback(ctx context.Context, task *Task, chain *FallbackChain) (*Response, error) {
    // Try primary model first
    response, err := r.executeTask(ctx, task, chain.primary)

    if err == nil && response != nil {
        // Check if response meets quality requirements
        for _, rule := range chain.fallbacks {
            if rule.Condition.ShouldFallback(response, err) {
                // Try fallback model
                fallbackCtx, cancel := context.WithTimeout(ctx, rule.Timeout)
                defer cancel()

                fallbackResponse, fallbackErr := r.executeTask(fallbackCtx, task, rule.Model)
                if fallbackErr == nil && !rule.Condition.ShouldFallback(fallbackResponse, fallbackErr) {
                    return fallbackResponse, nil
                }
            }
        }

        return response, nil
    }

    // Primary failed, try fallbacks
    for _, rule := range chain.fallbacks {
        if rule.Condition.ShouldFallback(response, err) {
            fallbackCtx, cancel := context.WithTimeout(ctx, rule.Timeout)
            defer cancel()

            fallbackResponse, fallbackErr := r.executeTask(fallbackCtx, task, rule.Model)
            if fallbackErr == nil {
                return fallbackResponse, nil
            }
        }
    }

    return nil, fmt.Errorf("all models in fallback chain failed")
}
```

---

## 🔗 Advanced Prompt Engineering Patterns

### Chain-of-Thought and Tree-of-Thought

```go
// Chain-of-Thought prompt engineering
type ChainOfThoughtPrompt struct {
    problem     string
    examples    []CoTExample
    stepGuides  []string
    validation  ValidationStrategy
}

type CoTExample struct {
    Problem   string
    Reasoning []string
    Answer    string
}

func (cot *ChainOfThoughtPrompt) BuildPrompt() string {
    var prompt strings.Builder

    prompt.WriteString("You are an expert problem solver. For each problem, think step by step and show your reasoning.\n\n")

    // Add examples
    for i, example := range cot.examples {
        prompt.WriteString(fmt.Sprintf("Example %d:\n", i+1))
        prompt.WriteString(fmt.Sprintf("Problem: %s\n", example.Problem))
        prompt.WriteString("Reasoning:\n")

        for j, step := range example.Reasoning {
            prompt.WriteString(fmt.Sprintf("%d. %s\n", j+1, step))
        }

        prompt.WriteString(fmt.Sprintf("Answer: %s\n\n", example.Answer))
    }

    // Add step guides
    if len(cot.stepGuides) > 0 {
        prompt.WriteString("When solving problems, follow these steps:\n")
        for i, guide := range cot.stepGuides {
            prompt.WriteString(fmt.Sprintf("%d. %s\n", i+1, guide))
        }
        prompt.WriteString("\n")
    }

    // Add the actual problem
    prompt.WriteString(fmt.Sprintf("Now solve this problem:\nProblem: %s\n", cot.problem))
    prompt.WriteString("Reasoning:\n")

    return prompt.String()
}

// Tree-of-Thought for complex reasoning
type TreeOfThoughtSolver struct {
    problem      string
    branchFactor int
    maxDepth     int
    evaluator    ThoughtEvaluator
    selector     BranchSelector
}

type ThoughtNode struct {
    thought    string
    depth      int
    score      float64
    parent     *ThoughtNode
    children   []*ThoughtNode
    isComplete bool
}

type ThoughtEvaluator interface {
    EvaluateThought(ctx context.Context, node *ThoughtNode) (float64, error)
    IsComplete(node *ThoughtNode) bool
}

type BranchSelector interface {
    SelectBranches(nodes []*ThoughtNode, k int) []*ThoughtNode
}

func (tot *TreeOfThoughtSolver) Solve(ctx context.Context) (*ThoughtNode, error) {
    // Initialize root node
    root := &ThoughtNode{
        thought: fmt.Sprintf("Let me think about how to solve: %s", tot.problem),
        depth:   0,
        score:   0.0,
    }

    // BFS through thought tree
    queue := []*ThoughtNode{root}

    for len(queue) > 0 && queue[0].depth < tot.maxDepth {
        current := queue[0]
        queue = queue[1:]

        // Generate child thoughts
        children, err := tot.generateChildThoughts(ctx, current)
        if err != nil {
            continue
        }

        // Evaluate each child
        for _, child := range children {
            score, err := tot.evaluator.EvaluateThought(ctx, child)
            if err != nil {
                continue
            }
            child.score = score
            current.children = append(current.children, child)

            // Check if this thought completes the solution
            if tot.evaluator.IsComplete(child) {
                return child, nil
            }
        }

        // Select best branches to continue exploring
        if len(current.children) > 0 {
            selected := tot.selector.SelectBranches(current.children, tot.branchFactor)
            queue = append(queue, selected...)
        }
    }

    // Return best leaf node if no complete solution found
    return tot.findBestLeaf(root), nil
}

func (tot *TreeOfThoughtSolver) generateChildThoughts(ctx context.Context, parent *ThoughtNode) ([]*ThoughtNode, error) {
    // Build prompt for generating next thoughts
    prompt := tot.buildThoughtGenerationPrompt(parent)

    // Use LLM to generate multiple possible next thoughts
    response, err := tot.llm.GenerateResponse(ctx, prompt)
    if err != nil {
        return nil, err
    }

    // Parse multiple thoughts from response
    thoughts := tot.parseThoughts(response.Content)

    // Create child nodes
    var children []*ThoughtNode
    for _, thought := range thoughts {
        child := &ThoughtNode{
            thought: thought,
            depth:   parent.depth + 1,
            parent:  parent,
        }
        children = append(children, child)
    }

    return children, nil
}

// ReAct (Reasoning and Acting) pattern
type ReActAgent struct {
    tools     map[string]Tool
    memory    *ConversationMemory
    planner   *ActionPlanner
    reflector *ReflectionEngine
}

type ReActStep struct {
    StepType StepType
    Content  string
    Result   string
}

type StepType int

const (
    StepThought StepType = iota
    StepAction
    StepObservation
    StepReflection
)

func (r *ReActAgent) SolveTask(ctx context.Context, task string) ([]*ReActStep, error) {
    var steps []*ReActStep
    maxSteps := 10

    // Initial thought
    thought := fmt.Sprintf("I need to solve: %s. Let me think about this step by step.", task)
    steps = append(steps, &ReActStep{
        StepType: StepThought,
        Content:  thought,
    })

    for i := 0; i < maxSteps; i++ {
        // Determine next action
        action, err := r.planner.PlanNextAction(ctx, task, steps)
        if err != nil {
            return steps, err
        }

        if action.Type == "finish" {
            break
        }

        // Execute action
        actionStep := &ReActStep{
            StepType: StepAction,
            Content:  action.Description,
        }
        steps = append(steps, actionStep)

        // Get observation
        observation, err := r.executeAction(ctx, action)
        if err != nil {
            observation = fmt.Sprintf("Error executing action: %s", err.Error())
        }

        obsStep := &ReActStep{
            StepType: StepObservation,
            Content:  observation,
        }
        steps = append(steps, obsStep)

        // Reflect on progress
        reflection, err := r.reflector.Reflect(ctx, task, steps)
        if err == nil {
            reflStep := &ReActStep{
                StepType: StepReflection,
                Content:  reflection,
            }
            steps = append(steps, reflStep)
        }

        // Think about next step
        nextThought, err := r.generateNextThought(ctx, task, steps)
        if err == nil {
            thoughtStep := &ReActStep{
                StepType: StepThought,
                Content:  nextThought,
            }
            steps = append(steps, thoughtStep)
        }
    }

    return steps, nil
}
```

### Prompt Template System

```go
// Advanced prompt template engine
type PromptTemplateEngine struct {
    templates map[string]*PromptTemplate
    functions map[string]TemplateFunction
    cache     *TemplateCache
}

type PromptTemplate struct {
    Name        string
    Template    string
    Variables   []TemplateVariable
    Validators  []TemplateValidator
    Transforms  []TemplateTransform
    Examples    []PromptExample
}

type TemplateVariable struct {
    Name        string
    Type        VariableType
    Required    bool
    Default     interface{}
    Description string
}

type PromptExample struct {
    Input       map[string]interface{}
    Expected    string
    Description string
}

func (pte *PromptTemplateEngine) RenderTemplate(templateName string, variables map[string]interface{}) (string, error) {
    template, exists := pte.templates[templateName]
    if !exists {
        return "", fmt.Errorf("template %s not found", templateName)
    }

    // Validate required variables
    if err := pte.validateVariables(template, variables); err != nil {
        return "", fmt.Errorf("variable validation failed: %w", err)
    }

    // Apply transforms
    transformedVars, err := pte.applyTransforms(template, variables)
    if err != nil {
        return "", fmt.Errorf("applying transforms: %w", err)
    }

    // Render template
    tmpl, err := text_template.New(templateName).Funcs(pte.getFunctionMap()).Parse(template.Template)
    if err != nil {
        return "", fmt.Errorf("parsing template: %w", err)
    }

    var buf bytes.Buffer
    if err := tmpl.Execute(&buf, transformedVars); err != nil {
        return "", fmt.Errorf("executing template: %w", err)
    }

    return buf.String(), nil
}

// Example: Code analysis template
const codeAnalysisTemplate = `
You are an expert code reviewer with deep knowledge of {{.language}} and software engineering best practices.

Analyze the following {{.language}} code and provide a comprehensive review:

## Code to Review:
` + "```{{.language}}\n{{.code}}\n```" + `

## Review Criteria:
{{range .criteria}}
- {{.}}
{{end}}

Please provide your analysis in the following format:

### Overall Assessment
[Provide a summary of the code quality and main observations]

### Detailed Analysis
{{if .include_security}}
#### Security Issues
[Identify any security vulnerabilities or concerns]
{{end}}

{{if .include_performance}}
#### Performance Considerations
[Analyze performance implications and potential optimizations]
{{end}}

#### Code Quality
[Assess code structure, readability, and maintainability]

#### Best Practices
[Evaluate adherence to {{.language}} best practices and conventions]

### Recommendations
[Provide specific, actionable recommendations for improvement]

### Rating: [Score from 1-10 with brief justification]
`

// Template for step-by-step problem solving
const stepByStepTemplate = `
You are helping solve a {{.problem_type}} problem. Break down your solution into clear, logical steps.

Problem: {{.problem}}

{{if .context}}
Context: {{.context}}
{{end}}

{{if .constraints}}
Constraints:
{{range .constraints}}
- {{.}}
{{end}}
{{end}}

Please solve this step by step:

1. **Understanding**: First, rephrase the problem in your own words to confirm understanding.

2. **Analysis**: Break down the problem into smaller components or identify key elements.

3. **Strategy**: Outline your approach or method for solving this problem.

4. **Solution**: Implement your strategy step by step, showing your work.

5. **Verification**: Check your solution and verify it meets all requirements.

{{if .show_alternatives}}
6. **Alternatives**: Briefly discuss alternative approaches or solutions.
{{end}}

Remember to:
- Show your reasoning at each step
- Explain any assumptions you make
- Double-check your work
{{if .include_code}}
- Provide working code examples where applicable
{{end}}
`
```

---

## 🔄 Streaming Response Management

### Advanced Response Processing

```go
// Sophisticated streaming response handler
type StreamingResponseManager struct {
    processors []StreamProcessor
    filters    []ContentFilter
    aggregator *ResponseAggregator

    // Real-time analysis
    sentimentAnalyzer *SentimentAnalyzer
    qualityChecker   *QualityChecker
    toxicityFilter   *ToxicityFilter

    // Stream control
    rateLimiter  *rate.Limiter
    bufferSize   int
    flushTimeout time.Duration
}

type StreamProcessor interface {
    ProcessChunk(ctx context.Context, chunk *ResponseChunk) (*ResponseChunk, error)
    ShouldTerminate(chunk *ResponseChunk) bool
    GetMetadata() map[string]interface{}
}

// Real-time content filtering during streaming
type RealTimeContentFilter struct {
    rules      []FilterRule
    classifier *ContentClassifier

    // Dynamic filtering
    adaptiveThreshold float64
    userPreferences   *UserPreferences
}

func (f *RealTimeContentFilter) ProcessChunk(ctx context.Context, chunk *ResponseChunk) (*ResponseChunk, error) {
    // Classify content in real-time
    classification, err := f.classifier.ClassifyContent(chunk.Content)
    if err != nil {
        return chunk, nil // Continue on classification errors
    }

    // Apply filtering rules
    for _, rule := range f.rules {
        if rule.Matches(classification) {
            action := rule.GetAction(classification.Confidence)

            switch action {
            case FilterActionBlock:
                return nil, fmt.Errorf("content blocked by filter: %s", rule.Reason)
            case FilterActionModify:
                chunk.Content = rule.ModifyContent(chunk.Content)
            case FilterActionWarn:
                chunk.Warnings = append(chunk.Warnings, rule.Reason)
            case FilterActionFlag:
                chunk.Flags = append(chunk.Flags, rule.Flag)
            }
        }
    }

    return chunk, nil
}

// Stream aggregation for multi-model responses
type MultiModelStreamAggregator struct {
    streams     map[string]<-chan *ResponseChunk
    weights     map[string]float64
    strategy    AggregationStrategy

    // Synchronization
    syncWindow  time.Duration
    buffers     map[string][]*ResponseChunk

    // Quality metrics
    qualityFunc func(string) float64
}

func (a *MultiModelStreamAggregator) AggregateStreams(ctx context.Context) (<-chan *ResponseChunk, error) {
    output := make(chan *ResponseChunk, 100)

    go func() {
        defer close(output)

        ticker := time.NewTicker(a.syncWindow)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                // Process synchronized chunks
                a.processSyncWindow(output)
            }
        }
    }()

    // Start readers for each stream
    for modelName, stream := range a.streams {
        go a.readStream(ctx, modelName, stream)
    }

    return output, nil
}

func (a *MultiModelStreamAggregator) processSyncWindow(output chan<- *ResponseChunk) {
    // Collect chunks from all models in this time window
    allChunks := make(map[string][]*ResponseChunk)

    for modelName, buffer := range a.buffers {
        if len(buffer) > 0 {
            allChunks[modelName] = make([]*ResponseChunk, len(buffer))
            copy(allChunks[modelName], buffer)
            a.buffers[modelName] = nil // Clear buffer
        }
    }

    if len(allChunks) == 0 {
        return
    }

    // Aggregate based on strategy
    switch a.strategy {
    case AggregationWeightedAverage:
        aggregated := a.weightedAverageAggregation(allChunks)
        if aggregated != nil {
            output <- aggregated
        }
    case AggregationBestQuality:
        aggregated := a.bestQualityAggregation(allChunks)
        if aggregated != nil {
            output <- aggregated
        }
    case AggregationEnsemble:
        aggregated := a.ensembleAggregation(allChunks)
        if aggregated != nil {
            output <- aggregated
        }
    }
}

func (a *MultiModelStreamAggregator) weightedAverageAggregation(chunks map[string][]*ResponseChunk) *ResponseChunk {
    if len(chunks) == 0 {
        return nil
    }

    // Find the model with most content to use as base
    var baseModel string
    var maxContent int

    for model, modelChunks := range chunks {
        totalContent := 0
        for _, chunk := range modelChunks {
            totalContent += len(chunk.Content)
        }
        if totalContent > maxContent {
            maxContent = totalContent
            baseModel = model
        }
    }

    if baseModel == "" {
        return nil
    }

    // Use base model's structure, but weight the content
    baseChunks := chunks[baseModel]
    if len(baseChunks) == 0 {
        return nil
    }

    aggregated := &ResponseChunk{
        Content:   a.weightedContentMerge(chunks),
        Metadata:  make(map[string]interface{}),
        Timestamp: time.Now(),
    }

    // Aggregate metadata
    for model, weight := range a.weights {
        if modelChunks, exists := chunks[model]; exists && len(modelChunks) > 0 {
            aggregated.Metadata[fmt.Sprintf("%s_weight", model)] = weight
            aggregated.Metadata[fmt.Sprintf("%s_chunks", model)] = len(modelChunks)
        }
    }

    return aggregated
}

func (a *MultiModelStreamAggregator) weightedContentMerge(chunks map[string][]*ResponseChunk) string {
    // Advanced content merging logic
    // This is a simplified version - production would use NLP techniques

    var allContent []string
    var weights []float64

    for model, modelChunks := range chunks {
        weight := a.weights[model]
        if weight == 0 {
            weight = 1.0
        }

        for _, chunk := range modelChunks {
            if chunk.Content != "" {
                allContent = append(allContent, chunk.Content)
                weights = append(weights, weight)
            }
        }
    }

    if len(allContent) == 0 {
        return ""
    }

    // Simple weighted selection - choose content from highest weighted model
    bestIndex := 0
    bestWeight := weights[0]

    for i, weight := range weights {
        if weight > bestWeight {
            bestWeight = weight
            bestIndex = i
        }
    }

    return allContent[bestIndex]
}
```

---

## 🎯 What's Next

In the next advanced tutorial, we'll explore **Protocol Implementation** where we'll:
- Deep dive into JSON-RPC 2.0 and custom protocol design
- Master WebSocket and SSE implementations
- Learn protocol versioning and backward compatibility
- Understand distributed system communication patterns

---

## 📝 Expert-Level Exercises

1. **Ensemble Implementation**: Build a complete voting ensemble system with Bayesian inference
2. **Custom Prompt Pattern**: Design and implement a novel prompt engineering pattern
3. **Adaptive Router**: Create an intelligent model router with ML-based performance prediction
4. **Stream Processing**: Build a real-time content analysis pipeline for streaming responses
5. **Multi-Agent System**: Design a sophisticated multi-agent collaboration framework

---

**Previous**: [Advanced 02: Extension Architecture](02-extension-architecture.md)
**Next**: [Advanced 04: Protocol Implementation](04-protocol-implementation.md)

---

*🧠 **AI Insight**: The future of AI lies not in single models, but in sophisticated orchestration of multiple AI systems working together intelligently.*
