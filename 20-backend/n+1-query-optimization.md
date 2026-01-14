## N+1 Query Optimization: Batch Query Pattern

**Problem**: When fetching a paginated list of entities that need aggregated related data (counts, dates, etc.), you can end up with N+1 queries:
1 query to fetch the list, then 1 query per item for each aggregate.

**Solution**: Use `GroupBy` + `ToDictionary` to batch queries and fetch all aggregates in parallel with the main query.

### Code Example (EF Core)

```csharp
// ❌ BAD: N+1 queries (1 users query + N feedback count queries)
var users = await _db.Users.Skip(offset).Take(pageSize).ToListAsync();
foreach (var user in users)
{
    var feedbackCount = await _db.EncounterFeedbacks
        .CountAsync(f => f.ReviewerUserId == user.UserId); // N queries!
    // ...
}

// ✅ GOOD: Batch query pattern (1 + 3 queries only)
var users = await _db.Users
    .OrderByDescending(u => u.CreatedAt)
    .Skip(offset).Take(pageSize)
    .ToListAsync();

var userIds = users.Select(u => u.UserId).ToList();

// Batch queries: Get all counts in 3 queries
var feedbackCounts = await _db.EncounterFeedbacks
    .Where(f => userIds.Contains(f.ReviewerUserId))
    .GroupBy(f => f.ReviewerUserId)
    .Select(g => new { UserId = g.Key, Count = g.Count() })
    .ToDictionaryAsync(x => x.UserId, x => x.Count);

var profileCounts = await _db.PlayerProfiles
    .Where(p => p.ClaimedByUserId != null && userIds.Contains(p.ClaimedByUserId.Value))
    .GroupBy(p => p.ClaimedByUserId)
    .Select(g => new { UserId = g.Key, Count = g.Count() })
    .ToDictionaryAsync(x => x.UserId!.Value, x => x.Count);

// Build DTOs using cached dictionaries (no more DB calls)
var dtos = users.Select(u => new UserDto(
    UserId: u.UserId.ToString(),
    FeedbackCount: feedbackCounts.GetValueOrDefault(u.UserId, 0),
    ProfilesCount: profileCounts.GetValueOrDefault(u.UserId, 0)
)).ToList();
```

### When to Use
- Paginated APIs that return aggregated data
- Admin dashboards listing users with stats
- Any scenario where you need counts/dates for related entities
- When N can be large (50+ items per page)

### Performance Impact
- **Before**: 1 + N queries (e.g., 51 queries for 50 users)
- **After**: 1 + 3 queries (constant, scales to any page size)

### Key Patterns
1. Fetch primary entities and collect their IDs
2. Use `GroupBy` + aggregation (`Count()`, `Max()`, etc.)
3. Convert to `Dictionary` keyed by the entity ID
4. Use `GetValueOrDefault()` when building DTOs to handle missing keys

**Related Issue**: Arc Raiders #21 (Admin Users List API)
