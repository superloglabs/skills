---
name: otel-dotnet-style
description: ".NET / C# OpenTelemetry style for ASP.NET Core and Worker Services: one AddOpenTelemetry bootstrap, per-signal OTLP/HTTP endpoints, registered ActivitySource/Meter, ILogger log bridge, error spans, and background-worker spans."
---

# OTel .NET Style

Use this skill for ASP.NET Core apps, minimal APIs, and Worker Services
(`BackgroundService`, queue/Kafka/MQTT consumers). Read `otel-onboarding-style`
first for the cross-language rules.

.NET has three failure modes that produce **no error and no telemetry**. Each
section below that fixes one is marked **silent failure**. Check all three
before reporting the install as done.

## Packages

```sh
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Instrumentation.AspNetCore   # web apps only
dotnet add package OpenTelemetry.Instrumentation.Http
```

Add `OpenTelemetry.Instrumentation.SqlClient`, `Npgsql.OpenTelemetry`
(`.AddNpgsql()`), or `OpenTelemetry.Instrumentation.EntityFrameworkCore` only
when the project already uses that data layer. Do not add the console exporter.

## Bootstrap

Keep all setup in one provider-neutral file (`Observability.cs`) and call it
from exactly one place in `Program.cs`. Each call adds another set of exporters,
so a second call makes every signal export twice. If the repo has several
entrypoints that share startup code, make sure only one path reaches it.

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using OpenTelemetry.Exporter;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

public static class Observability
{
    // Superlog public ingest config: write-only, safe in source.
    private const string SuperlogEndpoint = "https://intake.superlog.sh";
    private const string SuperlogPublicToken = "sl_public_...";

    public const string ServiceName = "mugline.api";
    public static readonly ActivitySource Tracer = new(ServiceName);
    public static readonly Meter Meter = new(ServiceName);

    public static IHostApplicationBuilder AddObservability(this IHostApplicationBuilder builder)
    {
        var attributes = new Dictionary<string, object>
        {
            ["deployment.environment.name"] = builder.Environment.EnvironmentName.ToLowerInvariant(),
            ["vcs.repository.url.full"] = "https://github.com/acme/mugline",
        };
        // Best effort: leave the attribute off rather than report a placeholder revision.
        var revision = Environment.GetEnvironmentVariable("GITHUB_SHA")
            ?? Environment.GetEnvironmentVariable("SOURCE_COMMIT");
        if (!string.IsNullOrEmpty(revision)) attributes["vcs.ref.head.revision"] = revision;

        builder.Services.AddOpenTelemetry()
            .ConfigureResource(resource => resource
                .AddService(ServiceName, serviceVersion: typeof(Observability).Assembly.GetName().Version?.ToString())
                .AddAttributes(attributes))
            .WithTracing(tracing => tracing
                .AddSource(ServiceName)
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddOtlpExporter(o => Superlog(o, "traces")))
            .WithMetrics(metrics => metrics
                .AddMeter(ServiceName)
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddOtlpExporter(o => Superlog(o, "metrics")))
            .WithLogging(logging => logging
                .AddOtlpExporter(o => Superlog(o, "logs")));
        return builder;
    }

    // The token MUST be sent as the `x-api-key` header. Ingest only reads
    // `x-api-key` or `Authorization: Bearer <token>`; any other header name 401s.
    private static void Superlog(OtlpExporterOptions options, string signal)
    {
        options.Protocol = OtlpExportProtocol.HttpProtobuf;
        options.Endpoint = new Uri($"{SuperlogEndpoint}/v1/{signal}");
        options.Headers = $"x-api-key={SuperlogPublicToken}";
    }
}
```

```csharp
var builder = WebApplication.CreateBuilder(args);   // or Host.CreateApplicationBuilder(args) for a worker
builder.AddObservability();
```

Drop `AddAspNetCoreInstrumentation()` in a Worker Service that has no HTTP
server.

### Endpoint path (silent failure)

The exporter defaults to gRPC; set `HttpProtobuf` explicitly. When `Endpoint`
is set **in code** with `HttpProtobuf`, the .NET exporter uses the URL
verbatim. It appends `/v1/traces` only when the endpoint comes from the
`OTEL_EXPORTER_OTLP_ENDPOINT` env var. Since Superlog config is inlined in
source, every exporter needs its full per-signal URL:

- `https://intake.superlog.sh/v1/traces`
- `https://intake.superlog.sh/v1/metrics`
- `https://intake.superlog.sh/v1/logs`

Do not:

```csharp
options.Endpoint = new Uri("https://intake.superlog.sh"); // POSTs to "/", gets 404, logs nothing
```

