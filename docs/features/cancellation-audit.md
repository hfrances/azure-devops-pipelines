# Cancellation Audit

Template: `scripts-templates/audit-cancellation-dotnet.yml`

## What It Checks

Flags public/internal/protected async methods that return:
- `Task`
- `Task<T>`
- `ValueTask`
- `ValueTask<T>`

and do not take `CancellationToken`.

## Exclusions

Ignored paths:
- `obj`
- `bin`
- `examples`
- test paths/names

Allowed exceptions:
- `override` methods
- middleware `Invoke` / `InvokeAsync`
- `BasicAuthenticationHandler.cs`
  - `HandleAuthenticateAsync`
  - `HandleChallengeAsync`

## Params

| Param | Default |
| --- | --- |
| `sourcesDirectory` | `$(Build.SourcesDirectory)` |
| `failOnViolations` | `true` |
| `condition` | `succeeded()` |

## Typical Fix

1. Add `CancellationToken cancellationToken`
2. Pass it downstream where possible
3. If not possible, call `cancellationToken.ThrowIfCancellationRequested()`

Example:

```csharp
public async Task<Result> DoWorkAsync(CancellationToken cancellationToken)
{
    cancellationToken.ThrowIfCancellationRequested();
    return await _service.RunAsync(cancellationToken);
}
```

## Canonical File

Document and use `audit-cancellation-dotnet.yml`.

There is also a duplicate `cancellation-audit-dotnet.yml`, but the standard templates reference `audit-cancellation-dotnet.yml`.
