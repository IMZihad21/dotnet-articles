We all know the pain: you need to generate a PDF from HTML. The naive solution? Spin up a Chromium instance per request. That works—until it doesn’t. You hit concurrency limits. Memory usage explodes. The app gets sluggish. Eventually, you’re rebooting containers just to keep things alive.

This post shows you how to **do it right**.

We’ll walk through a real-world implementation of a **headless Chromium PDF service** in .NET 8 using **PuppeteerSharp**, with:

* a single persistent browser instance
* a capped pool of prewarmed pages
* automatic recovery on browser crash
* full DI integration
* health checks

Let’s go.

---

## What This Service Solves

* Avoids spawning Chromium per request
* Supports up to N concurrent PDF renders
* Self-heals on browser crashes
* Plays nice with ASP.NET Core and K8s

---

## The Architecture (TL;DR)

* Chromium is launched once on startup.
* A pool of reusable pages (`IPage`) is created.
* Requests grab pages from the pool, render, and return them.
* Health checks verify Chromium is alive.
* If Chromium crashes, the service rebuilds everything from scratch.

---

## The Core: `ChromiumPdfRenderer.cs`

```csharp
namespace PdfGenerator
{
    public class ChromiumPdfRenderer(ILogger<ChromiumPdfRenderer> log)
    {
        private const int PoolLimit = 20;

        private IBrowser _browser = null!;
        private readonly ConcurrentQueue<IPage> _pageQueue = new();
        private readonly SemaphoreSlim _pageSemaphore = new(0, PoolLimit);
        private readonly SemaphoreSlim _browserGate = new(1, 1);
        private int _activePages;

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

        public async Task InitializeAsync()
        {
            await _browserGate.WaitAsync();
            try
            {
                await StartBrowserAsync();
                await SeedPagePoolAsync();
            }
            finally
            {
                _browserGate.Release();
            }
        }

        private async Task StartBrowserAsync()
        {
            var path = Environment.GetEnvironmentVariable("PUPPETEER_EXECUTABLE_PATH");

            if (string.IsNullOrWhiteSpace(path) || !File.Exists(path))
            {
                log.LogInformation("Downloading Chromium...");
                var fetcher = new BrowserFetcher();
                var revision = await fetcher.DownloadAsync();
                path = revision.GetExecutablePath();
            }

            _browser = await Puppeteer.LaunchAsync(new LaunchOptions
            {
                ExecutablePath = path,
                Headless = true,
                Timeout = 60000,
                Args =
                [
                    "--no-sandbox",
                    "--disable-setuid-sandbox",
                    "--disable-gpu",
                    "--disable-dev-shm-usage",
                    "--disable-extensions",
                    "--disable-sync",
                    "--metrics-recording-only",
                    "--no-first-run",
                    "--mute-audio",
                    "--hide-scrollbars",
                    "--headless=new"
                ]
            });

            _browser.Disconnected += async (_, _) =>
            {
                try
                {
                    log.LogWarning("Browser disconnected. Restarting...");
                    await RestartBrowserAsync();
                }
                catch (Exception ex)
                {
                    log.LogError(ex, "Error during browser restart");
                }
            };
        }

        private async Task SeedPagePoolAsync()
        {
            var pages = Enumerable.Range(0, PoolLimit)
                .Select(_ => SpawnPageAsync());

            await Task.WhenAll(pages);
        }

        private async Task SpawnPageAsync()
        {
            const int maxAttempts = 3;
            int tries = 0;

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
                    log.LogError(ex, "Failed to create page (try {Try})", tries);
                    if (tries == maxAttempts) throw;
                    await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, tries)));
                }
            }
        }

        private async Task<IPage> CheckoutPageAsync()
        {
            if (!await _pageSemaphore.WaitAsync(TimeSpan.FromSeconds(60)))
                throw new TimeoutException("Timed out acquiring a browser page.");

            while (_pageQueue.TryDequeue(out var page))
            {
                if (await IsPageAliveAsync(page)) return page;
                await DisposePageAsync(page);
            }

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

            throw new InvalidOperationException("No available pages and pool maxed out.");
        }

        private static async Task<bool> IsPageAliveAsync(IPage page)
        {
            try
            {
                return !page.IsClosed &&
                       await page.EvaluateExpressionAsync<string>("document.readyState") is "interactive" or "complete";
            }
            catch
            {
                return false;
            }
        }

        private async Task ReturnPageAsync(IPage page)
        {
            if (page.IsClosed || !_browser.IsConnected)
            {
                await DisposePageAsync(page);
                return;
            }

            try
            {
                await page.GoToAsync("about:blank");
                _pageQueue.Enqueue(page);
                _pageSemaphore.Release();
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
                if (!page.IsClosed)
                    await page.CloseAsync();
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

        private async Task RestartBrowserAsync()
        {
            await _browserGate.WaitAsync();
            try
            {
                while (_pageQueue.TryDequeue(out var p))
                    await DisposePageAsync(p);

                while (_pageSemaphore.CurrentCount > 0)
                    await _pageSemaphore.WaitAsync();

                if (_browser?.IsConnected == true)
                {
                    try { await _browser.CloseAsync(); }
                    catch (Exception ex) { log.LogError(ex, "Error closing browser"); }
                }

                await StartBrowserAsync();
                await SeedPagePoolAsync();
            }
            finally
            {
                _browserGate.Release();
            }
        }

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
                    await Task.Delay(200 * attempt);
                }
                finally
                {
                    if (page != null) await ReturnPageAsync(page);
                }
            }

            throw new InvalidOperationException("PDF rendering failed after retries.");
        }

        public async Task<bool> IsHealthyAsync(CancellationToken ct = default)
        {
            if (_browser == null || !_browser.IsConnected) return false;

            try
            {
                var version = await _browser.GetVersionAsync()
                    .WaitAsync(TimeSpan.FromSeconds(5), ct);

                return !string.IsNullOrEmpty(version);
            }
            catch
            {
                return false;
            }
        }
    }

    public class ChromiumHealthCheck(ChromiumPdfRenderer renderer) : IHealthCheck
    {
        public async Task<HealthCheckResult> CheckHealthAsync(
            HealthCheckContext context,
            CancellationToken cancellationToken = default)
        {
            return await renderer.IsHealthyAsync(cancellationToken)
                ? HealthCheckResult.Healthy("Browser is responsive.")
                : HealthCheckResult.Unhealthy("Browser is disconnected or unresponsive.");
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
            await renderer.InitializeAsync();
        }
    }
}
```

