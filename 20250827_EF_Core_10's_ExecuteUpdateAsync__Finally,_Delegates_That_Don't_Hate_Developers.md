## The Problem: Expression Trees Were a Pain

```csharp
// EF Core 9's special kind of torture
Expression<Func<SetPropertyCalls<Blog>, SetPropertyCalls<Blog>>> setters =
    s => s.SetProperty(b => b.Views, 8);

if (nameChanged)
{
    var blogParameter = Expression.Parameter(typeof(Blog), "b");
    setters = Expression.Lambda<Func<SetPropertyCalls<Blog>, SetPropertyCalls<Blog>>>(
        Expression.Call(
            instance: setters.Body,
            methodName: nameof(SetPropertyCalls<Blog>.SetProperty),
            typeArguments: new[] { typeof(string) },
            arguments: new Expression[]
            {
                Expression.Lambda<Func<Blog, string>>(
                    Expression.Property(blogParameter, nameof(Blog.Name)),
                    blogParameter),
                Expression.Constant("foo")
            }),
        setters.Parameters);
}
```

Remember this? This wasn't code - it was a cry for help. Expression tree manipulation for conditional updates was like performing brain surgery with oven mitts.

## The Solution: Delegates to the Rescue

EF Core 10 replaced the expression tree requirement with a simple delegate:

```csharp
// EF Core 10: Actually readable code
await context.Blogs.ExecuteUpdateAsync(s =>
{
    s.SetProperty(b => b.Views, 8);
    if (nameChanged)  // Yes, a real if statement!
    {
        s.SetProperty(b => b.Name, "foo");
    }
});
```

## Why This Matters

1. **Actual C# Code**: Use `if` statements, loops, and variables like a normal developer
2. **Compile-time safety**: The compiler actually understands what you're writing
3. **Maintainability**: The next developer won't want to murder you

## Migration: It's Actually Simple

Replace this:
```csharp
// Old way (bad)
Expression<Func<SetPropertyCalls<T>, SetPropertyCalls<T>>> expr = ...;
```

With this:
```csharp
// New way (good)
Func<SetPropertyCalls<T>, SetPropertyCalls<T>> del = ...;
```

Yes, it's that straightforward. If you've built complex expression tree machinery, you can now delete 80% of that code and actually understand what's happening.

## The Technical Why

The EF team finally acknowledged that forcing expression trees for everything was architectural overkill. The new approach:

- Still generates optimal SQL
- Doesn't sacrifice performance
- Actually respects developers' time

## Bottom Line

This breaking change is one of those rare improvements where you delete code and everything gets better. If you're not using EF Core 10 yet, this alone might be worth the upgrade.

Just don't tell your team how much time you wasted on expression trees before this existed.

---

## References

- **Official Documentation**: [EF Core 10 Breaking Changes](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/breaking-changes)
- **ExecuteUpdate Documentation**: [ExecuteUpdate and ExecuteDelete](https://learn.microsoft.com/en-us/ef/core/saving/execute-insert-update-delete)
- **GitHub Discussion**: [Accept non-expression setters parameter in ExecuteUpdate](https://github.com/dotnet/efcore/issues/32018)