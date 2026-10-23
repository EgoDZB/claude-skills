---
name: union-types-net11
description: >
  Guidance on C# 15 union types (the `union` keyword) shipped with .NET 11: declaration syntax,
  implicit union conversions, exhaustive pattern matching, null/default handling, the generated
  [Union] struct + IUnion shape, custom hand-written unions, and non-boxing TryGetValue/HasValue access.
  USE FOR: modeling "one of N types" results (Result/Option/OneOf-style), replacing OneOf or
  hand-rolled discriminated unions, switch exhaustiveness over closed sets of types, any code
  using `union` declarations in projects targeting net11.0 / C# 15. Use it even when the user just
  says "discriminated union", "sum type", "OneOf" or "Result type" in a .NET 11 context.
  DO NOT USE FOR: projects targeting .NET 10 or earlier (no `union` keyword), F# discriminated unions,
  closed class hierarchies or closed enums (separate, uncommitted proposals).
---

# Union Types — C# 15 / .NET 11

A union is a closed set of existing *case types*. A value of the union holds exactly one value of one of those types. The compiler gives you implicit conversions from each case type and exhaustiveness checking in `switch`.

## Target framework and language version

```xml
<TargetFramework>net11.0</TargetFramework>
```

- Introduced in .NET 11 Preview 2. Build without `LangVersion`; add `<LangVersion>preview</LangVersion>` only if the compiler reports CS8652 for `union`.
- `System.Runtime.CompilerServices.UnionAttribute` and `IUnion` ship in the runtime from .NET 11 Preview 5. Only on Preview 2–4 SDKs, declare them yourself:

```csharp
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Class | AttributeTargets.Struct, AllowMultiple = false)]
    public sealed class UnionAttribute : Attribute;

    public interface IUnion
    {
        object? Value { get; }
    }
}
```

## Declaration

Case types are existing types listed in parentheses. Cases are not declared inline (older blog posts showing inline case declarations describe an earlier proposal and are wrong).

```csharp
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);
public union IntOrString(int, string);

public record class None;
public record class Some<T>(T Value);
public union Option<T>(None, Some<T>);
```

Case types can be any type convertible to `object`: classes, structs, interfaces, type parameters, nullable types, other unions.

A union may have a body with methods and computed properties:

```csharp
public record class Meters(double Value);
public record class Feet(double Value);

public union Length(Meters, Feet)
{
    public double TotalMeters => this switch
    {
        Meters m => m.Value,
        Feet f => f.Value * 0.3048,
        null => throw new InvalidOperationException("The Length has no value."),
    };

    public Length Add(Length other) => new Meters(TotalMeters + other.TotalMeters);
}
```

```csharp
public union OneOrMore<T>(T, IEnumerable<T>)
{
    public IEnumerable<T> AsEnumerable() => Value switch
    {
        T single => [single],
        IEnumerable<T> multiple => multiple,
        null => []
    };
}
```

Body restrictions: no instance fields, auto-properties or field-like events; no public single-parameter constructors (the compiler generates one per case type). Unions get no generated equality, cloning or deconstruction.

## Conversions

Each case type converts implicitly to the union through the generated constructor:

```csharp
Pet pet = new Dog("Rex");
Console.WriteLine(pet.Value);
```

- A user-defined implicit operator for the same source type wins over the union conversion.
- If several case types are equally applicable to a source value, the conversion is ambiguous and is a compile error. Avoid overlapping case types (e.g. `object` together with `string`).
- `T?` conversions work when `T` is a union type.

## Pattern matching and exhaustiveness

Patterns apply to the union's `Value`, not to the union itself:

```csharp
var name = pet switch
{
    Dog d => d.Name,
    Cat c => c.Name,
    Bird b => b.Name,
};
```

- A switch that handles every case type is exhaustive: do not add `_` just to silence warnings, because a discard hides the missing-case warning when a new case type is added later.
- Exceptions: `_`, `var` and `not` patterns apply to the union value itself.
- `pet is Pet` is always true (CS0183). A leading `not null` does not unwrap either, so `pet is not null` is always true for a struct union. To test for a value use `pet.Value is not null`.
- Positional patterns work on case types: `Dog(var name) => name`.
- Runtime check for an empty union of unknown type: `value is IUnion { Value: null }`.

### Null and default

`default(Pet)` has `Value == null`. Without a `null` arm, switching over it throws `SwitchExpressionException`. Add a `null` arm whenever a default or uninitialized union can reach the switch (fields, arrays, `default` returns):

```csharp
var description = pet switch
{
    Dog d => d.Name,
    Cat c => c.Name,
    Bird b => b.Name,
    null => "no pet",
};
```

If any case type is nullable, `Value` is "maybe null" and the compiler requires a `null` arm (CS8655). For class-based unions and `Pet?`, `null` matches both a null reference and a null `Value`.

## What the compiler generates

A `union` declaration becomes a struct:

```csharp
[Union] public struct Pet : IUnion
{
    public Pet(Cat value) => Value = value;
    public Pet(Dog value) => Value = value;
    public Pet(Bird value) => Value = value;
    public object? Value { get; }
}
```

Consequences: always a struct, contents stored as a single `object?`, value-type cases (`int`, `bool`, structs) are boxed. On hot paths with value-type cases, prefer a hand-written union with the non-boxing access pattern below.

## Hand-written (custom) unions

Any class or struct is treated as a union if it has:
- the `[System.Runtime.CompilerServices.Union]` attribute;
- one or more public constructors, each with a single by-value or `in` parameter (one per case type);
- a public `object? Value { get; }` (implementing `IUnion`).

```csharp
[System.Runtime.CompilerServices.Union]
public struct Shape : System.Runtime.CompilerServices.IUnion
{
    private readonly object? _value;

    public Shape(Circle value) { _value = value; }
    public Shape(Rectangle value) { _value = value; }

    public object? Value => _value;
}
```

```csharp
[System.Runtime.CompilerServices.Union]
public class Result<T> : System.Runtime.CompilerServices.IUnion
{
    private readonly object? _value;

    public Result(T? value) { _value = value; }
    public Result(Exception? value) { _value = value; }

    public object? Value => _value;
}
```

The compiler trusts the type, so it must behave soundly: `Value` is always null or an instance of one case type, it never changes after construction, constructing from a value and reading `Value` round-trips, and `TryGetValue`/`HasValue` (if present) agree with `Value`.

### Non-boxing access

Add `bool HasValue` and one `bool TryGetValue(out TCase value)` per case type; the `out` parameter must be identity-convertible to the case type (or to the underlying type for a nullable value-type case). The compiler then uses `TryGetValue` for type patterns and `HasValue` for null patterns instead of reading `Value`, which avoids boxing for value-type cases. Store value-type cases in typed fields plus a tag instead of `object?`.

### Union member providers

Instead of constructors, a type can delegate its union members to a nested `IUnionMembers` interface with one static `Create` method per case type and a `Value` property (`TryGetValue`/`HasValue` optional). Supported since Preview 5; not available on Preview 2–4 SDKs.

## Out of scope

- Closed hierarchies and closed enums are separate proposals, not part of C# 15.

## Checklist

- [ ] `net11.0` target; `LangVersion preview` only if the SDK needs it.
- [ ] Case types are existing types; no overlapping/ambiguous cases.
- [ ] Switches list every case type without a catch-all `_`.
- [ ] `null` arm wherever a default union or nullable case type can appear.
- [ ] Value-type cases on hot paths use a custom union with `TryGetValue`/`HasValue`.

## More info

- https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/union
- https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/csharp.md
- https://devblogs.microsoft.com/dotnet/csharp-15-union-types/
