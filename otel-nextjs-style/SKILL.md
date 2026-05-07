---
name: otel-nextjs-style
description: "Next.js/Vercel OpenTelemetry style: instrumentation.ts, @vercel/otel bootstrap, native @opentelemetry/api call sites, env docs, and no raw NodeSDK replacement."
---

# OTel Next.js Style

For Next.js apps, prefer the framework entrypoint.

```ts
// instrumentation.ts
import { registerOTel } from "@vercel/otel";

export function register() {
  registerOTel({
    serviceName: process.env.OTEL_SERVICE_NAME ?? "mugline-web",
  });
}
```

Do not replace this with a custom `NodeSDK` bootstrap unless the repo is not a
normal Next/Vercel app or already has a custom provider that must be extended.

For JavaScript/TypeScript LLM providers, prefer provider instrumentation over
manual child spans. For Anthropic, add OpenInference in the same bootstrap and
keep call sites native:

```ts
import Anthropic from "@anthropic-ai/sdk";
import { AnthropicInstrumentation } from "@arizeai/openinference-instrumentation-anthropic";
import { OTLPLogExporter } from "@opentelemetry/exporter-logs-otlp-http";
import { BatchLogRecordProcessor } from "@opentelemetry/sdk-logs";
import { registerOTel } from "@vercel/otel";

const anthropicInstrumentation = new AnthropicInstrumentation({
  traceConfig: {
    hideInputs: true,
    hideOutputs: true,
  },
});

anthropicInstrumentation.manuallyInstrument(Anthropic);

export function register() {
  registerOTel({
    serviceName: process.env.OTEL_SERVICE_NAME ?? "mugline-web",
    instrumentations: [anthropicInstrumentation],
    logRecordProcessor: new BatchLogRecordProcessor(new OTLPLogExporter()),
  });
}
```

## Route Handlers

Use native OTel APIs where auto-instrumentation is blind.

```ts
import { withSpan } from "@superlog/otel-helpers";

const tracer = trace.getTracer("mugline.web");
const meter = metrics.getMeter("mugline.web");
const requests = meter.createCounter("mug.copy.generated");

export async function POST(request: Request) {
  const tenantId = request.headers.get("x-tenant-id") ?? "tenant_demo";
  return await withSpan("mug.copy.generate", async (span) => {
    span.setAttribute("tenant.id", tenantId);
    requests.add(1, { "tenant.id": tenantId, outcome: "success" });
    return Response.json({ ok: true });
  }, { tracer });
}
```

For TypeScript route handlers, use `@superlog/otel-helpers` `withSpan` for
bounded business spans and add `@superlog/otel-helpers` to `package.json` when it
is not already present. This is required when the package can be installed. It
keeps span lifecycle/error handling out of the handler body and avoids a large
indentation diff. Do not expand the whole route into
`tracer.startActiveSpan(...)` plus `try` / `catch` / `finally` unless the helper
cannot be added or the span has a true cross-callback lifecycle.

If a route has an LLM call and OpenInference/provider instrumentation supports
that SDK, do not wrap the provider call. Leave `client.messages.create(...)` /
equivalent in place and put business context on the active product span or
structured log. Only add manual LLM token/cost metrics for gaps the provider
instrumentation does not cover.

For `@vercel/otel@1.x`, the logs option is `logRecordProcessor` (singular).
Do not use `logRecordProcessors` unless the installed type definitions prove
that version supports it.

`console.info` is not OTLP log export. If there is no existing logger bridge,
use `@opentelemetry/api-logs` for production log records:

```ts
logger.emit({
  severityNumber: SeverityNumber.INFO,
  severityText: "INFO",
  body: "generated mug copy",
  attributes: {
    "tenant.id": tenantId,
    "gen_ai.provider.name": "anthropic",
    "gen_ai.request.model": model,
    "app.gen_ai.use_case": "web.mug_copy",
    outcome: "success",
  },
});
```

## Env And Smoke

Document server-side env vars:

```text
OTEL_EXPORTER_OTLP_ENDPOINT=https://intake.superlog.sh
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <key>
OTEL_SERVICE_NAME=mugline-web
```

Smoke checks should use tools already in the repo, e.g. `npm run typecheck` or
`npm run build`. Do not assume `ts-node` exists unless it is already installed.
