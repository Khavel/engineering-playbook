# C# Raw String Literals with Mixed Content (Braces + Interpolation)

## Problem
When using C# raw string literals (`$"""..."""`) with both:
- CSS classes/styles containing braces: `{{ }}`
- Interpolated variables: `{variableName}`

You get **error CS9006**: "El literal de cadena sin formato interpolado no comienza con suficientes caracteres..."

## Root Cause
Raw string literals with a single `$` prefix cannot simultaneously escape both CSS braces and variable interpolations. The parser gets confused about which braces are literal CSS vs. interpolation.

## Solution: Use Verbatim String (`$@"..."`)
For HTML email templates, use **verbatim strings with interpolation** instead of raw strings:

### ❌ FAILS - Raw String with Mixed Content
```csharp
private static string BuildEmailBody(string displayName, string verificationLink)
{
    return $"""
        <style>
            body {{ color: #333; }}
            .button {{ background: #22c55e; }}
        </style>
        <p>Hi {displayName},</p>
        <a href="{verificationLink}">Verify</a>
        """;  // ERROR CS9006
}
```

### ✅ WORKS - Verbatim String with Interpolation
```csharp
private static string BuildEmailBody(string displayName, string verificationLink)
{
    return $@"
        <style>
            body {{ color: #333; }}
            .button {{ background: #22c55e; }}
        </style>
        <p>Hi {displayName},</p>
        <a href=""{verificationLink}"">Verify</a>
        ";
}
```

## Comparison Table
| Feature | Raw `$"""` | Verbatim `$@"` |
|---------|-----------|----------------|
| CSS braces `{{ }}` | Requires `$$"""` or triple braces `{{{` | ✅ Works naturally |
| Variable interpolation `{var}` | ✅ Works naturally | ✅ Works naturally |
| Multiline support | ✅ Clean | ✅ Clean |
| Quotes in content | ✅ Easy (no escaping) | Need `""` for literal quotes |
| **HTML email templates** | ❌ Avoid | ✅ Recommended |

## When to Use Each
- **Use `$@""`** (verbatim): HTML email templates, SQL queries with brackets, any mixed brace content
- **Use `$"""`** (raw): Pure text with many quotes but no curly braces (markdown, JSON payloads)
- **Use `$""`** (regular): Simple strings, no special formatting needs

## Key Takeaway
For email templates with CSS styling, **verbatim strings (`$@""`) are the pragmatic choice**. Raw strings introduce complexity that isn't worth the marginal benefit.

## Reference
- [C# Raw String Literals](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/tutorials/interpolated-strings#raw-string-literals)
- [String Interpolation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/tokens/interpolated-strings)