`Headers` is a single `key=value` string (comma-separated for several), not a
dictionary.

### Register sources and meters (silent failure)

A span from an `ActivitySource` that was never passed to `.AddSource(name)` is
dropped: `StartActivity` returns `null` and the `?.` calls hide it. The same
applies to a `Meter` without `.AddMeter(name)`. Use one shared name for the
source, the meter, and both registrations. If the repo already has other
`ActivitySource` names, register each one.

### Logs (silent failure)

`WithTracing` and `WithMetrics` do not export logs. `ILogger` is the log
bridge: `.WithLogging(...)` attaches an OTLP provider underneath the existing
logging pipeline, and every `ILogger` record written inside a span carries
`trace_id` / `span_id`.

Keep the project's existing logger setup, console/file providers, and levels.
If the project uses Serilog, keep it and write to the Microsoft logging
providers (`AddSerilog(..., writeToProviders: true)`) or add
`Serilog.Sinks.OpenTelemetry`; do not replace Serilog with `ILogger` calls.

## Bounded work

`using var` ends the span when the scope exits. Span names are `domain.verb`.

```csharp
public async Task<Order> SubmitAsync(string tenantId, string orderId)
{
    using var activity = Observability.Tracer.StartActivity("order.submit");
    activity?.SetTag("tenant.id", tenantId);
    activity?.SetTag("order.id", orderId);
    ...
}
```

Do not call `StartActivity` and `Stop()` in separate places for bounded work,
and do not construct an `ActivitySource` per call or per request.

## Error paths

Record the exception and set the status on the current activity, then rethrow.

```csharp
catch (Exception ex)
{
    Activity.Current?.AddException(ex);
    Activity.Current?.SetStatus(ActivityStatusCode.Error, ex.Message);
    logger.LogError(ex, "order submit failed for tenant {TenantId}", tenantId);
    throw;
}
```

`AddException` needs .NET 9+ or `System.Diagnostics.DiagnosticSource` 9+. On
older targets use `RecordException` from `OpenTelemetry.Trace`.

Use `throw;`, not `throw ex;`.

## Background workers and consumers

ASP.NET Core creates a span per request. A `BackgroundService` has no ambient
span, so anything it does is untraced unless the loop opens one. Open one span
per unit of work (per message, per batch, per tick), not one span around the
whole `ExecuteAsync` loop.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await foreach (var message in consumer.ReadAllAsync(stoppingToken))
    {
        using var activity = Observability.Tracer.StartActivity("meter_reading.process", ActivityKind.Consumer);
        activity?.SetTag("messaging.system", "kafka");
        activity?.SetTag("messaging.destination.name", message.Topic);
        try
        {
            await handler.HandleAsync(message, stoppingToken);
        }
        catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
        {
            throw; // host shutdown, not a failure: do not record it or keep looping
        }
        catch (Exception ex)
        {
            activity?.AddException(ex);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            logger.LogError(ex, "meter reading failed on {Topic}", message.Topic);
        }
    }
}
```

If the producer put a W3C `traceparent` in the message headers, pass it as the
parent so the consumer span joins the producer's trace:

```csharp
ActivityContext.TryParse(traceparent, null, isRemote: true, out var parent);
using var activity = Observability.Tracer.StartActivity("meter_reading.process", ActivityKind.Consumer, parent);
```

## Metrics

Create instruments once, as static fields next to the meter.

```csharp
public static readonly Counter<long> OrdersSubmitted =
    Observability.Meter.CreateCounter<long>("orders.submitted");
public static readonly Histogram<double> OrderDuration =
    Observability.Meter.CreateHistogram<double>("order.duration", unit: "ms");

OrdersSubmitted.Add(1,
    new KeyValuePair<string, object?>("tenant.id", tenantId),
    new KeyValuePair<string, object?>("outcome", "success"));
```

Low-cardinality tags only. For LLM token and cost telemetry follow
`otel-onboarding-style`; do not add app-side cost metrics.

## Smoke check

After install, hit one endpoint (or let the worker process one message) and
confirm all three signals leave the process:

- a custom span exists: `StartActivity(...)` returns non-null
- exports go to `/v1/traces`, `/v1/metrics`, and `/v1/logs`, not `/`
- an `ILogger` line written inside a request shows up with a `trace_id`

The exporter does not log failed exports by default. To see them, enable the
SDK self-diagnostics file (`OTEL_DIAGNOSTICS.json` with
`{"LogDirectory": ".", "FileSize": 1024, "LogLevel": "Warning"}` in the
working directory), then remove it once the install is verified.
