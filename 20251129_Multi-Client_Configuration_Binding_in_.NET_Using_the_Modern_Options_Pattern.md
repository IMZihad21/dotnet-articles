Implementing multi-client configuration in .NET is most effective when global settings and client-specific settings are registered through the modern options pattern. When clients are fixed and represented in code, named options provide a clean mechanism for binding each client’s configuration from `appsettings.json`.

## Configuration Structure

The configuration includes a global section and per-client sections. Here's an example JSON structure:

```json
{
  "GlobalSettings": {
    "FeatureXEnabled": true,
    "ApiEndpoint": "https://example.com"
  },
  "Clients": {
    "ClientA": {
      "ConnectionString": "Server=A",
      "Region": "EU"
    },
    "ClientB": {
      "ConnectionString": "Server=B",
      "Region": "US"
    }
  }
}
```

## Defining Clients

Clients are defined as fixed identifiers in code using an enumeration:

```csharp
public enum Client
{
    ClientA,
    ClientB
}
```

## Strongly Typed Models

Strongly typed models map the configuration directly:

```csharp
public class GlobalSettings
{
    public bool FeatureXEnabled { get; set; }
    public string ApiEndpoint { get; set; }
}

public class ClientSettings
{
    public string ConnectionString { get; set; }
    public string Region { get; set; }
}
```

## Registering Options

Registration uses the current options pattern. Global settings are bound once, while client settings are bound per client through named options keyed by the client identifier.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOptions<GlobalSettings>()
    .Bind(builder.Configuration.GetSection("GlobalSettings"));

foreach (Client client in Enum.GetValues(typeof(Client)))
{
    var name = client.ToString();
    builder.Services.AddOptions<ClientSettings>(name)
        .Bind(builder.Configuration.GetSection($"Clients:{name}"));
}

var app = builder.Build();
```

## Consuming Options

Consumers access global settings through standard options and client settings through named options resolved by the client identifier.

```csharp
public class ClientConsumer
{
    private readonly IOptionsSnapshot<ClientSettings> _options;

    public ClientConsumer(IOptionsSnapshot<ClientSettings> options)
    {
        _options = options;
    }

    public ClientSettings Get(Client client)
    {
        return _options.Get(client.ToString());
    }
}
```

This approach keeps configuration centralized, uses fixed client identifiers as stable keys, and relies entirely on the built-in options system for predictable and maintainable multi-client configuration.