---
name: otel-go-style
description: "Go OpenTelemetry style: module-level tracers/meters, func-based spans, context propagation, OTLP HTTP, resource attributes, logs via slog, and no wrapper APIs."
---

# OTel Go Style

Acquire OTel objects at package scope, not per-call.

```go
var (
    tracer = otel.Tracer("mugline.api")
    meter  = otel.Meter("mugline.api")

    ordersSubmitted = meter.Int64Counter("orders.submitted")
    orderDuration   = meter.Int64Histogram("order.duration_ms",
        metric.WithUnit("ms"),
    )
)
```

## Bootstrap

Initialize the SDK once, early in `main`, before any HTTP server or worker starts. Use HTTP OTLP exporters only — gRPC pulls in C bindings that complicate containers and build environments.

```go
// otel.go
func initOTel(ctx context.Context) (func(context.Context) error, error) {
    endpoint := os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
    authHeader := os.Getenv("OTEL_EXPORTER_OTLP_HEADERS")

    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint(urlOnly(endpoint)),
        otlptracehttp.WithHeaders(parseAuthHeader(authHeader)),
    )
    if err != nil {
        return nil, fmt.Errorf("creating trace exporter: %w", err)
    }

    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(serviceName()),
            semconv.ServiceVersion(version()),
            semconv.DeploymentEnvironment(environment()),
            semconv.VCSRepositoryURLFull(repoURL()),
            semconv.VCSRefHeadRevision(commitSHA()),
        ),
    )
    if err != nil {
        return nil, fmt.Errorf("creating resource: %w", err)
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter,
            sdktrace.WithMaxQueueSize(2048),
            sdktrace.WithBatchTimeout(5*time.Second),
            sdktrace.WithMaxExportBatchSize(512),
        ),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
    )
    otel.SetTracerProvider(tp)

    mp := sdkmetric.NewMeterProvider(
        sdkmetric.WithResource(res),
        sdkmetric.WithReader(sdkmetric.NewPeriodicReader(
            otlpmetrichttp.New(ctx,
                otlpmetrichttp.WithEndpoint(urlOnly(endpoint)),
                otlpmetrichttp.WithHeaders(parseAuthHeader(authHeader)),
            ),
            sdkmetric.WithInterval(15*time.Second),
        )),
    )
    otel.SetMeterProvider(mp)

    lp := sdklog.NewLoggerProvider(
        sdklog.WithProcessor(sdklog.NewBatchProcessor(
            otlploghttp.New(ctx,
                otlploghttp.WithEndpoint(urlOnly(endpoint)),
                otlploghttp.WithHeaders(parseAuthHeader(authHeader)),
            ),
        )),
    )
    otel.SetLoggerProvider(lp)

    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return func(ctx context.Context) error {
        var errs []error
        for _, fn := range []func(context.Context) error{tp.Shutdown, mp.Shutdown, lp.Shutdown} {
            if err := fn(ctx); err != nil {
                errs = append(errs, err)
            }
        }
        return errors.Join(errs...)
    }, nil
}

func urlOnly(endpoint string) string {
    u, err := url.Parse(endpoint)
    if err != nil {
        return endpoint
    }
    return u.Host
}

func parseAuthHeader(header string) map[string]string {
    if header == "" {
        return nil
    }
    return map[string]string{"authorization": header}
}
```

Import the `otel.go` package from `init` in `main.go` to guarantee it runs before any instrumented code.

```go
package main

import (
    "context"
    _ "github.com/example/mugline/otel" // bootstrap; blank import keeps it side-effect only
    "github.com/example/mugline/internal/server"
)

func main() {
    ctx := context.Background()
    shutdown, err := initOTel(ctx)
    // ...
    defer shutdown(ctx)
    server.Run(ctx)
}
```

If `OTEL_EXPORTER_OTLP_HEADERS` is absent, log one warning and return without
installing exporters. Leaving the app in no-op mode is better than crashing.

## Instrumented HTTP

Use `otelhttp` for HTTP server and client spans.

```go
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

mux.Handle("/orders", otelhttp.NewMiddleware(
    http.HandlerFunc(handleOrders),
    otelhttp.WithServerName("mugline-api"),
)())

// Outbound HTTP calls also propagate trace context automatically.
client := &http.Client{Transport: otelhttp.NewTransport(http.DefaultTransport)}
```

## Spans for Business Operations

Use `tracer.Start(ctx, "operation.name", ...)` for operations with clear boundaries.

```go
func processOrder(ctx context.Context, orderID, tenantID string) error {
    ctx, span := tracer.Start(ctx, "order.process",
        trace.WithAttributes(
            attribute.String("tenant.id", tenantID),
            attribute.String("order.id", orderID),
        ),
    )
    defer span.End()

    err := validateOrder(ctx, orderID)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(trace.Status{Code: trace.StatusCodeError, Message: err.Error()})
        return err
    }

    span.SetAttributes(attribute.String("outcome", "success"))
    ordersSubmitted.Add(ctx, 1,
        metric.WithAttributes(attribute.String("tenant.id", tenantID)),
    )
    return nil
}
```