### Initialization

```csharp
public async Task InitializeAsync()
```

This gets called at app startup. It:

* Launches Chromium (downloads if needed)
* Creates a fixed number of reusable pages
* Hooks a crash listener to restart on failure

### Internals

* `ConcurrentQueue<IPage>` holds ready-to-use pages
* `SemaphoreSlim` tracks available pages
* `Interlocked` manages active count

All of this ensures thread safety and pool limits.

---

## Browser Recovery

```csharp
_browser.Disconnected += ...
```

If Chromium crashes (yes, it happens), we:

1. Dispose all pages
2. Drain the semaphore
3. Kill and relaunch Chromium
4. Repopulate the pool

This keeps the app from spiraling when the browser dies mid-request.

---

## Page Pooling

* Pages are created up-front via `NewPageAsync()`
* Each one is reused across requests
* On release, the page is reset with `GoToAsync("about:blank")`
* If the page is broken or closed, we discard and replace it

```csharp
private async Task<IPage> CheckoutPageAsync()
```

This method grabs a usable page or waits with timeout. If no page is healthy, it spawns a new one (as long as we’re under the cap).

---

## PDF Rendering

```csharp
public async Task<byte[]> RenderPdfAsync(string html)
```

The public API. You hand it HTML. It:

1. Acquires a page from the pool
2. Loads the HTML (waits for network idle)
3. Generates PDF via Puppeteer
4. Returns byte array
5. Resets or disposes the page

Also includes retry logic (3x max) with backoff and logs.

---

## Health Check

```csharp
public async Task<bool> IsHealthyAsync()
```

Pings Chromium for its version. If it’s responsive, it’s healthy. If not, we know something went wrong (likely a disconnect or crash).

This is exposed via:

```csharp
public class ChromiumHealthCheck : IHealthCheck
```

Used in your K8s readiness or liveness probes.

---

## `Program.cs` Setup

Here’s how you integrate the whole thing into your ASP.NET Core app:

```csharp
builder.Services
    .AddChromiumPdfRenderer()
    .AddHealthChecks()
    .AddChromiumHealthCheck();

var app = builder.Build();

// Important: call this explicitly to launch browser and seed page pool
await app.Services.InitializeChromiumPdfRenderer();

app.MapHealthChecks("/healthz");

app.Run();
```

This guarantees the browser is ready **before** traffic hits the app.

---

## Usage Example

Inject the service and generate your PDF like this:

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

## Design Notes

Why this pattern?

* **No cold starts**: Chromium is persistent.
* **Concurrency-safe**: Semaphore + pool avoids overload.
* **Crash-safe**: Browser restarts are automatic.
* **K8s-friendly**: Health check integration is built in.
* **Fast**: Page reuse avoids repeated setup overhead.

---

## What You Could Add

* Metrics: expose current pool size, usage stats
* Circuit breaker: disable rendering under pressure
* PDF watermarking or header/footer via Puppeteer options
* Redis-backed job queue to serialize high-volume workloads

---

## Summary

This isn’t a toy. It’s a real-world pattern we’ve run in production under load.

You get:

* Fast, safe PDF generation
* Minimal memory churn
* Stable long-lived Chromium processes

If you're rendering HTML to PDF in .NET, don’t spin up new processes every time. Manage one browser, pool your pages, and let the infra handle the rest.
