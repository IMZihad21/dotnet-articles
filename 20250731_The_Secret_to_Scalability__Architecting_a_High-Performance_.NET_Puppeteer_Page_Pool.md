Generating PDFs from HTML by spawning Chromium per request works initially but fails under load due to concurrency limits, memory issues, and crashing containers. This implementation solves these problems with a persistent browser instance and intelligent page pooling.

---

#### Core Benefits
- Eliminates per-request Chromium spawning  
- Handles high concurrency with capped resources  
- Self-healing on browser crashes  
- Seamless ASP.NET Core and Kubernetes integration  

---

### Complete Implementation
Production-validated code with all original components preserved.

#### 1. Initialization and Browser Launch
**Purpose**: Ensures Chromium is ready before handling requests  
**Key Features**: Thread-safe startup with crash protection  

```csharp
public async Task InitializeAsync()
{
    // Exclusive lock for thread safety
    await _browserGate.WaitAsync();
    try
    {
        await StartBrowserAsync();
        await SeedPagePoolAsync();  // Prewarm pages
    }
    finally
    {
        _browserGate.Release();
    }
}

private async Task StartBrowserAsync()
{
    // Handle Chromium download if missing
    var path = Environment.GetEnvironmentVariable("PUPPETEER_EXECUTABLE_PATH");
    if (string.IsNullOrWhiteSpace(path) || !File.Exists(path))
    {
        var fetcher = new BrowserFetcher();
        var revision = await fetcher.DownloadAsync();
        path = revision.GetExecutablePath();
    }

    // Configure for stability and security
    _browser = await Puppeteer.LaunchAsync(new LaunchOptions
    {
        ExecutablePath = path,
        Args = [ 
            "--no-sandbox", 
            "--disable-dev-shm-usage",  // Critical for Docker
            "--headless=new"            // Modern headless mode
        ]
    });

    // Automatic crash recovery
    _browser.Disconnected += async (_, _) => 
    {
        await RestartBrowserAsync();
    };
}

private async Task SeedPagePoolAsync()
{
    // Concurrent page creation
    var pageTasks = Enumerable.Range(0, PoolLimit)
        .Select(_ => SpawnPageAsync());
    
    await Task.WhenAll(pageTasks);
}

private async Task SpawnPageAsync()
{
    const int maxAttempts = 3;
    int tries = 0;

    // Retry logic with exponential backoff
    while (tries < maxAttempts)
    {
        try
        {
            var page = await _browser.NewPageAsync();
            _pageQueue.Enqueue(page);
            _pageSemaphore.Release();
            Interlocked.Increment(ref _activePages);
            log.LogInformation("Page spawned. Active count: {Count}", _activePages);
            return;
        }
        catch (Exception ex)
        {
            tries++;
            log.LogError(ex, "Page creation failed (attempt {Try})", tries);
            if (tries == maxAttempts) throw;
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, tries)));
        }
    }
}
```

---

#### 2. Page Pool Management
**Purpose**: Reuses pages to eliminate creation overhead  
**Key Features**: Health validation and thread-safe operations  

```csharp
private async Task<IPage> CheckoutPageAsync()
{
    // Timeout after 60 seconds
    if (!await _pageSemaphore.WaitAsync(TimeSpan.FromSeconds(60)))
        throw new TimeoutException("Timed out acquiring page");

    // Validate page health
    while (_pageQueue.TryDequeue(out var page))
    {
        if (await IsPageAliveAsync(page)) return page;
        await DisposePageAsync(page);  // Remove unusable pages
    }

    // Create new page if under pool limit
    while (true)
    {
        var current = _activePages;
        if (current >= PoolLimit) break;

        if (Interlocked.CompareExchange(ref _activePages, current + 1, current) == current)
        {
            try
            {
                var newPage = await _browser.NewPageAsync();
                log.LogInformation("Spawned fallback page. Active count: {Count}", _activePages);
                return newPage;
            }
            catch
            {
                Interlocked.Decrement(ref _activePages);
                throw;
            }
        }
    }
    
    throw new InvalidOperationException("Pool exhausted");
}

private async Task ReturnPageAsync(IPage page)
{
    // Cleanup closed pages
    if (page.IsClosed || !_browser.IsConnected)
    {
        await DisposePageAsync(page);
        return;
    }

    try
    {
        // Reset page state
        await page.GoToAsync("about:blank");
        _pageQueue.Enqueue(page);
        _pageSemaphore.Release();  // Return to pool
    }
    catch (Exception ex)
    {
        log.LogError(ex, "Failed to reset page");
        await DisposePageAsync(page);
    }
}

private async Task DisposePageAsync(IPage page)
{
    try
    {
        if (!page.IsClosed) await page.CloseAsync();
    }
    catch (Exception ex)
    {
        log.LogError(ex, "Error closing page");
    }
    finally
    {
        Interlocked.Decrement(ref _activePages);
        log.LogInformation("Page disposed. Active count: {Count}", _activePages);
    }
}

// Page health validation
private static async Task<bool> IsPageAliveAsync(IPage page)
{
    try
    {
        return !page.IsClosed && 
               await page.EvaluateExpressionAsync<string>("document.readyState") 
                   is "interactive" or "complete";
    }
    catch
    {
        return false;
    }
}
```

