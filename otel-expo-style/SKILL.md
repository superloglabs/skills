---
name: otel-expo-style
description: "Expo / React Native OpenTelemetry style: bootstrap guards, init ordering, public env vars, mobile-compatible exporters, and product action spans."
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

## Env

Client bundles need public build-time vars:

```text
EXPO_PUBLIC_OTEL_EXPORTER_OTLP_ENDPOINT=https://intake.superlog.sh
EXPO_PUBLIC_OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <key>
EXPO_PUBLIC_OTEL_SERVICE_NAME=mugline-mobile
```
