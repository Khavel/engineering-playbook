# LLM Token & Cost Optimization

## Problem

LLM API costs scale with:
- **Number of calls** × **tokens per call** × **model rate**

Without optimization, costs escalate through:
- Verbose prompts with unnecessary context
- Full file contents in responses (not diffs)
- Too many retry/fix iterations
- Premium models where cheaper ones suffice
- Unfiltered logs/output in prompts

## Pattern: Measure → Limit → Optimize

### 1. Measure First (Always)

**Don't guess. Track every call.**

```csharp
public class TokenUsage
{
    public string AgentName { get; set; }
    public string Phase { get; set; }
    public int Iteration { get; set; }
    public string Model { get; set; }
    public int InputTokens { get; set; }
    public int OutputTokens { get; set; }
    public DateTime Timestamp { get; set; }
}

public class TokenBudget
{
    private List<TokenUsage> _usages = new();
    
    public int TotalTokens => _usages.Sum(u => u.InputTokens + u.OutputTokens);
    public int TotalCalls => _usages.Count;
    
    public Dictionary<string, int> GetTokensByPhase() { }
    public IEnumerable<TokenUsage> GetTopNByTokens(int n) { }
}
```

**What to measure:**
- Input/output tokens per call
- Which phase/agent/iteration
- Model used
- Timestamp

**Output at end:**
```
=== Token Usage Summary ===
Total Calls: 8, Total Tokens: 45,320
By Phase: Fix=28K (62%), Plan=10K, Implement=7K
Top Burners:
  #1: Builder Fix iteration 3 - 12,450 tokens
  #2: Builder Fix iteration 2 - 9,800 tokens
```

Then you know where to optimize.

### 2. Set Hard Limits (Prevent Runaway)

**Budget enforcement before spending:**

```csharp
public void CheckBudgetOrThrow(Context ctx)
{
    if (ctx.MaxCalls > 0 && ctx.TokenBudget.TotalCalls >= ctx.MaxCalls)
        throw new BudgetExceededException($"Hit limit: {ctx.MaxCalls} calls");
    
    if (ctx.MaxTokens > 0 && ctx.TokenBudget.TotalTokens >= ctx.MaxTokens)
        throw new BudgetExceededException($"Hit limit: {ctx.MaxTokens:N0} tokens");
}

// Call BEFORE every LLM invocation
public async Task<T> CallAgent<T>()
{
    CheckBudgetOrThrow(context);
    return await agent.CallAsync();
}
```

**CLI/config:**
```bash
--maxAgentCalls 15     # Hard cap on calls
--maxTokens 50000      # Hard cap on tokens
```

**Why:** Catches infinite loops, prevents surprises in CI/CD.

### 3. Optimize (Tactical Wins)

#### A. Trim Input Context

**Problem:** Sending "the world" in every prompt.

**Solution:** Budget your context.

```csharp
public static string TrimTestOutput(string output, int maxLines = 150)
{
    var lines = output.Split('\n');
    var failureKeywords = new[] { "Failed!", "Error:", "Assert", "Exception" };
    var noiseKeywords = new[] { "Restore", "Build succeeded", "up-to-date" };
    
    var interesting = new List<(int idx, string line)>();
    
    for (int i = 0; i < lines.Length; i++)
    {
        // Skip noise
        if (noiseKeywords.Any(kw => lines[i].Contains(kw))) continue;
        
        // Keep failures + context (±5 lines)
        if (failureKeywords.Any(kw => lines[i].Contains(kw)))
        {
            for (int j = Math.Max(0, i-5); j <= Math.Min(lines.Length-1, i+5); j++)
                interesting.Add((j, lines[j]));
        }
    }
    
    return string.Join("\n", interesting.OrderBy(x => x.idx).Distinct());
}
```

**Result:** Test output 3KB → 500 bytes (85% reduction).

**Apply to:**
- Test failures (keep only errors + small context)
- Stack traces (dedupe repeated frames)
- Logs (keep only last N lines or errors)
- File diffs (not full files)

#### B. Multi-Tier Testing (Fast → Slow)

**Problem:** Running full test suite on every fix iteration wastes time and API calls.

**Solution:** Smoke test first, full test once.

```csharp
for (int iteration = 0; iteration < maxIterations; iteration++)
{
    // Strategy: fast smoke test during loop, full test at end
    bool isLastIteration = (iteration == maxIterations - 1);
    string testCommand = isLastIteration 
        ? config.FullTestCommand     // "dotnet test"
        : config.SmokeTestCommand;   // "dotnet build" or "dotnet test --filter Category=Smoke"
    
    var result = await RunTest(testCommand);
    
    if (result.Passed && isLastIteration)
        break; // Success
    
    if (!result.Passed)
        await CallFixAgent(result.Output);
}
```

**Smoke test options:**
- `dotnet build --no-restore` (catches compile errors)
- Linting only
- Subset of fast unit tests (`--filter Category=Smoke`)

**Result:** 5 full test runs → 1 full + 4 smoke = 50-80% less test time = fewer fix iterations.

#### C. Model Economics

**Problem:** Using GPT-4 for everything.

**Solution:** Right model for right task.

| Agent | Calls | Recommended Model | Why |
|-------|-------|------------------|-----|
| Planner | 1 | Premium (GPT-4) | Quality matters, runs once |
| Builder (fix loop) | N | Cheaper (GPT-3.5) | Called many times |
| Reviewer | 1 | Medium | Runs once, needs decent quality |