---

#### 3. Crash Recovery System
**Purpose**: Automatically recovers from browser crashes  
**Key Features**: Full state rebuild without downtime  

```csharp
private async Task RestartBrowserAsync()
{
    // Freeze operations during recovery
    await _browserGate.WaitAsync();
    try
    {
        // Cleanup dead pages
        while (_pageQueue.TryDequeue(out var p))
            await DisposePageAsync(p);
        
        // Reset semaphore
        while (_pageSemaphore.CurrentCount > 0)
            await _pageSemaphore.WaitAsync();
        
        // Close zombie browser
        if (_browser?.IsConnected == true)
            await _browser.CloseAsync();
        
        // Reinitialize everything
        await StartBrowserAsync();
        await SeedPagePoolAsync();
    }
    finally
    {
        _browserGate.Release();
    }
}
```

---

#### 4. PDF Rendering Core
**Purpose**: Converts HTML to PDF efficiently  
**Key Features**: Network idle wait and retry logic  

```csharp
public async Task<byte[]> RenderPdfAsync(string html)
{
    const int maxRetries = 3;
    int attempt = 0;
    
    while (attempt < maxRetries)
    {
        IPage? page = null;
        try
        {
            page = await CheckoutPageAsync();
            
            // Wait for full page load
            await page.SetContentAsync(html, new NavigationOptions
            {
                WaitUntil = [WaitUntilNavigation.Networkidle0],
                Timeout = 60000
            });
            
            return await page.PdfDataAsync(DefaultPdfOptions);
        }
        catch (Exception ex)
        {
            attempt++;
            log.LogError(ex, "PDF generation failed (attempt {Try})", attempt);
            if (page != null) await DisposePageAsync(page);
            if (attempt == maxRetries) throw;
            await Task.Delay(200 * attempt);  // Incremental backoff
        }
        finally
        {
            if (page != null) await ReturnPageAsync(page);
        }
    }
    throw new InvalidOperationException("PDF rendering failed");
}

// PDF configuration defaults
private static readonly PdfOptions DefaultPdfOptions = new()
{
    PrintBackground = true,
    Landscape = false,
    Format = PaperFormat.A4,
    MarginOptions = new MarginOptions
    {
        Top = "30px",
        Bottom = "30px",
        Left = "20px",
        Right = "20px"
    }
};
```

---

#### 5. Health Checks and Integration
**Purpose**: Kubernetes-ready monitoring  
**Key Features**: Browser responsiveness testing  

```csharp
public async Task<bool> IsHealthyAsync(CancellationToken ct = default)
{
    if (_browser == null || !_browser.IsConnected) return false;
    
    try
    {
        // Quick connectivity test
        var version = await _browser.GetVersionAsync()
            .WaitAsync(TimeSpan.FromSeconds(5), ct);
        return !string.IsNullOrEmpty(version);
    }
    catch
    {
        return false;
    }
}

public class ChromiumHealthCheck(ChromiumPdfRenderer renderer) : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct)
    {
        return await renderer.IsHealthyAsync(ct)
            ? HealthCheckResult.Healthy("Browser responsive")
            : HealthCheckResult.Unhealthy("Browser disconnected");
    }
}

public static class ChromiumPdfServiceExtensions
{
    public static IServiceCollection AddChromiumPdfRenderer(this IServiceCollection services)
    {
        services.AddSingleton<ChromiumPdfRenderer>();
        services.AddSingleton<IHealthCheck, ChromiumHealthCheck>();
        return services;
    }

    public static IHealthChecksBuilder AddChromiumHealthCheck(this IHealthChecksBuilder builder)
    {
        return builder.AddCheck<ChromiumHealthCheck>(nameof(ChromiumHealthCheck));
    }

    public static async Task InitializeChromiumPdfRenderer(this IServiceProvider provider)
    {
        var renderer = provider.GetRequiredService<ChromiumPdfRenderer>();
        await renderer.InitializeAsync();  // Critical initialization
    }
}
```

