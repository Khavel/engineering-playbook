# Business Logic & Service Layer Patterns

## Service Pattern

Encapsulate complex business logic in services:

```csharp
public class ReputationRecomputeService
{
    private readonly ApplicationDbContext _db;
    private const double DecayScaleDays = 180.0;
    private const double DisputeMultiplier = 0.25;

    public ReputationRecomputeService(ApplicationDbContext db)
    {
        _db = db;
    }

    public async Task RecomputeAsync(Guid profileId, CancellationToken ct = default)
    {
        // Load all relevant feedback
        var feedbacks = await _db.EncounterFeedbacks
            .Where(f => f.TargetProfileId == profileId && f.Status != FeedbackStatus.Removed)
            .ToListAsync(ct);

        // Complex business logic
        double score = CalculateReputation(feedbacks);
        string bucket = DetermineBucket(score, feedbacks.Count);

        // Persist results
        var profile = await _db.PlayerProfiles.FirstOrDefaultAsync(p => p.ProfileId == profileId, ct);
        if (profile == null) return;

        profile.RepScore = (int)score;
        profile.RepBucket = bucket;
        profile.UpdatedAt = DateTime.UtcNow;

        await _db.SaveChangesAsync(ct);
    }

    private double CalculateReputation(List<EncounterFeedback> feedbacks)
    {
        // Complex algorithm: weighted decay, dispute factors, etc.
        double weightedSum = 0.0;
        double totalWeight = 0.0;
        
        var now = DateTime.UtcNow;
        
        foreach (var fb in feedbacks)
        {
            var value = fb.Outcome switch
            {
                FeedbackOutcome.Positive => 1.0,
                FeedbackOutcome.Neutral => 0.0,
                FeedbackOutcome.Caution => -1.0,
                _ => 0.0
            };

            var ageDays = (now - fb.CreatedAt).TotalDays;
            var decay = Math.Exp(-ageDays / DecayScaleDays);
            var mult = fb.Status == FeedbackStatus.Disputed ? DisputeMultiplier : 1.0;
            var weight = (double)fb.WeightApplied * decay * mult;
            
            weightedSum += weight * value;
            totalWeight += weight;
        }

        // Bayesian prior for scores with insufficient data
        const double priorK = 8.0;
        const double priorMean = 0.5;
        
        double p = 0.5;
        if (totalWeight > 0.0)
        {
            var mean = weightedSum / totalWeight;
            p = (mean + 1.0) / 2.0; // map [-1,1] to [0,1]
        }

        var finalScore = (priorK * priorMean + totalWeight * p) / (priorK + totalWeight);
        return Math.Round(100.0 * finalScore);
    }

    private string DetermineBucket(double score, int count)
    {
        if (count < 3) return "insufficient-data";
        if (score <= 34) return "mostly-caution";
        if (score <= 64) return "mixed";
        return "mostly-positive";
    }
}
```

## Service Registration

Register services in `Program.cs`:

```csharp
builder.Services.AddScoped<ReputationRecomputeService>();
builder.Services.AddScoped<AuthenticationService>();
builder.Services.AddScoped<ValidationService>();
```

## Dependency Injection in Controllers

```csharp
[ApiController]
[Route("api/[controller]")]
public class FeedbackController : ControllerBase
{
    private readonly ApplicationDbContext _db;
    private readonly ReputationRecomputeService _repService;

    public FeedbackController(ApplicationDbContext db, ReputationRecomputeService repService)
    {
        _db = db;
        _repService = repService;
    }

    [HttpPost("reply")]
    public async Task<ActionResult> Reply(int feedbackId, ReplyRequest request)
    {
        // Validation
        if (string.IsNullOrWhiteSpace(request.Text))
            return BadRequest("Reply cannot be empty");

        // Business logic
        var feedback = await _db.EncounterFeedbacks.FindAsync(feedbackId);
        if (feedback == null)
            return NotFound();

        // Complex operation
        feedback.Status = FeedbackStatus.Replied;
        await _db.SaveChangesAsync();
        await _repService.RecomputeAsync(feedback.TargetProfileId);

        return Ok();
    }
}
```

## Key Principles

1. **Single Responsibility** - Each service does one thing well
2. **Dependency Injection** - No hardcoded dependencies
3. **Async/Await** - Always use async for I/O operations
4. **Testability** - Services are easy to test with mocks
5. **Separation of Concerns** - Business logic separate from HTTP concerns
