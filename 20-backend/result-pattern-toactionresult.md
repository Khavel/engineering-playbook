# Result Pattern with ToActionResult Extension

## Problem
Converting service results to HTTP responses with proper status codes is repetitive and error-prone.

## Solution
Use Result<T> monad with extension methods that map error codes to HTTP status codes.

## Result Pattern

`csharp
namespace YourApi.Shared.Errors;

public class Result<T>
{
    public T? Value { get; }
    public ErrorResponse? Error { get; }
    public bool IsSuccess => Error == null;
    public bool IsFailure => !IsSuccess;

    private Result(T? value, ErrorResponse? error)
    {
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(ErrorResponse error) => new(default, error);

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<ErrorResponse, TResult> onFailure)
    {
        return IsSuccess ? onSuccess(Value!) : onFailure(Error!);
    }
}
`

## Error Response

`csharp
public record ErrorResponse(
    string Code,
    string Message,
    Dictionary<string, string[]>? Errors = null
)
{
    public static ErrorResponse BadRequest(string msg) => new("bad_request", msg);
    public static ErrorResponse NotFound(string msg = "Resource not found") => new("not_found", msg);
    public static ErrorResponse Conflict(string msg) => new("conflict", msg);
    public static ErrorResponse RateLimit(string msg) => new("rate_limit", msg);
    public static ErrorResponse Forbidden(string msg = "Access denied") => new("forbidden", msg);
    public static ErrorResponse Validation(Dictionary<string, string[]> errors) 
        => new("validation_error", "One or more validation errors occurred", errors);
}
`

## ToActionResult Extension

`csharp
public static class ErrorResponseExtensions
{
    public static IActionResult ToActionResult(this ErrorResponse error)
    {
        var statusCode = error.Code switch
        {
            "bad_request" => StatusCodes.Status400BadRequest,
            "validation_error" => StatusCodes.Status400BadRequest,
            "unauthorized" => StatusCodes.Status401Unauthorized,
            "forbidden" => StatusCodes.Status403Forbidden,
            "not_found" => StatusCodes.Status404NotFound,
            "conflict" => StatusCodes.Status409Conflict,
            "rate_limit" => StatusCodes.Status429TooManyRequests,
            _ => StatusCodes.Status500InternalServerError
        };

        return new ObjectResult(new { error = new { 
            code = error.Code, 
            message = error.Message, 
            errors = error.Errors 
        }}) { StatusCode = statusCode };
    }

    public static IActionResult ToActionResult<T>(this Result<T> result)
    {
        return result.IsSuccess 
            ? new OkObjectResult(result.Value) 
            : result.Error!.ToActionResult();
    }
}
`

## Controller Usage

`csharp
[HttpPost]
public async Task<IActionResult> SubmitFeedback(PostFeedbackRequest request)
{
    var result = await _feedbackService.SubmitFeedbackAsync(userId, request);
    return result.ToActionResult(); // Automatic status code mapping!
}
`

## Why This Works

- **Centralized mapping**: Status codes defined once, used everywhere
- **Type-safe errors**: No magic strings in controllers
- **Composable**: Result.Match() for custom handling when needed
- **Testable**: Services return Result<T>, easy to verify error cases