---

#### 6. Startup Configuration
**Purpose**: Ensures proper initialization sequence  
**Key Features**: DI integration and health checks  

```csharp
var builder = WebApplication.CreateBuilder(args);

// Service registration
builder.Services
    .AddChromiumPdfRenderer()
    .AddHealthChecks()
    .AddChromiumHealthCheck();

var app = builder.Build();

// Initialize before handling requests
await app.Services.InitializeChromiumPdfRenderer(); 

app.MapHealthChecks("/healthz"); 

app.Run();
```

---

#### 7. Usage in Controllers
```csharp
public class PdfController(ChromiumPdfRenderer pdf) : ControllerBase
{
    [HttpPost("/pdf")]
    public async Task<IActionResult> Render([FromBody] string html)
    {
        var buffer = await pdf.RenderPdfAsync(html);
        return File(buffer, "application/pdf");
    }
}
```

---

### Architectural Advantages
1. **Persistent Browser** - Single instance handles all requests  
2. **Page Pooling** - Reusable pages eliminate creation overhead  
3. **Automatic Recovery** - Self-healing after crashes  
4. **Concurrency Control** - Semaphores prevent resource exhaustion  
5. **Cloud Native** - Built-in health checks for orchestration  

---

### Full Class Implementation
Complete solution with all original components:

```csharp
namespace PdfGenerator
{
    public class ChromiumPdfRenderer(ILogger<ChromiumPdfRenderer> log)
    {
        // Configuration and state management
        private const int PoolLimit = 20;
        private IBrowser _browser = null!;
        private readonly ConcurrentQueue<IPage> _pageQueue = new();
        private readonly SemaphoreSlim _pageSemaphore = new(0, PoolLimit);
        private readonly SemaphoreSlim _browserGate = new(1, 1);
        private int _activePages;

        private static readonly PdfOptions DefaultPdfOptions = new() { /* ... */ };

        // All methods from previous sections
        public async Task InitializeAsync() { /* ... */ }
        private async Task StartBrowserAsync() { /* ... */ }
        private async Task SeedPagePoolAsync() { /* ... */ }
        private async Task SpawnPageAsync() { /* ... */ }
        private async Task<IPage> CheckoutPageAsync() { /* ... */ }
        private async Task ReturnPageAsync(IPage page) { /* ... */ }
        private async Task DisposePageAsync(IPage page) { /* ... */ }
        private async Task RestartBrowserAsync() { /* ... */ }
        public async Task<byte[]> RenderPdfAsync(string html) { /* ... */ }
        public async Task<bool> IsHealthyAsync() { /* ... */ }
    }
    
    public class ChromiumHealthCheck(ChromiumPdfRenderer renderer) : IHealthCheck
    {
        public async Task<HealthCheckResult> CheckHealthAsync(
            HealthCheckContext context, CancellationToken ct) { /* ... */ }
    }

    public static class ChromiumPdfServiceExtensions
    {
        public static IServiceCollection AddChromiumPdfRenderer(this IServiceCollection services) { /* ... */ }
        public static IHealthChecksBuilder AddChromiumHealthCheck(this IHealthChecksBuilder builder) { /* ... */ }
        public static async Task InitializeChromiumPdfRenderer(this IServiceProvider provider) { /* ... */ }
    }
}
```

---

### Key Takeaways
1. **Avoid per-request browsers** - Use persistent instances  
2. **Implement page pooling** - Reuse pages with health checks  
3. **Plan for failures** - Build automatic recovery  
4. **Control concurrency** - Prevent resource exhaustion  
5. **Monitor health** - Essential for containerized environments  
