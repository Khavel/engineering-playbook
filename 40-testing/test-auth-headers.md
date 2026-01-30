# Test Authentication via Request Headers

## Problem
Integration tests need to authenticate as different users/roles without real JWT tokens or complex auth setup.

## Solution
Custom test auth handler that reads user identity from request headers.

## Test Auth Handler

`csharp
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public const string AuthenticationScheme = "TestScheme";

    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder)
        : base(options, logger, encoder) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        // Read test identity from headers
        var testUserId = Context.Request.Headers["X-Test-UserId"].ToString();
        var testRole = Context.Request.Headers["X-Test-Role"].ToString();

        if (string.IsNullOrEmpty(testUserId))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, testUserId),
            new Claim("sub", testUserId)
        };

        if (!string.IsNullOrEmpty(testRole))
        {
            claims.Add(new Claim(ClaimTypes.Role, testRole));
        }

        var identity = new ClaimsIdentity(claims, AuthenticationScheme);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, AuthenticationScheme);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
`

## WebAppFactory Setup

`csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureTestServices(services =>
    {
        services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = TestAuthHandler.AuthenticationScheme;
            options.DefaultChallengeScheme = TestAuthHandler.AuthenticationScheme;
        })
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
            TestAuthHandler.AuthenticationScheme, options => { });
    });
}
`

## HttpClient Extensions

`csharp
public static class HttpClientExtensions
{
    public static void SetTestUser(this HttpClient client, Guid userId, string? role = null)
    {
        // Clear any existing auth headers
        client.DefaultRequestHeaders.Remove("X-Test-UserId");
        client.DefaultRequestHeaders.Remove("X-Test-Role");
        
        client.DefaultRequestHeaders.Add("X-Test-UserId", userId.ToString());
        if (!string.IsNullOrEmpty(role))
        {
            client.DefaultRequestHeaders.Add("X-Test-Role", role);
        }
    }

    public static void SetModeratorUser(this HttpClient client, Guid userId)
    {
        client.SetTestUser(userId, "moderator");
    }

    public static void SetSuperadminUser(this HttpClient client, Guid userId)
    {
        client.SetTestUser(userId, "superadmin");
    }

    public static void ClearAuth(this HttpClient client)
    {
        client.DefaultRequestHeaders.Remove("X-Test-UserId");
        client.DefaultRequestHeaders.Remove("X-Test-Role");
    }
}
`

## Test Usage

`csharp
[Fact]
public async Task AdminEndpoint_WithModeratorRole_Succeeds()
{
    // Arrange
    var userId = Guid.NewGuid();
    _client.SetModeratorUser(userId);

    // Act
    var response = await _client.GetAsync("/api/admin/queue");

    // Assert
    response.StatusCode.Should().NotBe(HttpStatusCode.Unauthorized);
}

[Fact]
public async Task AdminEndpoint_WithoutAuth_Returns401()
{
    // Arrange - Don't set auth headers
    _client.ClearAuth();

    // Act
    var response = await _client.GetAsync("/api/admin/queue");

    // Assert
    response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
}
`

## Why This Approach

- **Fast**: No actual JWT signing/verification
- **Flexible**: Change user/role per request
- **Realistic**: Goes through real auth middleware
- **Isolated**: Each test controls its own identity
- **Simple**: No complex test doubles needed
