# .NET Backend Architecture & Setup

## Project Structure

```
src/ArcRaiders.Trust.Api/
├── Controllers/          # REST API endpoints
├── Data/                 # DbContext, database config, seed data
├── Models/               # DTOs, request/response models, entities
├── Services/             # Business logic services
├── Middleware/           # Custom middleware (error handling, auth)
└── Migrations/           # EF Core database migrations
```

## Program Setup & Configuration

### Dependency Injection & Services
Register services in `Program.cs` following this pattern:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Controllers with JSON config
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    });

// Entity Framework
var conn = Environment.GetEnvironmentVariable("DATABASE_URL") 
    ?? builder.Configuration.GetConnectionString("DefaultConnection") 
    ?? "Host=localhost;Database=arc_raiders;Username=postgres;Password=postgres";
    
builder.Services.AddDbContext<ApplicationDbContext>(opt =>
    opt.UseNpgsql(conn));

// JWT Authentication
var jwtKey = Environment.GetEnvironmentVariable("JWT_KEY") 
    ?? builder.Configuration["Jwt:Key"] 
    ?? "dev-key-change-me";
    
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.RequireHttpsMetadata = false;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtKey))
        };
    });

// Business services
builder.Services.AddScoped<ReputationRecomputeService>();

// Swagger/OpenAPI
builder.Services.AddSwaggerGen(c =>
{
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer",
        In = ParameterLocation.Header
    });
});
```

### Middleware Setup
Register middleware in order:

```csharp
var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Custom exception middleware (must be early)
app.UseMiddleware<ApiExceptionMiddleware>();

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## Database & Entity Framework

### DbContext Pattern
```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<PlayerProfile> PlayerProfiles { get; set; }
    public DbSet<EncounterFeedback> EncounterFeedbacks { get; set; }
    public DbSet<Report> Reports { get; set; }
    public DbSet<Dispute> Disputes { get; set; }
    
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) 
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        // Configure entities, indexes, relationships here
    }
}
```

### Model Entity Pattern
```csharp
public class EncounterFeedback
{
    public int Id { get; set; }
    public Guid SubmitterId { get; set; }
    public Guid TargetProfileId { get; set; }
    
    public FeedbackOutcome Outcome { get; set; }
    public FeedbackStatus Status { get; set; }
    
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    
    // Navigation properties
    public PlayerProfile? TargetProfile { get; set; }
    public User? Submitter { get; set; }
}
```

## Error Handling Middleware

Centralize exception handling with custom middleware:

```csharp
public class ApiExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ApiExceptionMiddleware> _log;

    public ApiExceptionMiddleware(RequestDelegate next, ILogger<ApiExceptionMiddleware> log)
    {
        _next = next;
        _log = log;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "Unhandled exception");
            context.Response.ContentType = "application/json";
            context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
            
            var payload = new 
            { 
                error = new 
                { 
                    code = "internal_error", 
                    message = "An unexpected error occurred.",
                    details = ex.Message
                } 
            };
            
            await context.Response.WriteAsJsonAsync(payload);
        }
    }
}
```

## Environment Configuration

Use environment variables for sensitive data and deployment:

```bash
# Development (appsettings.Development.json)
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=arc_raiders;..."
  },
  "Jwt": {
    "Key": "dev-secret-key"
  }
}

# Production (via environment variables on Fly.io, etc.)
DATABASE_URL=postgresql://...
JWT_KEY=your-secret-key
```

## Health Check & Liveness
Include basic health checks for deployment:

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>();

app.MapHealthChecks("/health");
```
