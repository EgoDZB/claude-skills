---
name: async-validation-net11
description: >
  Guidance on asynchronous validation introduced in .NET 11: AsyncValidationAttribute,
  IAsyncValidatableObject and Validator.ValidateObjectAsync (System.ComponentModel.DataAnnotations),
  async validation of Minimal API request bodies via AddValidation(), and async Options validation
  (IAsyncValidateOptions<T>, OptionsBuilder.ValidateAsync, ValidateOnStart) including the RC 1
  breaking change where synchronous access to async-only-validated options throws.
  USE FOR: validators that need I/O (database uniqueness checks, HTTP/health probes, external
  lookups), validating options against external resources at startup, OptionsValidationException
  after upgrading to .NET 11 RC 1, Blazor async form validation (EditContext.ValidateAsync),
  replacing FluentValidation's MustAsync, any project targeting net11.0 that mentions async validation.
  DO NOT USE FOR: projects targeting .NET 10 or earlier, purely synchronous DataAnnotations or
  IValidateOptions usage, FluentValidation-specific APIs.
---

# Asynchronous Validation — .NET 11

.NET 11 adds async validation in two independent places:

1. **DataAnnotations / Minimal APIs** — models validated per request (Preview 6).
2. **Options pattern** — configuration validated at host startup (Preview 6, reshaped in RC 1).

Pick the section that matches the user's scenario.

## Target framework

```xml
<TargetFramework>net11.0</TargetFramework>
```

---

## 1. DataAnnotations and Minimal APIs

### Async attribute

Derive from `AsyncValidationAttribute` and override `IsValidAsync`. The synchronous `IsValid` is still abstract: if the rule cannot be evaluated synchronously, make it throw so a sync code path fails loudly instead of silently passing.

```csharp
public sealed class UniqueEmailAttribute : AsyncValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext context) =>
        throw new InvalidOperationException("Validate this attribute with IsValidAsync.");

    protected override async Task<ValidationResult?> IsValidAsync(
        object? value, ValidationContext context, CancellationToken cancellationToken)
    {
        var users = context.GetRequiredService<IUserService>();
        if (value is string email && await users.EmailExistsAsync(email, cancellationToken))
        {
            return new ValidationResult("That email is already registered.");
        }

        return ValidationResult.Success;
    }
}
```

### Async object-level validation

`IAsyncValidatableObject` extends `IValidatableObject`; implement both. `[EnumeratorCancellation]` needs `using System.Runtime.CompilerServices;`.

```csharp
public class ReservationRequest : IAsyncValidatableObject
{
    [Required]
    public string Email { get; set; } = "";

    public DateOnly Date { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext context) =>
        throw new InvalidOperationException("Validate this type with ValidateAsync.");

    public async IAsyncEnumerable<ValidationResult> ValidateAsync(
        ValidationContext context,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var rooms = context.GetRequiredService<IRoomService>();
        if (!await rooms.HasAvailabilityAsync(Date, cancellationToken))
        {
            yield return new ValidationResult(
                "No rooms are available on that date.", [nameof(Date)]);
        }
    }
}
```

### Minimal APIs

```csharp
builder.Services.AddValidation();

app.MapPost("/reservations", (ReservationRequest request) =>
    Results.Ok(request));
```

The framework validates the body, awaiting async validators, before the handler runs. Minimal API validation always takes the async path.

Validation metadata comes from a source generator in the assembly that calls `AddValidation()`:
- mark root model types with `[ValidatableType]` (types reachable from them are included automatically);
- types in another assembly need an extension method in that assembly that calls `AddValidation()`;
- model types declared in `.razor` files are not supported.

Missing metadata fails silently: no exception, no log entry, the request is simply not validated. When validation "does nothing", check this first.

### Rules to design by