Name spans as `domain.verb`: `order.process`, `payment.charge`, `email.send`,
`agent.run`, `job.archive`. Record exceptions with `span.RecordError(err)` and
set `StatusCodeError`. Never put PII in attributes (emails, passwords, tokens,
full request bodies). Do not use detached `tracer.Start(...)` followed by
manual `span.End()` for bounded work — use `defer span.End()` instead.

## Context Propagation

OpenTelemetry Go relies on `context.Context` to carry trace context. Every
instrumented call must receive a context that already carries the active span,
not `context.Background()`. This is Go's equivalent of "parent span":
every function that creates a span receives its parent through `ctx`.

```go
// WRONG: creates a root span with no connection to the request
func getUser(ctx context.Context, userID string) (*User, error) {
    ctx = context.Background() // ← breaks the trace
    ctx, span := tracer.Start(ctx, "user.get")
    defer span.End()
    // ...
}

// RIGHT: propagates the incoming trace context
func getUser(ctx context.Context, userID string) (*User, error) {
    ctx, span := tracer.Start(ctx, "user.get",
        trace.WithAttributes(attribute.String("user.id", userID)),
    )
    defer span.End()
    // ...
}
```

When spawning goroutines, pass `ctx` explicitly. For background workers,
start a root span and pass it or a derived child to each batch item.

```go
// Goroutine inherits parent span via ctx
go func(ctx context.Context) {
    ctx, span := tracer.Start(ctx, "job.process_batch")
    defer span.End()
    processBatch(ctx, items)
}(ctx)
```

## Logs

Use the standard library `log/slog` with an OTel log bridge so log lines carry
`trace_id` and `span_id` automatically.

```go
import (
    "log/slog"
    "go.opentelemetry.io/otel/sdk/log"
    sdklog "go.opentelemetry.io/otel/sdk/log"
    otlploghttp "go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp"
    "go.opentelemetry.io/contrib/sdk/log/slog"
)

loggerProvider := sdklog.NewLoggerProvider(
    sdklog.WithProcessor(sdklog.NewBatchProcessor(
        otlploghttp.New(ctx,
            otlploghttp.WithEndpoint(urlOnly(endpoint)),
            otlploghttp.WithHeaders(parseAuthHeader(authHeader)),
        ),
    )),
)

// Wire slog to the OTel logger provider
handler := slogotel.NewSlogHandler(loggerProvider.Logger(""))
slog.SetDefault(slog.New(handler))

slog.InfoCtx(ctx, "order processed",
    slog.String("tenant_id", tenantID),
    slog.String("order_id", orderID),
)
```

`slog.InfoCtx` automatically injects `trace_id` and `span_id` from the span
stored in `ctx`. Preserve existing `slog` configuration; the OTel bridge
replaces the handler underneath without changing call sites.

Do not emit error logs unless the operation cannot recover and requires manual
intervention.

## Metrics

Three categories to cover:

- **Business counters.** State transitions (created, started, completed, failed, retried).
  Low-cardinality dimensions only — tenant, channel, status. Never user/order IDs.

```go
ordersSubmitted.Add(ctx, 1,
    metric.WithAttributes(
        attribute.String("tenant.id", tenantID),
        attribute.String("outcome", "success"),
    ),
)
```

- **Latency histograms.** Operations the operator cares about. Record duration in
  milliseconds so the histogram is human-legible.

```go
elapsed := time.Since(startedAt).Milliseconds()
orderDuration.Record(ctx, elapsed,
    metric.WithAttributes(attribute.String("tenant.id", tenantID)),
)
```

- **LLM token counters.** If the project calls OpenAI, Anthropic, or Google, record
  token usage per call site. Prefer provider instrumentation (OpenInference in
  contrib) where available so native SDK calls stay readable.

```go
llmTokensInput := meter.Int64Counter("llm.tokens.input",
    metric.WithUnit("tokens"),
)
llmTokensOutput := meter.Int64Counter("llm.tokens.output",
    metric.WithUnit("tokens"),
)

llmTokensInput.Add(ctx, inputTokens,
    metric.WithAttributes(
        attribute.String("tenant.id", tenantID),
        attribute.String("gen_ai.provider.name", "anthropic"),
        attribute.String("gen_ai.request.model", model),
        attribute.String("app.gen_ai.use_case", "voice.initial_greeting"),
        attribute.String("app.gen_ai.call_site", "callMugCopyLLM"),
        attribute.String("outcome", "success"),
    ),
)
```

Do not add `llm.cost_usd` pricing metrics in application handlers. Superlog
computes estimated LLM cost centrally in the UI/query layer from captured
provider/model/token data.

## Resource Attributes

Set these on the OTel resource for every service, via `resource.New` at
initialization:

