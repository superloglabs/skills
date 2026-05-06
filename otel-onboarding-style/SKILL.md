---
name: otel-onboarding-style
description: "General OpenTelemetry onboarding style for Superlog managed agents: native APIs, signal quality, env vars, LLM metrics, and smoke checks."
---

# OTel Onboarding Style

Use native OpenTelemetry APIs. Do not invent helper APIs.

Do:

```ts
const tracer = trace.getTracer("mugline.api");
const meter = metrics.getMeter("mugline.api");
const ordersSubmitted = meter.createCounter("orders.submitted");

await tracer.startActiveSpan("order.submit", async (span) => {
  span.setAttributes({
    "tenant.id": tenantId,
    "order.id": orderId,
    outcome: "success",
  });
  ordersSubmitted.add(1, { "tenant.id": tenantId, outcome: "success" });
  span.end();
});
```

Do not:

```ts
await sendSuperlogSpan(...);
recordCounter(...);
withTelemetry(...);
```

## Naming

- Files/functions are provider-neutral: `telemetry.ts`, `observability.ts`,
  `initTelemetry()`, `initObservability()`.
- The word Superlog belongs only in endpoint/key setup comments or PR
  instructions.
- Span names are conventional and low-cardinality: `checkout.process`,
  `voice.session`, `llm.generate_copy`.
- Prefer semantic product-operation span names over provider transport names.
  `llm.generate_copy` or `llm.voice_response` is usually more useful than
  `llm.anthropic.messages.create`.

## Env Vars

Use standard OTel env vars:

```text
OTEL_EXPORTER_OTLP_ENDPOINT=https://intake.superlog.sh
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <key>
```

Use public equivalents only for client bundles, e.g.
`EXPO_PUBLIC_OTEL_EXPORTER_OTLP_ENDPOINT`.

Using standard SDK env-var exporter configuration is fine when it sends traces,
metrics, and logs to the configured backend. Explicit per-signal exporter URLs
are also fine when the runtime needs them.

If required OTLP credentials are absent, make that visible at startup: log one
clear warning and skip exporter setup instead of failing later or silently
dropshipping unauthenticated telemetry.

If the repo can call telemetry init from multiple paths, guard provider/exporter
setup so repeated imports, tests, reloads, or framework callbacks do not install
duplicate processors or log handlers. For a single-entrypoint app that starts
cleanly, keep this simple.

Include standard resource attributes when values are available:
`service.name`, `service.version`, `deployment.environment.name`.

## Signals

- Traces: all critical operations have spans with relevant attributes.
- Logs: structured, concise, OTLP-forwarded, and trace/span-correlated.
- Metrics: critical operations have low-cardinality counters/histograms.
- Tenant/org/project information is included where available.
- Do not put raw user ids or request ids in metric tags unless the repo already
  treats them as bounded tenant-like ids.

## LLM Metrics

If the app uses LLMs, every provider/call site needs token and cost metrics.

```ts
llmInputTokens.add(inputTokens, {
  "tenant.id": tenantId,
  "llm.provider": "anthropic",
  "llm.model": model,
  "llm.use_case": "voice.initial_greeting",
  "llm.call_site": "_callMugCopyLlm",
  outcome: "success",
});
```

Use counters for additive totals:

- `llm.tokens.input`
- `llm.tokens.output`
- `llm.cost_usd`

Token counters use `unit="tokens"` or the SDK equivalent. Cost counters use
`unit="USD"`.

Use histograms for latency/duration distributions.

If the app has OpenAI, Anthropic, and Google callers, instrument all three.

## Smoke Checks

Add a durable smoke path when the repo has a natural place for it: README,
TESTING guide, script, npm command, pytest, or checked-in command note.

The smoke should explicitly prove startup/import with OTel env vars present so
provider setup, exporter construction, log bridging, and framework
instrumentation initialize without errors. Then, where practical, exercise an
actual instrumented span/log/metric or OTLP export attempt. A generic health
route only proves the server responds; prefer an operation that crosses the
instrumentation you added.
