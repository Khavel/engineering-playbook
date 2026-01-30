# In-Memory Rate Limiting with Sliding Window

## Problem
Need to limit requests per IP without external dependencies (Redis). Must be thread-safe and allow easy reset in tests.

## Solution
In-memory dictionary with sliding window using timestamps.

## Implementation

`csharp
public interface IIpRateLimitService
{
    bool IsLimited(string ipAddress);
    void RecordAttempt(string ipAddress);
    void Reset(); // For testing
}

public class IpRateLimitService : IIpRateLimitService
{
    private const int MaxAttemptsPerHour = 10;
    private readonly ILogger<IpRateLimitService> _logger;
    private readonly Dictionary<string, List<DateTime>> _attempts = new();
    private readonly object _lock = new();

    public IpRateLimitService(ILogger<IpRateLimitService> logger)
    {
        _logger = logger;
    }

    public bool IsLimited(string ipAddress)
    {
        lock (_lock)
        {
            if (!_attempts.TryGetValue(ipAddress, out var times))
                return false;

            // Sliding window: only count attempts in last hour
            var cutoffTime = DateTime.UtcNow.AddHours(-1);
            _attempts[ipAddress] = times.Where(t => t > cutoffTime).ToList();

            return _attempts[ipAddress].Count >= MaxAttemptsPerHour;
        }
    }

    public void RecordAttempt(string ipAddress)
    {
        lock (_lock)
        {
            if (!_attempts.ContainsKey(ipAddress))
                _attempts[ipAddress] = new List<DateTime>();

            _attempts[ipAddress].Add(DateTime.UtcNow);
            _logger.LogDebug("Recorded attempt for IP: {Ip}, count: {Count}", 
                ipAddress, _attempts[ipAddress].Count);
        }
    }

    public void Reset()
    {
        lock (_lock)
        {
            _attempts.Clear();
            _logger.LogDebug("Rate limit data cleared");
        }
    }
}
`

## Multiple Limiters (Registration vs Login)

`csharp
public interface IIpRateLimitService
{
    bool IsRegistrationLimited(string ip);
    void RecordRegistrationAttempt(string ip);
    bool IsLoginLimited(string ip);
    void RecordLoginAttempt(string ip);
    void Reset();
}

public class IpRateLimitService : IIpRateLimitService
{
    private const int MaxRegistrationPerHour = 5;  // Stricter
    private const int MaxLoginPerHour = 10;
    
    private readonly Dictionary<string, List<DateTime>> _registrationAttempts = new();
    private readonly Dictionary<string, List<DateTime>> _loginAttempts = new();
    private readonly object _lock = new();

    // Implement separate methods for each limit type...
}
`

## Controller Usage

`csharp
[HttpPost("register")]
public async Task<IActionResult> Register(RegisterRequest request)
{
    var ip = HttpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";

    if (_rateLimitService.IsRegistrationLimited(ip))
    {
        _logger.LogWarning("Rate limited registration from {Ip}", ip);
        return StatusCode(429, new { error = "Too many attempts. Try again later." });
    }

    _rateLimitService.RecordRegistrationAttempt(ip);
    
    // ... process registration
}
`

## Registration (Singleton)

`csharp
// Program.cs - Must be Singleton for shared state across requests
builder.Services.AddSingleton<IIpRateLimitService, IpRateLimitService>();
`

## Test Reset

`csharp
public async Task ResetDatabaseAsync()
{
    // Reset rate limiter to avoid cross-test interference
    var rateLimiter = Services.GetRequiredService<IIpRateLimitService>();
    rateLimiter.Reset();
    
    // ... reset database ...
}
`

## Key Points

- **Singleton lifetime**: Shared across all requests
- **Thread-safe**: Lock around dictionary operations
- **Sliding window**: Auto-expires old attempts on check
- **Testable**: Reset() method for clean test state
- **Separate limits**: Different limits for different operations
- **No external deps**: Works without Redis/distributed cache
