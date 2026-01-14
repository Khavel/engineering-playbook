# .NET Integration Testing with Testcontainers

## Test Infrastructure Setup

Use Testcontainers for isolated, reproducible database testing:

```csharp
using Testcontainers.PostgreSql;

public class IntegrationTestWebAppFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer = 
        new PostgreSqlBuilder("postgres:15-alpine")
            .WithDatabase("arcraiders_test")
            .WithUsername("testuser")
            .WithPassword("testpass")
            .Build();

    public string ConnectionString => _dbContainer.GetConnectionString();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real DbContext
            services.RemoveAll<DbContextOptions<ApplicationDbContext>>();
            services.RemoveAll<ApplicationDbContext>();

            // Add test DbContext with isolated container
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseNpgsql(ConnectionString);
            });

            // Apply migrations to test database
            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            db.Database.Migrate();
        });
        
        // Replace authentication with test scheme
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

    public async Task InitializeAsync() => await _dbContainer.StartAsync();
    public async Task DisposeAsync() => await _dbContainer.StopAsync();
}
```

## Integration Test Pattern

```csharp
public class AdminControllerTests : IClassFixture<IntegrationTestWebAppFactory>, IAsyncLifetime
{
    private readonly IntegrationTestWebAppFactory _factory;
    private readonly HttpClient _client;

    public AdminControllerTests(IntegrationTestWebAppFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }
    
    public async Task InitializeAsync()
    {
        // Reset database for clean test state
        await _factory.ResetDatabaseAsync();
    }
    
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task ModerationQueue_Reports_ReturnsReportsList()
    {
        // Arrange
        var moderatorId = Guid.NewGuid();
        await SeedTestData();
        _client.SetModeratorUser(moderatorId);

        // Act
        var response = await _client.GetAsync("/api/admin/moderation/queue?type=reports");

        // Assert
        response.EnsureSuccessStatusCode();
        var reports = await response.Content.ReadFromJsonAsync<List<Report>>();
        reports.Should().NotBeNull();
        reports.Should().HaveCountGreaterThan(0);
    }
}
```

## Test Authentication

Create a test authentication handler for isolated testing:

```csharp
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public const string AuthenticationScheme = "TestScheme";
    
    private ClaimsPrincipal _currentPrincipal = new();

    public TestAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder) 
        : base(options, logger, encoder) { }

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        return Task.FromResult(AuthenticateResult.Success(
            new AuthenticationTicket(_currentPrincipal, AuthenticationScheme)));
    }
    
    public void SetPrincipal(ClaimsPrincipal principal) 
        => _currentPrincipal = principal;
}

// Extension method for convenience
public static class HttpClientExtensions
{
    public static void SetModeratorUser(this HttpClient client, Guid userId)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId.ToString()),
            new Claim(ClaimTypes.Role, "Moderator")
        };
        var identity = new ClaimsIdentity(claims, "test");
        var principal = new ClaimsPrincipal(identity);
        // Set principal in handler...
    }
}
```

## Testing Best Practices

- **Use Testcontainers** for real database testing, not mocks
- **IAsyncLifetime** for proper async setup/teardown
- **IClassFixture** to share factory across tests
- **ResetDatabaseAsync()** before each test for isolation
- **FluentAssertions** for readable assertions
- **Arrange-Act-Assert** pattern for clarity
- Test observable behavior, not implementation details
- Use meaningful test names that describe the scenario

## Common Test Patterns

### Testing Authorization
```csharp
[Fact]
public async Task DeleteReport_Unauthorized_ReturnsForbidden()
{
    // Arrange - wrong user
    var reportId = await CreateTestReport();
    _client.SetUser(Guid.NewGuid()); // Different user

    // Act
    var response = await _client.DeleteAsync($"/api/reports/{reportId}");

    // Assert
    response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
}
```

### Testing Data Persistence
```csharp
[Fact]
public async Task CreateReport_PersistsToDatabase()
{
    // Arrange
    var request = new CreateReportRequest { Reason = "Test" };

    // Act
    var response = await _client.PostAsJsonAsync("/api/reports", request);

    // Assert
    response.EnsureSuccessStatusCode();
    
    // Verify in database
    using var scope = _factory.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    var report = await db.Reports.FirstOrDefaultAsync(r => r.Reason == "Test");
    report.Should().NotBeNull();
}
```
