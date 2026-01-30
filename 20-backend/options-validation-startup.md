# Startup Configuration Validation with IValidateOptions

## Problem
Invalid configuration (missing secrets, malformed URLs) should fail fast in production but allow development to work without all config.

## Solution
Use IValidateOptions<T> for environment-aware validation that runs at startup.

## Options Class

`csharp
public class OAuthOptions
{
    public const string SectionName = "OAuth";

    public SteamOptions Steam { get; set; } = new();
    public DiscordOptions Discord { get; set; } = new();
    public string FrontendBaseUrl { get; set; } = string.Empty;

    public bool HasAnyProviderConfigured => 
        Steam.IsConfigured || Discord.IsConfigured;
}

public class DiscordOptions
{
    public string ClientId { get; set; } = string.Empty;
    public string ClientSecret { get; set; } = string.Empty;
    public bool IsConfigured => !string.IsNullOrWhiteSpace(ClientId);
}
`

## Options Validator

`csharp
public class OAuthOptionsValidator : IValidateOptions<OAuthOptions>
{
    private readonly IHostEnvironment _environment;

    public OAuthOptionsValidator(IHostEnvironment environment)
    {
        _environment = environment;
    }

    public ValidateOptionsResult Validate(string? name, OAuthOptions options)
    {
        var errors = new List<string>();

        // Require at least one provider in production
        if (!_environment.IsDevelopment() && !options.HasAnyProviderConfigured)
        {
            errors.Add("At least one OAuth provider must be configured in production.");
        }

        // Validate required URLs in production
        if (!_environment.IsDevelopment() && string.IsNullOrWhiteSpace(options.FrontendBaseUrl))
        {
            errors.Add("FrontendBaseUrl must be configured.");
        }

        // Validate paired credentials (if one is set, both must be)
        if (!string.IsNullOrWhiteSpace(options.Discord.ClientId) && 
            string.IsNullOrWhiteSpace(options.Discord.ClientSecret))
        {
            errors.Add("Discord ClientSecret required when ClientId is provided.");
        }

        return errors.Count > 0 
            ? ValidateOptionsResult.Fail(errors) 
            : ValidateOptionsResult.Success;
    }
}
`

## Registration

`csharp
public static class OAuthConfigurationExtensions
{
    public static IServiceCollection AddOAuthConfiguration(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.Configure<OAuthOptions>(
            configuration.GetSection(OAuthOptions.SectionName));
        
        services.AddSingleton<IValidateOptions<OAuthOptions>, OAuthOptionsValidator>();
        
        return services;
    }

    public static WebApplication ValidateOAuthConfiguration(this WebApplication app)
    {
        var options = app.Services.GetRequiredService<IOptions<OAuthOptions>>();
        var logger = app.Services.GetRequiredService<ILogger<OAuthOptions>>();

        try
        {
            // Accessing .Value triggers validation
            var opts = options.Value;
            logger.LogInformation("OAuth: Steam={Steam}, Discord={Discord}", 
                opts.Steam.IsConfigured, opts.Discord.IsConfigured);
        }
        catch (OptionsValidationException ex)
        {
            logger.LogCritical("Config validation failed: {Errors}", 
                string.Join("; ", ex.Failures));

            if (!app.Environment.IsDevelopment())
            {
                throw new InvalidOperationException(
                    $"Configuration invalid: {string.Join("; ", ex.Failures)}", ex);
            }
        }

        return app;
    }
}
`

## Usage in Program.cs

`csharp
builder.Services.AddOAuthConfiguration(builder.Configuration);

// ... build app ...

app.ValidateOAuthConfiguration(); // Fails fast if invalid
app.Run();
`

## Benefits

- **Fail-fast in production**: Bad config = startup failure
- **Graceful dev mode**: Warnings only, allows partial config
- **Type-safe**: Strongly typed options vs raw config strings
- **Testable**: Validators can be unit tested
- **Clear errors**: Specific messages for each validation failure
