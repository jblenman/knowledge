# C# / .NET Knowledge

## Async / parallel — the three things people confuse

- **Sequential `await`** — one call after another, but the thread is freed between awaits. Scales *throughput* on a server because threads aren't blocked; does nothing for end-to-end latency of a single request.
- **`Task.WhenAll`** — fire multiple independent I/O-bound tasks, await them all. Reduces *latency* when calls don't depend on each other. Still single-threaded in terms of CPU work.
- **True parallelism** (`Parallel.For`, `Task.Run` across cores) — for CPU-bound work. Wrong tool for HTTP-bound workloads.

Rule of thumb: for a pile of independent HTTP calls, reach for `Task.WhenAll`, not `Parallel.For`.

## Cursor-based paging — generic helper

Many APIs (OData-flavored, Microsoft Graph, AWS, etc.) return a continuation token + a preconstructed "next page URL". Pages must be fetched sequentially — you don't know the next URL until the current response lands. A generic helper keeps callers clean:

```csharp
public record PagingInfo(int Count, int PageSize, string? ContinuationToken, string? NextPageUrl);
public record PagedResponse<T>(IReadOnlyList<T> Value, PagingInfo Paging);

public async Task<List<T>> FetchAllPagesAsync<T>(
    string initialUrl,
    CancellationToken ct = default)
{
    var all = new List<T>();
    var url = initialUrl;
    while (!string.IsNullOrEmpty(url))
    {
        var page = await _http.GetFromJsonAsync<PagedResponse<T>>(url, ct)
                   ?? throw new InvalidOperationException($"Null response from {url}");
        all.AddRange(page.Value);
        url = page.Paging.NextPageUrl;
    }
    return all;
}
```

Notes:
- Pages can't be parallelized in cursor paging — only offset/`$skip` paging allows that.
- If the API enforces a per-page response size cap (e.g. 25 MB), tune `pageSize` against that, not against record count.
- If the API supports field projection (`$select`), pass it through to shrink payloads before resorting to ETL.

## `HttpClient` with conditional proxy

When the same build runs locally (no proxy) and on a server that sits behind an outbound HTTP proxy, drive the proxy decision from `appsettings.json` rather than code branches:

```csharp
// Program.cs
builder.Services.AddHttpClient("api", c =>
{
    c.BaseAddress = new Uri(builder.Configuration["Api:BaseUrl"]!);
})
.ConfigurePrimaryHttpMessageHandler(sp =>
{
    var proxyUrl = builder.Configuration["Api:ProxyUrl"];
    var handler = new HttpClientHandler();
    if (!string.IsNullOrWhiteSpace(proxyUrl))
    {
        handler.Proxy = new WebProxy(proxyUrl) { UseDefaultCredentials = true };
        handler.UseProxy = true;
    }
    return handler;
});
```

`appsettings.json` has no `ProxyUrl` locally; `appsettings.Production.json` (or the server's environment-specific file) supplies it. No `#if DEBUG`, no code changes between environments.
