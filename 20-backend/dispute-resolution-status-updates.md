# Dispute Resolution: Side-Effects vs Query-Time Filtering in .NET APIs

When resolving disputes or moderation events, you often must decide between:

- Writing side-effects to related entities immediately (e.g., mark feedback as Removed when dispute is Upheld)
- Or keeping data immutable and applying filtering at read-time (e.g., hide disputed feedback via queries only)

This note outlines a pragmatic pattern for .NET + EF Core when the decision is to reflect moderation outcomes on related entities.

## Pattern: Update Related Entity on Resolution

- Keep dispute as source of truth (status + resolution) for auditing.
- Apply targeted side-effects to related entities for simpler reads and consistent UI behavior.
- Log a `ModerationEvent` for traceability.

### Controller example (ASP.NET Core)

```csharp
[HttpPost("disputes/{disputeId}/resolve")]
public async Task<IActionResult> ResolveDispute(Guid disputeId, ResolveDto dto)
{
    var actor = GetActorId();
    var d = await _db.Disputes.FindAsync(disputeId);
    if (d == null) return NotFound();

    d.Status = DisputeStatus.Resolved;
    if (Enum.TryParse<DisputeResolution>(dto.Resolution, true, out var resolution))
        d.Resolution = resolution;

    d.ResolvedAt = DateTime.UtcNow;

    // Side-effect: reflect resolution on related feedback
    if (d.FeedbackId != Guid.Empty)
    {
        var fb = await _db.EncounterFeedbacks.FindAsync(d.FeedbackId);
        if (fb != null && d.Resolution.HasValue)
        {
            if (d.Resolution.Value == DisputeResolution.Upheld)
                fb.Status = FeedbackStatus.Removed;   // hide content
            else if (d.Resolution.Value == DisputeResolution.Rejected)
                fb.Status = FeedbackStatus.Active;    // restore content
            // Partial → either no-op or custom policy (e.g., soft demotion)
        }
    }

    _db.ModerationEvents.Add(new ModerationEvent
    {
        EventId = Guid.NewGuid(),
        EntityType = ModerationEntity.Dispute,
        EntityId = disputeId,
        Action = ModerationAction.ResolveDispute,
        ActorUserId = actor,
        ReasonCode = dto.ReasonCode ?? "dispute_resolve",
        Notes = dto.Notes,
        CreatedAt = DateTime.UtcNow
    });

    await _db.SaveChangesAsync();
    return Ok();
}
```

### Why this works well

- Simplifies queries and UI logic (no special-case filtering everywhere).
- Aligns with typical moderation expectations (removed content stays removed).
- Keeps an auditable trail via `Dispute` + `ModerationEvent`.

## Alternatives: Read-Time Filtering Only

- Never mutate related content; instead, exclude/hide at read-time based on dispute status.
- Pros: immutable content; clear audit separation.
- Cons: complex queries across endpoints; higher risk of inconsistent UI behavior.

Use this when regulatory or audit constraints disallow content mutation.

## Implementation Tips

- Enums: make `Enum.TryParse(..., ignoreCase: true, ...)` tolerant to DTO casing.
- Transactions: for multi-entity updates, prefer a single `SaveChangesAsync()` or an explicit transaction if multiple units of work are involved.
- Events: always write a `ModerationEvent` so you can reconstruct timelines.
- Idempotency: a second resolve call should be harmless (no extra side effects).
- Partial outcomes: define a clear policy (e.g., demote visibility or tag).

## Testing Strategy (xUnit + Moq)

- Arrange: seed a `Dispute` (Open) and a related `EncounterFeedback` (Disputed).
- Act: call `ResolveDispute` with `Resolution = "Upheld"`.
- Assert: dispute is `Resolved` + `Resolution = Upheld`; feedback `Status = Removed`.

```csharp
[Fact]
public async Task ResolveDispute_Upheld_RemovesFeedback()
{
    // Arrange (mock DbSets with in-memory lists)
    var dispute = new Dispute { DisputeId = id, FeedbackId = fbId, Status = DisputeStatus.Open };
    var feedback = new EncounterFeedback { FeedbackId = fbId, Status = FeedbackStatus.Disputed };
    // ...set up mocked context to return dispute/feedback via FindAsync...

    // Act
    var result = await controller.ResolveDispute(id, new ResolveDto("Upheld", "reviewed_valid", "notes"));

    // Assert
    result.Should().BeOfType<OkResult>();
    dispute.Status.Should().Be(DisputeStatus.Resolved);
    dispute.Resolution.Should().Be(DisputeResolution.Upheld);
    feedback.Status.Should().Be(FeedbackStatus.Removed);
}
```

## When to prefer read-time filtering

- Legal/Compliance forbids content mutation
- Historical replays must reflect original states only
- Multi-tenant systems where tenants apply different viewing policies

In these cases, centralize filtering in query helpers and enforce consistently across endpoints, plus add integration tests guarding for regressions.
