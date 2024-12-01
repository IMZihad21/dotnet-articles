In this article, I’ll guide you through implementing a custom role-based authorization handler in ASP.NET Core using JWT authentication. This approach allows you to dynamically control API route access based on user roles and route permissions, making it ideal for applications with complex role hierarchies or permissions.

Our solution involves:

- A custom authorization requirement and handler.
- Dynamic permissions management based on role-based access.
- Configuring role-specific routes and frontend permissions for API endpoints.

Let’s dive into the code and see how to set up this custom authorization handler in ASP.NET Core.

**Setting Up the Custom Authorization Requirement**

First, let’s define a custom authorization requirement, `RoleBasedRequirement`, that takes a `requireValidation` parameter. This allows us to configure whether to enforce role-based validation or not, which is useful for environments like development where you might not want to enforce permissions.

```csharp
public class RoleBasedRequirement(bool requireValidation) : IAuthorizationRequirement
{
    public bool RequireValidation { get; } = requireValidation;
}
```

**Defining Globally Allowed Paths**

To make certain API paths accessible to everyone, regardless of roles, we define `GloballyAllowedHttpPath`. This way, specific routes (e.g., login) remain accessible without authentication.

```csharp
public static class GloballyAllowedHttpPath
{
    public const string Login = "Login";
    public const string UpdateUserPassword = "UpdateUserPassword"; 
    public const string GetUserInfobyToken = "GetUserInfobyToken";
    public const string GetUserWiseMenu = "GetUserWiseMenu";
    public const string Dropdown = "Dropdown";
}
```

**Implementing the Custom Authorization Handler**

Now we’ll build `RoleBasedAuthorizationHandler`, the core of our setup. This handler evaluates the request's API route and checks the user’s role-based permissions, using a user manager dependency (`IUserManager`) to fetch the role permissions dynamically.

```csharp
public class RoleBasedAuthorizationHandler : AuthorizationHandler<RoleBasedRequirement>
{
    private readonly IUserManager _userManager;
    private readonly IWebHostEnvironment _environment;

    public RoleBasedAuthorizationHandler(IUserManager userManager, IWebHostEnvironment environment)
    {
        _userManager = userManager;
        _environment = environment;
    }

    protected override async Task HandleRequirementAsync(AuthorizationHandlerContext context, RoleBasedRequirement requirement)
    {
        if (!requirement.RequireValidation || context.Resource is not HttpContext httpContext)
        {
            context.Succeed(requirement);
            return;
        }

        try
        {
            if (IsGloballyAllowedPath(httpContext.Request.Path.Value) || 
                !long.TryParse(context.User.FindFirstValue("RoleId"), out var userRoleId))
            {
                context.Succeed(requirement);
                return;
            }

            if (!httpContext.Request.Headers.TryGetValue("X-Active-Route", out StringValues routeHeader) ||
                string.IsNullOrEmpty(routeHeader.FirstOrDefault()))
            {
                await WriteForbiddenResponseAsync(httpContext);
                return;
            }

            var pathSegments = httpContext.Request.Path.Value.Split('/', StringSplitOptions.RemoveEmptyEntries);
            var apiRoute = pathSegments.Length >= 3 ? $"{pathSegments.ElementAtOrDefault(1)}/{pathSegments.ElementAtOrDefault(2)}" : string.Empty;
            var frontendRoute = routeHeader.FirstOrDefault()?.Split('/', StringSplitOptions.RemoveEmptyEntries).ElementAtOrDefault(0);

            if (string.IsNullOrEmpty(apiRoute) || string.IsNullOrEmpty(frontendRoute))
            {
                await WriteForbiddenResponseAsync(httpContext);
                return;
            }

            var permissions = await _userManager.GetRolewiseUserMenuPermission();
            var rolePermissions = permissions?
                .Where(x => x.RoleId == userRoleId &&
                            x.FrontendRoute.Equals(frontendRoute, StringComparison.OrdinalIgnoreCase) &&
                            x.HasPermission)
                .SelectMany(x => x.ApiRoute.Split(',', StringSplitOptions.RemoveEmptyEntries))
                .ToHashSet(StringComparer.OrdinalIgnoreCase);

            if (rolePermissions is not null && rolePermissions.Contains(apiRoute))
                context.Succeed(requirement);
            else
                await WriteForbiddenResponseAsync(httpContext);
        }
        catch
        {
            await WriteForbiddenResponseAsync(httpContext);
        }
    }

    private static bool IsGloballyAllowedPath(string path)
    {
        FieldInfo[] fields = typeof(GloballyAllowedHttpPath).GetFields(BindingFlags.Public | BindingFlags.Static | BindingFlags.GetField);

        return fields.Any(field =>
        {
            if (field.FieldType == typeof(string))
            {
                return field.GetValue(null) is string fieldValue && path.Contains(fieldValue, StringComparison.OrdinalIgnoreCase);
            }
            return false;
        });
    }

    private static async Task WriteForbiddenResponseAsync(HttpContext httpContext)
    {
        httpContext.Response.StatusCode = StatusCodes.Status403Forbidden;
        await httpContext.Response.WriteAsJsonAsync(Utilities.ForbiddenResponse());
    }
}
```

**Configuring the Service and Middleware**

Finally, we register our handler and set up the authorization policy to utilize this custom role-based requirement. This is configured in `RoleBasedAuthorizationService`.

```csharp
public static class RoleBasedAuthorizationService
{
    public static IServiceCollection AddRoleBasedAuthorization(this IServiceCollection services, IConfiguration configuration)
    {
        var requireValidation = configuration.GetValue<bool>("ApplicationConfiguration:IsRoleWiseMenuActionPermissionEnabled", false);

        services.AddScoped<IAuthorizationHandler, RoleBasedAuthorizationHandler>();

        services.AddAuthorizationBuilder()
            .SetDefaultPolicy(new AuthorizationPolicyBuilder()
                .AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
                .RequireAuthenticatedUser()
                .AddRequirements(new RoleBasedRequirement(requireValidation))
                .Build());

        return services;
    }
}
```

With this setup, you now have a flexible, role-based authorization handler in ASP.NET Core that dynamically controls API access based on user roles and routes. This can be easily adapted to meet various security requirements and scaled to accommodate more complex permission systems as your application grows.