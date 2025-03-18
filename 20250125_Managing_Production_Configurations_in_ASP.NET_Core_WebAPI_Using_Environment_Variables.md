When you’re building modern, scalable applications with ASP.NET Core, one of the biggest challenges is managing configurations across different environments—development, staging, and production. You want a system that’s flexible, secure, and easy to update without touching your code or triggering a redeploy. The good news? ASP.NET Core has an amazing built-in configuration system that makes all of this possible.

In this article, we’ll break down exactly how you can use environment variables to manage your configurations, how the `ASPNETCORE_ENVIRONMENT` variable works, and how ASP.NET Core prioritizes settings when you have multiple configuration sources. By the end, you’ll know how to keep your app’s configuration clean, secure, and production-ready.

---

## Why Environment Variables Are a Game-Changer

First things first: why bother with environment variables? The short answer is that they’re:

- Secure: You can keep sensitive data like API keys and connection strings out of your codebase.
- Dynamic: No need to rebuild or redeploy your app to update configurations—just change the variable.
- Consistent: They’re a standard way of managing settings across modern deployment platforms like Docker, Kubernetes, and cloud services.

---

## How ASP.NET Core Handles Configurations

ASP.NET Core’s configuration system is built around **flexibility**. It pulls settings from multiple sources and layers them together. Here’s how it works:

1. Default **appsettings.json** - This file contains your app’s baseline settings—think of it as your starting point.

2. Environment-Specific **appsettings** Files - For example, if you set **ASPNETCORE_ENVIRONMENT=Production**, ASP.NET Core will automatically load **appsettings.Production.json**. This lets you define overrides for specific environments.

3. Environment Variables - These take priority over anything in your **appsettings** files. You can dynamically inject them at runtime, making them perfect for production.

4. Command-Line Arguments - If you really need something to take precedence over everything else, you can pass it as a command-line argument.

---

## Let’s Break It Down: How Environment Variables Work

To show how environment variables override settings, let’s start with a basic example. Imagine this **appsettings.json** file:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "AppSettings": {
    "ApiUrl": "https://api.default.com",
    "ApiKey": "default-api-key"
  }
}
```

**Setting Up an Environment-Specific File**

Now, if you add an **appsettings.Production.json** file for production:

```json
{
  "AppSettings": {
    "ApiUrl": "https://api.production.com"
  }
}
```

When you set **ASPNETCORE_ENVIRONMENT=Production**, ASP.NET Core will automatically merge **appsettings.Production.json** into the configuration. So, **ApiUrl** will now be `https://api.production.com`, while **ApiKey** remains as **default-api-key**.

**Using Environment Variables to Override Everything**

Environment variables take it one step further. If you set the following environment variables:

```bash
AppSettings__ApiUrl=https://api.override.com
AppSettings__ApiKey=override-api-key
```

ASP.NET Core will prioritize these values over both **appsettings.json** and **appsettings.Production.json**. The final configuration would be:

- **ApiUrl**: `https://api.override.com`
- **ApiKey**: **override-api-key**

Notice the pattern here: ASP.NET Core uses **__** (double underscores) to represent nested JSON keys. So, **AppSettings:ApiUrl** in JSON becomes **AppSettings__ApiUrl** as an environment variable.

---

## Priority Order: Who Wins?

If you’re wondering how ASP.NET Core decides which configuration to use when you have multiple sources, here’s the priority list, from lowest to highest:

- **Default values in code**: If you hardcode defaults in your app.
- **appsettings.json**: Your baseline configuration.
- **appsettings.<ASPNETCORE_ENVIRONMENT>.json**: Environment-specific overrides (e.g., **appsettings.Production.json**).
- **Environment variables**: Perfect for production or dynamic settings.
- **Command-line arguments**: The final word—these override everything else.

This order is critical because it ensures that you can always override configurations dynamically, without needing to touch your app’s code or rebuild it.

---

## Using Environment Variables in Docker Compose

Let’s say you’re running your app in Docker Compose. You can easily pass environment variables in the **docker-compose.yml** file:

```yaml
version: "3.9"
services:
  webapi:
    image: yourapp:latest
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - AppSettings__ApiUrl=https://api.docker.com
      - AppSettings__ApiKey=docker-api-key
    ports:
      - "5000:80"
```

Or, if you prefer, you can use an **.env** file to manage these variables:

```yaml
version: "3.9"
services:
  webapi:
    image: yourapp:latest
    env_file:
      - .env
    ports:
      - "5000:80"
```

Here’s what the **.env** file might look like:

```bash
ASPNETCORE_ENVIRONMENT=Production
AppSettings__ApiUrl=https://api.envfile.com
AppSettings__ApiKey=envfile-api-key
```

---

## Using Environment Variables in Kubernetes

In Kubernetes, ConfigMaps and Secrets are the standard way to manage environment variables.

**ConfigMap for Non-Sensitive Data**

Create a ConfigMap like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: appsettings-config
data:
  AppSettings__ApiUrl: https://api.k8s.com
```

**Secret for Sensitive Data**

For things like API keys, use a Secret instead:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: appsettings-secret
type: Opaque
data:
  AppSettings__ApiKey: a3BpLXZhbHVlCg== # Base64-encoded value
```

**Deployment Example**

Then reference these in your app’s deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapi-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapi
  template:
    metadata:
      labels:
        app: webapi
    spec:
      containers:
      - name: webapi
        image: yourapp:latest
        envFrom:
        - configMapRef:
            name: appsettings-config
        - secretRef:
            name: appsettings-secret
```

---

## Using <u>[dotnet-appsettings-env](https://github.com/dassump/dotnet-appsettings-env)</u> Tool (Thanks to <u>[@dassump](https://github.com/dassump)</u>)

Download the tool from <u>[GitHub Releases](https://github.com/dassump/dotnet-appsettings-env/releases)</u>. This tool is designed to quickly convert even large appsettings.json files into environment variable definitions, streamlining the configuration process for deployments using Kubernetes, Docker or Docker Compose.

```bash
# Download from GitHub Releases: https://github.com/dassump/dotnet-appsettings-env/releases
# Display available options:
dotnet-appsettings-env --help
```

By converting extensive appsettings data into environment variables, you can manage configurations more efficiently and reduce manual errors during deployments.

---

## The Big Picture: Production-Ready Configuration

To recap:

1. Start with **appsettings.json** for defaults.
2. Add environment-specific files like **appsettings.Production.json** for targeted overrides.
3. Use environment variables for dynamic settings in production.
4. Remember the priority order: environment variables and command-line arguments always win.

This approach ensures your app is ready for production, no matter where it’s running—on-prem, in Docker, or in Kubernetes. You’ll have full control over configurations without ever touching the code.

So, next time you’re setting up a production environment, skip the manual edits and let ASP.NET Core’s configuration system do the heavy lifting. It’s simple, powerful, and just works.