# ClaimsPrincipal Extension Methods

## Problem
Extracting user ID from JWT claims is repetitive and error-prone (different claim types, null checks, GUID parsing).

## Solution
Extension methods that encapsulate claim extraction with proper null handling.

## Implementation

`csharp
using System.Security.Claims;

namespace YourApi.Shared.Extensions;

public static class ClaimsPrincipalExtensions
{
    /// <summary>
    /// Gets the user ID from claims. Returns null if not found or invalid.
    /// Checks both NameIdentifier and "sub" claims.
    /// </summary>
    public static Guid? GetUserId(this ClaimsPrincipal user)
    {
        var sub = user.FindFirstValue(ClaimTypes.NameIdentifier) 
            ?? user.FindFirstValue("sub");
        
        if (sub != null && Guid.TryParse(sub, out var userId))
        {
            return userId;
        }
        return null;
    }

    /// <summary>
    /// Gets the user ID or throws UnauthorizedAccessException.
    /// Use in [Authorize] endpoints where user MUST exist.
    /// </summary>
    public static Guid GetRequiredUserId(this ClaimsPrincipal user)
    {
        return user.GetUserId() 
            ?? throw new UnauthorizedAccessException("User ID not found in claims");
    }

    /// <summary>
    /// Gets email from claims if present.
    /// </summary>
    public static string? GetEmail(this ClaimsPrincipal user)
    {
        return user.FindFirstValue(ClaimTypes.Email) 
            ?? user.FindFirstValue("email");
    }

    /// <summary>
    /// Checks if user has a specific role.
    /// </summary>
    public static bool HasRole(this ClaimsPrincipal user, string role)
    {
        return user.IsInRole(role);
    }

    /// <summary>
    /// Gets custom claim value by name.
    /// </summary>
    public static string? GetClaimValue(this ClaimsPrincipal user, string claimType)
    {
        return user.FindFirstValue(claimType);
    }
}
`

## Controller Usage

`csharp
[Authorize]
[HttpPost("feedback")]
public async Task<IActionResult> SubmitFeedback(FeedbackRequest request)
{
    // Safe extraction - throws if missing
    var userId = User.GetRequiredUserId();
    
    var result = await _feedbackService.SubmitAsync(userId, request);
    return result.ToActionResult();
}

[Authorize]
[HttpGet("me")]
public async Task<IActionResult> GetProfile()
{
    // Optional extraction - handles gracefully
    var userId = User.GetUserId();
    if (userId == null)
        return Unauthorized();
    
    var profile = await _profileService.GetAsync(userId.Value);
    return Ok(profile);
}
`

## Before (Repetitive)

`csharp
// ❌ This pattern repeated everywhere
var sub = User.FindFirstValue(ClaimTypes.NameIdentifier);
if (sub == null || !Guid.TryParse(sub, out var userId))
{
    return Unauthorized();
}
`

## After (Clean)

`csharp
// ✅ One line
var userId = User.GetRequiredUserId();
`

## Benefits

- **DRY**: Claim extraction in one place
- **Safe**: Null checks and parsing handled
- **Flexible**: Both optional and required variants
- **Testable**: Easy to mock ClaimsPrincipal with known claims
