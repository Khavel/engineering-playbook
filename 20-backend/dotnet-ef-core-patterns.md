# .NET Entity Framework Core Patterns

## Eager Loading to Prevent N+1 Queries

### The Problem
When loading related entities one-by-one in a loop, you execute one initial query plus N additional queries (one per item). This is a critical performance issue in large datasets.

**Bad Example:**
```csharp
var reports = await _db.Reports.Take(50).ToListAsync();
foreach (var report in reports)
{
    var feedback = await _db.EncounterFeedback.FindAsync(report.FeedbackId); // N queries
    var profile = await _db.PlayerProfile.FindAsync(feedback.TargetProfileId); // N queries
}
```
Result: 1 + 50 + 50 = 101 queries for 50 items.

### The Solution
Use `.Include()` and `.ThenInclude()` to eagerly load related entities in a single SQL JOIN query.

**Good Example:**
```csharp
var reports = await _db.Reports
    .Include(r => r.Feedback)
    .ThenInclude(f => f!.TargetProfile)
    .Include(r => r.Reporter)
    .OrderByDescending(r => r.CreatedAt)
    .Take(50)
    .ToListAsync();
```
Result: 1 optimized SQL query with INNER JOINs (~150x improvement).

### Prerequisites
Navigation properties must be defined on the model:
```csharp
public class Report
{
    public int Id { get; set; }
    public int FeedbackId { get; set; }
    
    // Navigation properties for eager loading
    public EncounterFeedback? Feedback { get; set; }
    public User? Reporter { get; set; }
}
```

## Authorization Patterns

### Role-Based Access Control
Use `[Authorize]` attributes with role checks:
```csharp
[Authorize(Roles = "Moderator,SuperAdmin")]
[HttpPost("api/admin/feedback/{feedbackId}/resolve")]
public async Task<ActionResult> ResolveFeedback(int feedbackId)
```

### Ownership Verification
Always verify the requesting user owns the resource before allowing actions:
```csharp
var feedback = await _db.EncounterFeedback.FindAsync(feedbackId);
var targetProfile = await _db.PlayerProfile.FindAsync(feedback.TargetProfileId);

if (targetProfile.UserId != userId)
{
    return Forbid(); // 403 Forbidden
}
```

## Data Validation Patterns

### Input Validation
Validate inputs server-side before processing:
```csharp
if (string.IsNullOrWhiteSpace(request.ReplyText))
{
    return BadRequest("Reply text cannot be empty");
}

if (request.ReplyText.Length > 240)
{
    return BadRequest("Reply text cannot exceed 240 characters");
}
```

### Duplicate Prevention
Check for duplicates and return 409 Conflict:
```csharp
var existingDispute = await _db.Disputes
    .Where(d => d.FeedbackId == feedbackId && d.InitiatorId == userId)
    .FirstOrDefaultAsync();

if (existingDispute != null)
{
    return Conflict("You have already disputed this feedback");
}
```

### Rate Limiting
Implement rate limits to prevent abuse:
```csharp
var reportsLastHour = await _db.Reports
    .Where(r => r.ReporterId == userId && r.CreatedAt > DateTime.UtcNow.AddHours(-1))
    .CountAsync();

if (reportsLastHour >= 10)
{
    return StatusCode(429, "Too many reports. Max 10 per hour.");
}
```

## HTTP Status Codes

| Code | Usage |
|------|-------|
| 200 | Success with response body |
| 201 | Created resource |
| 400 | Bad request (validation, format) |
| 403 | Forbidden (authorization failure) |
| 404 | Not found |
| 409 | Conflict (duplicate, state conflict) |
| 429 | Too many requests (rate limit) |
| 500 | Server error |

## DTO Usage

Always return DTOs, never expose EF entities directly:
```csharp
[HttpGet("api/admin/moderation/queue")]
public async Task<ActionResult<ModerationQueueDto>> GetModerationQueue()
{
    var reports = await _db.Reports
        .Include(r => r.Feedback).ThenInclude(f => f!.TargetProfile)
        .Include(r => r.Reporter)
        .ToListAsync();
    
    return Ok(new ModerationQueueDto
    {
        Reports = reports.Select(r => new ReportDto
        {
            Id = r.Id,
            Reason = r.Reason,
            ReporterName = r.Reporter!.Gamertag,
            // ... other properties
        }).ToList()
    });
}
```
