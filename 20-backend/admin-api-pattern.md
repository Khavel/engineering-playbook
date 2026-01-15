## Admin API Design Pattern

**Context**: Building admin endpoints that list entities with filtering, search, and aggregated stats (like user management, moderation queues, etc.).

### Pattern Overview

Admin APIs typically need:
1. **Role-based authorization** - Only moderators/admins can access
2. **Pagination** - Handle large datasets
3. **Search & Filtering** - Find specific records quickly
4. **Aggregated data** - Count related records, dates, etc.
5. **DTOs matching frontend** - Consistent data format

### Implementation Template

```csharp
[ApiController]
[Authorize(Roles = "moderator,admin")]
[Route("api/admin")]
public class AdminController : ControllerBase
{
    [HttpGet("users")]
    public async Task<IActionResult> GetUsers(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        [FromQuery] string? search = null,
        [FromQuery] string? status = null)
    {
        // 1. Build base query with filters
        var query = _db.Users.AsQueryable();

        if (!string.IsNullOrWhiteSpace(search))
            query = query.Where(u => u.Email.Contains(search));

        if (!string.IsNullOrWhiteSpace(status))
            query = status.ToLower() switch {
                "active" => query.Where(u => u.TrustLevel >= 1),
                "banned" => query.Where(u => u.TrustLevel == -1),
                _ => query
            };

        // 2. Get total before pagination (for pagination info)
        var total = await query.CountAsync();

        // 3. Apply pagination
        var users = await query
            .OrderByDescending(u => u.CreatedAt)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();

        // 4. Batch fetch aggregates (avoid N+1!)
        var userIds = users.Select(u => u.UserId).ToList();
        var feedbackCounts = await _db.EncounterFeedbacks
            .Where(f => userIds.Contains(f.ReviewerUserId))
            .GroupBy(f => f.ReviewerUserId)
            .Select(g => new { UserId = g.Key, Count = g.Count() })
            .ToDictionaryAsync(x => x.UserId, x => x.Count);

        // 5. Build response DTOs
        var dtos = users.Select(u => new UserDto(
            UserId: u.UserId.ToString(),
            Email: u.Email,
            FeedbackCount: feedbackCounts.GetValueOrDefault(u.UserId, 0)
        )).ToList();

        return Ok(new PaginatedResponseDto<UserDto>(
            Items: dtos,
            Total: total,
            Page: page,
            PageSize: pageSize
        ));
    }
}
```

### API Contract Example

```http
GET /api/admin/users?page=1&pageSize=20&search=email&status=active
Authorization: Bearer {moderator-token}

200 OK
{
  "items": [
    {
      "userId": "guid",
      "email": "user@example.com",
      "status": "active",
      "feedbackCount": 45,
      "createdAt": "2024-01-15T00:00:00Z"
    }
  ],
  "total": 100,
  "page": 1,
  "pageSize": 20
}
```

### Key Principles

1. **Separate Concerns**: Filter logic, pagination, aggregation in clear steps
2. **Efficient Queries**: Batch load related data to avoid N+1 problems
3. **Frontend-First DTOs**: Shape response around what frontend needs
4. **Flexible Filtering**: Search + status filtering work independently
5. **Clear Pagination**: Always return total count and current page info

### When Adapting
- Change the base entity (`_db.Users` → whatever you're listing)
- Adjust filter cases based on entity status field
- Add/remove aggregates based on frontend needs
- Use same pagination pattern for consistency

**Related Issue**: Arc Raiders #21 (Admin Users List), #5 (Admin Dashboard)
**Proven in**: ArcRaiders Trust Platform
