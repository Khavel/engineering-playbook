# Generic Pagination Response Pattern

## Problem
Every paginated endpoint needs the same response structure: items, total count, page info, and navigation helpers.

## Solution
Generic record with computed properties for navigation.

## Implementation

`csharp
namespace YourApi.Shared.Pagination;

public record PaginatedResponse<T>(
    List<T> Items,
    int Total,
    int Page,
    int PageSize
)
{
    public int TotalPages => (int)Math.Ceiling((double)Total / PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}
`

## Service Usage

`csharp
public async Task<PaginatedResponse<FeedbackDto>> GetFeedbackAsync(
    Guid profileId, int page = 1, int pageSize = 20)
{
    var query = _db.Feedbacks
        .Where(f => f.ProfileId == profileId)
        .OrderByDescending(f => f.CreatedAt);

    var total = await query.CountAsync();
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(f => new FeedbackDto(f))
        .ToListAsync();

    return new PaginatedResponse<FeedbackDto>(items, total, page, pageSize);
}
`

## Controller Usage

`csharp
[HttpGet("profile/{id}/feedback")]
public async Task<ActionResult<PaginatedResponse<FeedbackDto>>> GetFeedback(
    Guid id, [FromQuery] int page = 1, [FromQuery] int pageSize = 20)
{
    // Clamp page size to prevent abuse
    pageSize = Math.Clamp(pageSize, 1, 100);
    
    var result = await _feedbackService.GetFeedbackAsync(id, page, pageSize);
    return Ok(result);
}
`

## JSON Output

`json
{
  "items": [...],
  "total": 42,
  "page": 2,
  "pageSize": 10,
  "totalPages": 5,
  "hasNextPage": true,
  "hasPreviousPage": true
}
`

## TypeScript Client Type

`	ypescript
interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
  hasNextPage: boolean;
  hasPreviousPage: boolean;
}
`

## Benefits

- **Single definition**: One generic type for all paginated endpoints
- **Self-documenting**: Navigation properties make intent clear
- **Immutable**: Record type ensures thread safety
- **Consistent API**: Frontend always knows what to expect