```csharp
var plannerClient = CreateClient(config.PlannerModel ?? "gpt-4");
var builderClient = CreateClient(config.BuilderModel ?? "gpt-3.5-turbo");
```

**CLI:**
```bash
--plannerModel gpt-4 
--builderModel gpt-3.5-turbo
```

**Track actual model used in telemetry** for cost attribution.

#### D. Diff-Based Responses (10x Savings)

**Problem:** LLM returns full file contents after small change.

**Bad:**
```json
{
  "filePath": "Program.cs",
  "content": "<entire 500-line file with one line changed>"
}
```
Output tokens: ~2000

**Good:**
```json
{
  "filePath": "Program.cs",
  "editType": "patch",
  "content": "@@ -42,1 +42,1 @@\n-    var x = 0;\n+    var x = 1;"
}
```
Output tokens: ~50

**Implementation:**
```csharp
public enum EditType { FullFile, Patch, Delete }

public class Edit
{
    public string FilePath { get; set; }
    public EditType Type { get; set; }
    public string Content { get; set; } // Full content or unified diff
}

public void ApplyEdit(Edit edit)
{
    if (edit.Type == EditType.FullFile)
        File.WriteAllText(edit.FilePath, edit.Content);
    else if (edit.Type == EditType.Patch)
        ApplyPatch(edit.FilePath, edit.Content);
}
```

**Prompt engineering:**
```
IMPORTANT: Return only changed files as unified diffs.
Do NOT return full file contents unless the file is new.

Format:
{
  "filePath": "src/File.cs",
  "editType": "patch",
  "content": "@@ -10,3 +10,3 @@\n- old line\n+ new line"
}
```

#### E. Stop Redundant Calls

**Problem:** Calling expensive agents unnecessarily.

**Examples:**
- ❌ Calling Reviewer on every fix iteration → ✅ Call once at end
- ❌ Rebuilding plan on every retry → ✅ Plan once, fix incrementally
- ❌ Fetching repo context every call → ✅ Cache and reuse

```csharp
// Bad: Review after every fix
for (int i = 0; i < maxIters; i++)
{
    await builder.Fix();
    await reviewer.Review(); // ❌ Expensive, called N times
}

// Good: Review once at end
for (int i = 0; i < maxIters; i++)
{
    await builder.Fix();
}
await reviewer.Review(); // ✅ Called once
```

## Decision Tree: When to Optimize

```
Start here → Are you calling LLMs in production/CI? 
  No → Don't optimize yet
  Yes ↓

Do you know your cost/token metrics?
  No → MEASURE FIRST (implement telemetry)
  Yes ↓

Is cost/latency a problem?
  No → Monitor only
  Yes ↓

Apply in order:
  1. Set budget limits (prevent disasters)
  2. Trim input context (test output, logs)
  3. Multi-tier testing (smoke → full)
  4. Model selection (cheap for loops)
  5. Diff-based responses (biggest win)
```

## Real Example: 80% Reduction

**Before:**
```bash
# Naive orchestrator run
Calls: 8 (plan + build + 3 fix iters × 2)
Tokens: 45,320
- Fix iteration prompts include 12KB test output each
- Builder returns full files (500 lines → 2K tokens each)
- Running full test suite 4 times
```

**After:**
```bash
# Optimized orchestrator run
Calls: 5 (plan + build + 2 smoke + 1 full test + 1 review)
Tokens: 9,250 (80% reduction)
- Trimmed test output (12KB → 1.5KB)
- Smoke test ("dotnet build") during fixes
- Full test once at end
- Budget limit: --maxTokens 20000
```

**Code:**
```bash
dotnet run \
  --smokeTest "dotnet build --no-restore" \
  --onlyFullTestEnd \
  --maxAgentCalls 10 \
  --maxTokens 20000 \
  --builderModel "gpt-3.5-turbo"
```

## Checklist

- [ ] Token usage tracked per call (input/output/phase/agent)
- [ ] Budget limits enforced (`--maxCalls`, `--maxTokens`)
- [ ] Test output trimmed before sending to agents
- [ ] Using smoke tests in fix loops (not full suite)
- [ ] Model selection: cheap for loops, premium for critical
- [ ] Summary report at end showing top token burners
- [ ] OpenTelemetry spans tagged with token metrics
- [ ] (Future) Diff-based responses instead of full files

## References

- OpenAI Token Counting: https://platform.openai.com/tokenizer
- Prompt Engineering: Keep prompts under 2K tokens when possible
- Rule of Thumb: 1 token ≈ 4 characters (English)
- GitHub Copilot: Premium requests scale with token count × model multiplier

## Related Patterns

- **Progressive Enhancement**: Start cheap, escalate if needed
- **Circuit Breaker**: Budget limits = circuit breaker for costs
- **Observability**: Can't optimize what you don't measure
- **Caching**: Don't re-fetch context that hasn't changed

---

**When this was learned:** Feb 2026 - Agent Orchestrator token optimization  
**Root cause:** LLM costs scaling unexpectedly with fix iterations and verbose contexts  
**Impact:** 80% token reduction, budget protection, observable cost patterns  
**Applies to:** Any system with LLM loops (agents, chatbots, code generation)