- `service.name` (required)
- `service.version`
- `deployment.environment.name`
- `vcs.repository.url.full` — the canonical https URL of the repo (e.g.
  `https://github.com/acme/api`). Hardcode this alongside `service.name`; if a
  build env exposes it (e.g. `VERCEL_GIT_REPO_OWNER`+`VERCEL_GIT_REPO_SLUG` on
  Vercel, `RAILWAY_GIT_REPO_OWNER`+`RAILWAY_GIT_REPO_NAME` on Railway), read
  from env so a fork or rename does not drift.
- `vcs.ref.head.revision` — the commit SHA, read on a best-effort basis from
  env: `VERCEL_GIT_COMMIT_SHA`, `RAILWAY_GIT_COMMIT_SHA`, `GITHUB_SHA`,
  `SOURCE_COMMIT`, `HEROKU_SLUG_COMMIT`, etc. Do not shell out to `git`. Omit
  the attribute entirely if no env source is available. Skipping the SHA is
  acceptable; skipping the repo URL is not.

```go
resource.New(ctx,
    resource.WithAttributes(
        semconv.ServiceName(serviceName()),
        semconv.ServiceVersion(version()),
        semconv.DeploymentEnvironment(environment()),
        semconv.VCSRepositoryURLFull(repoURL()),
        semconv.VCSRefHeadRevision(commitSHA()),
    ),
)
```

Use the OTel semantic-convention keys exactly — `vcs.repository.url.full` and
`vcs.ref.head.revision`. Do not invent parallel attributes like `git.repo` or
`app.repo_url`.

## Coexist with Existing Vendors

Do not remove Sentry, Datadog, New Relic, Honeycomb, Logtail, or any other
observability SDK. OTel sits alongside them. The user explicitly wants both
signals flowing during migration; ripping out the incumbent is not your call.

## Naming Conventions

- Files: provider-neutral — `otel.go`, `telemetry.go`, `observability.go`.
  The word "Superlog" belongs only in endpoint/key setup comments or deploy
  instructions.
- Spans: conventional, low-cardinality — `order.process`, `voice.session`,
  `llm.generate_copy`. Prefer semantic product-operation names over transport
  names: `llm.generate_copy` is more useful than `llm.anthropic.messages.create`.
- Metric instruments: `domain.operation.unit` — `orders.submitted`,
  `order.duration_ms`, `llm.tokens.input`.
- Attributes: OTel semantic convention keys (`tenant.id`, `order.id`) plus
  `app.gen_ai.use_case`, `app.gen_ai.call_site`. Do not invent custom keys.

## Env Vars

Use standard OTel env vars:

```text
OTEL_EXPORTER_OTLP_ENDPOINT=https://intake.superlog.sh
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <key>
```

Do not use legacy names like `SUPERLOG_API_KEY` or `SUPERLOG_INTAKE_URL`. Use
neutral placeholders such as `<key from dashboard>` or `SUPERLOG_TEST`.

## Init Behavior

If `OTEL_EXPORTER_OTLP_HEADERS` is absent, log one clear warning and return
without installing exporters. Leaving the app in no-op mode is better than
crashing local dev or silently dropping telemetry.

```go
func initOTel(ctx context.Context) (func(context.Context) error, error) {
    if os.Getenv("OTEL_EXPORTER_OTLP_HEADERS") == "" {
        slog.Warn("otel disabled; OTEL_EXPORTER_OTLP_HEADERS is not set")
        return func(context.Context) error { return nil }, nil
    }
    // ...
}
```

## Smoke Checks

A good smoke path runs the server with OTel env vars present, hits a route that
exercises an instrumented business span, and confirms OTLP POSTs return 2xx.
For a minimal Go smoke:

```bash
go run -ldflags="-X main.commit=$(git rev-parse --short HEAD)" \
    -tags=smoke \
    ./cmd/server &
sleep 2
curl -s http://localhost:8080/health
curl -s http://localhost:8080/orders -X POST \
    -H "Content-Type: application/json" \
    -d '{"tenant_id":"tenant_smoke","order_id":"order_001"}'
kill %1
```

Confirm the telemetry smoke test explicitly exercises a real instrumented
operation, not only a static health route.

## Hard Rules

- Use `go.opentelemetry.io/otel` and its official SDK packages. Do not install
  vendor-wrapped telemetry helpers.
- Acquire tracers and meters at package scope, not per-call.
- Every instrumented function receives `ctx context.Context` and passes it to
  `tracer.Start`. Never substitute `context.Background()`.
- Span names follow `domain.verb` convention. Record exceptions on the active span.
- Metric dimensions are low-cardinality: tenant, channel, status, outcome.
  Never user IDs or order IDs.
- Use HTTP OTLP exporters only.
- Resource attributes include `service.name`, `service.version`,
  `deployment.environment.name`, `vcs.repository.url.full`, and
  `vcs.ref.head.revision`. Skip the SHA if no env source exists; skip the
  repo URL is not.