- On the async path only `ValidateAsync`/`IsValidAsync` is called — put the real logic there.
- Validators on the same member may start concurrently and collection items validate in parallel. Rules must not depend on execution order or share mutable state; services resolved from `ValidationContext` must be safe for concurrent use (be careful with a scoped `DbContext`, which is not thread-safe).
- If property-level validation fails, type-level validation (`IAsyncValidatableObject.ValidateAsync`) is skipped. Do not rely on it to report every error at once.
- Services resolved from `ValidationContext` must be registered in the container; prefer `GetRequiredService` so a missing registration fails instead of silently skipping the rule.
- Always pass the `CancellationToken` through to I/O calls.
- Outside Minimal APIs, call `Validator.ValidateObjectAsync`, `TryValidateObjectAsync`, `ValidatePropertyAsync` or `ValidateValueAsync` instead of the sync `Validator` methods, which would hit the throwing sync overrides.
- Localization: use `AddLocalization()` with `ValidationOptions.LocalizerProvider` / `IValidationMessageFormatter`. The Preview 5 `AddValidationLocalization<T>()` / `ErrorMessageKeyProvider` snippets are superseded; do not copy them.

### Blazor

Async form validation uses the async path for per-field validation and on submit:
- `await editContext.ValidateAsync()` to validate the form; `EditContext.Validate()` is obsolete in .NET 11;
- `EditContext.AddValidationTask` to register a custom async rule;
- `EditContext.IsValidationPending(field)` / `IsValidationFaulted(field)` to show pending or failed state in the UI.

With static SSR, custom attributes are not enforced in the browser unless they also provide a client-side rule.

---

## 2. Options pattern

### Registering async validation

```csharp
services.AddOptions<MyOptions>()
    .Configure(o => o.Value = 42)
    .ValidateAsync(o => Task.FromResult(o.Value > 0), "Value must be positive.")
    .ValidateOnStart();

await host.StartAsync();
```

Async validators run during host startup when `ValidateOnStart()` is used. Use them for checks that need I/O (endpoint reachability, secrets existence, etc.).

Related types: `IAsyncValidateOptions<TOptions>`, `AsyncValidateOptions<TOptions>` and its overloads with up to five dependencies (`AsyncValidateOptions<TOptions, TDep1, ..., TDep5>`), and the `ValidateAsync` extension methods on `OptionsBuilder<TOptions>`.

### Breaking change in RC 1 — sync access throws

Starting in .NET 11 RC 1, if an options type has **only** async validators, synchronous access throws `OptionsValidationException` instead of returning an unvalidated instance. Affected: `IOptions<T>.Value`, `IOptionsMonitor<T>.CurrentValue`/`Get`, `IOptionsSnapshot<T>.Value`/`Get`, `IOptionsFactory<T>.Create`. No AppContext switch disables this.

Fix:
- Call `ValidateOnStart()` and finish `host.StartAsync()` before any synchronous access to the options.
- Keep at least one synchronous validator (`Validate(...)`, DataAnnotations, or `IValidateOptions<T>`) on the type. `IOptionsSnapshot<T>` in later scopes and `IOptionsMonitor<T>` after a configuration reload recreate options synchronously and do not rerun async validators.

### Implementing `IAsyncValidateOptions<T>` directly

Since RC 1 the interface derives from `IValidateOptions<TOptions>` and is no longer contravariant (`in TOptions` removed). Implementers must also provide `Validate(string? name, TOptions options)`:
- return `ValidateOptionsResult.Skip` when the validator does not apply to that name/instance;
- return `ValidateOptionsResult.Fail(...)` when the check cannot be performed synchronously.

Preview 6/7 code (and blog posts from before RC 1) that implements only `ValidateAsync` or relies on `in TOptions` must be updated and recompiled.

---

## Checklist

- [ ] `net11.0` target.
- [ ] Real logic lives in `IsValidAsync`/`ValidateAsync`; sync overrides throw or return `Fail`/`Skip` deliberately.
- [ ] No order dependence or shared mutable state between async validators; `CancellationToken` flows to I/O.
- [ ] Minimal APIs: `builder.Services.AddValidation()` registered; root model types marked `[ValidatableType]`.
- [ ] Blazor: submit uses `EditContext.ValidateAsync()`.
- [ ] Options: `ValidateOnStart()` + await startup before sync access; at least one sync validator kept for snapshot/monitor paths.

## More info

- https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview6/aspnetcore.md
- https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview6/libraries.md
- https://learn.microsoft.com/dotnet/core/compatibility/extensions/11/async-options-validation-sync-access
- https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/aspnetcore.md
- https://learn.microsoft.com/aspnet/core/fundamentals/validation
