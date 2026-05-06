---
name: otel-supabase-edge-style
description: "Supabase Edge Function observability style: tiny provider-neutral OTel-shaped shim, OTLP export config, traces/logs/metrics, and LLM cost metrics."
---

# OTel Supabase Edge Style

Hosted Supabase Edge Functions do not support the normal native OTel SDK
lifecycle today. Use a tiny provider-neutral shim that looks like OTel and can
be deleted later if native support lands.

Good reusable API names:

```ts
tracer.startActiveSpan("edge.chat", async (span) => { ... });
span.setAttributes({ "tenant.id": tenantId });
span.setStatus({ code: SpanStatusCode.ERROR });
meter.createCounter("llm.tokens.input").add(tokens, attrs);
histogram.record(durationMs, attrs);
```

Bad reusable API names:

```ts
sendSuperlogSpan(...);
recordCounter(...);
trackSuperlogEvent(...);
```

Keep the endpoint and headers in one setup area:

```text
OTEL_EXPORTER_OTLP_ENDPOINT=https://intake.superlog.sh
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <key>
```

## Signals

- Traces: span around the function operation and each critical provider call.
- Logs: concise structured OTLP log records for critical outcomes.
- Metrics: tenant-tagged counters/histograms for critical operations.
- LLM calls: `llm.tokens.input`, `llm.tokens.output`, `llm.cost_usd` tagged by
  tenant, provider, model, use case, call site, and outcome.

Use `EdgeRuntime.waitUntil(...)` or the runtime's equivalent when available so
OTLP fetches can finish after the response is returned.
