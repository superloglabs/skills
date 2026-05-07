---
name: otel-onboarding-style
description: "General OpenTelemetry onboarding style for Superlog managed agents: native APIs, signal quality, env vars, LLM metrics, and smoke checks."
---

# OTel Onboarding Style

Use native OpenTelemetry APIs. Do not invent helper APIs.

In TypeScript/JavaScript, use the published `@superlog/otel-helpers` `withSpan`
helper for bounded business spans and add `@superlog/otel-helpers` to
`package.json` when it is not already present. This is required when the package
can be installed. `withSpan` is the intended replacement for expanding a whole
function into `tracer.startActiveSpan(...)` plus `try` / `catch` / `finally`. Do
not use helpers to wrap provider SDK calls that OpenInference/provider
instrumentation can observe directly.

Do:

```ts
const tracer = trace.getTracer("mugline.api");
const meter = metrics.getMeter("mugline.api");
const ordersSubmitted = meter.createCounter("orders.submitted");

await withSpan("order.submit", async (span) => {
  span.setAttributes({
    "tenant.id": tenantId,
    "order.id": orderId,
    outcome: "success",
  });
  ordersSubmitted.add(1, { "tenant.id": tenantId, outcome: "success" });
}, { tracer });
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

Do not use legacy names such as `SUPERLOG_API_KEY` or `SUPERLOG_INTAKE_URL`,
even as placeholder text in docs or comments. Use neutral placeholders such as
`<key from dashboard>` or `SUPERLOG_TEST`.

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

If the app uses LLMs, first look for provider instrumentation that already
captures model/provider/token/error spans. In JavaScript/TypeScript, prefer
OpenInference packages such as
`@arizeai/openinference-instrumentation-anthropic` for supported SDKs. Keep the
real provider call native and readable.

```ts
const response = await client.messages.create({
  model,
  max_tokens: 100,
  messages,
});
```

Every provider/call site still needs enough telemetry to answer usage questions.
Let provider instrumentation own model/provider/token spans where it supports
them. Do not duplicate those attributes at every application call site, and do
not put provider pricing tables or cost math in product handlers. Superlog
computes estimated LLM cost centrally in the UI/query layer from captured
provider/model/token data.

```ts
llmInputTokens.add(inputTokens, {
  "tenant.id": tenantId,
  "gen_ai.provider.name": "anthropic",
  "gen_ai.request.model": model,
  "app.gen_ai.use_case": "voice.initial_greeting",
  "app.gen_ai.call_site": "_callMugCopyLlm",
  outcome: "success",
});
```

Use counters for additive totals only when provider instrumentation cannot
capture token usage:

- `llm.tokens.input`
- `llm.tokens.output`

Token counters use `unit="tokens"` or the SDK equivalent. If
OpenInference/provider instrumentation already captures token usage, do not
duplicate token counters just to mirror it. Do not add `llm.cost_usd` or
equivalent app-side cost metrics for normal LLM calls; cost belongs in Superlog's
central pricing layer.

Use histograms for latency/duration distributions.

Prefer current `gen_ai.*` semantic-convention-style attribute names for LLM
provider/model/token attributes, plus `app.gen_ai.*` for bounded application
dimensions such as use case and call site. Avoid inventing parallel `llm.*`
attributes unless the repo already standardizes on them.

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
