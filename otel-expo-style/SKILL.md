---
name: otel-expo-style
description: "Expo / React Native OpenTelemetry style: bootstrap guards, init ordering, inline endpoint + ingest key, mobile-compatible exporters, and product action spans."
---

# OTel Expo Style

Preserve existing runtime guards. If the app avoids native SDKs in Expo Go or an
unsupported runtime branch, initialize OTel only inside the supported branch.

```ts
const isExpoGo = process.env.EXPO_PUBLIC_RUNTIME === "expo-go";

if (!isExpoGo) {
  initObservability();
  Sentry.init({ dsn: process.env.EXPO_PUBLIC_SENTRY_DSN });
}

registerRootComponent(App);
```

In supported builds, call `initObservability()` before Sentry and before app
registration/user code.

## SDK Shape

Use native OTel JS APIs and browser/mobile-compatible providers/exporters. Do
not create `recordCounter`, `sendSuperlogSpan`, or tracking-specific helper
files.

```ts
export const tracer = trace.getTracer("mugline.mobile");
export const meter = metrics.getMeter("mugline.mobile");
export const ordersSubmitted = meter.createCounter("mug.orders.submitted");
```

Add automatic fetch/XHR instrumentation when compatible so API calls get spans
without wrapping application code.

## Product Actions

Wrap low-risk business operations with native active spans.

```ts
await tracer.startActiveSpan("mug.order.submit", async (span) => {
  try {
    span.setAttribute("tenant.id", tenantId);
    ordersSubmitted.add(1, { "tenant.id": tenantId, outcome: "success" });
  } catch (error) {
    span.recordException(error as Error);
    span.setStatus({ code: SpanStatusCode.ERROR });
    throw error;
  } finally {
    span.end();
  }
});
```

## Configuration

Inline the endpoint and ingest key directly in the observability module. The
key is project-scoped + write-only (Sentry-DSN shaped), so source-level
configuration is the right default for Expo — and it sidesteps the
build-time-vs-runtime quirks of `EXPO_PUBLIC_*` env vars in different
build profiles.

```ts
const SUPERLOG_ENDPOINT = "https://intake.superlog.sh";
const SUPERLOG_KEY = "superlog_live_…"; // set by superlog-onboard skill on pairing
const SERVICE_NAME = "mugline-mobile";

const exporter = new OTLPTraceExporter({
  url: `${SUPERLOG_ENDPOINT}/v1/traces`,
  headers: { authorization: `Bearer ${SUPERLOG_KEY}` },
});
```
