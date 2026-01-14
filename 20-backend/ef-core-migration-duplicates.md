# EF Core Migration Duplicate Column Prevention

When scaffolding follow-up EF Core migrations, the migration generator can accidentally re-add a column that was already introduced in a prior migration. This causes "column ... already exists" errors when applying migrations to databases that were previously up-to-date.

## The Problem

`csharp
// Migration 1: AddLastActivityAtToUser.cs
migrationBuilder.AddColumn<DateTime>(
    name: "LastActivityAt",
    table: "Users",
    type: "timestamp with time zone",
    nullable: true);

// Migration 2: AddSystemSettings.cs (later)
migrationBuilder.AddColumn<DateTime>(
    name: "LastActivityAt",  // ⚠️ Re-adding the same column!
    table: "Users",
    type: "timestamp with time zone",
    nullable: true);
`

Result: On database update, ALTER TABLE "Users" ADD "LastActivityAt" runs twice → PostgreSQL error 42701.

## The Solution

After running dotnet ef migrations add ..., review the generated migration file for duplicate add/drop operations. Remove them or regenerate.

1. **Identify** - Check migration for columns added in earlier migrations
2. **Remove** - Delete the duplicate AddColumn/DropColumn blocks
3. **Verify** - Run dotnet test or dotnet ef database update to ensure migration applies cleanly
4. **Commit** - Cleaned migration chains are idempotent across environments

## Quick Checklist

- [ ] After dotnet ef migrations add Name, scan for previously-added columns
- [ ] Delete duplicate add/drop operations
- [ ] Ensure ModelSnapshot reflects final schema state
- [ ] Test with dotnet test before committing (integration tests apply migrations)

## Why This Matters

Duplicate migrations break deployments across dev/staging/prod and prevent fresh database initialization. Clean migration chains ensure idempotency: running all migrations on a fresh database produces the same schema as on an up-to-date database.
