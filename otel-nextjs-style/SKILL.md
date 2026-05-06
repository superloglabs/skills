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

## Route Handlers

Use native OTel APIs where auto-instrumentation is blind.

```ts
const tracer = trace.getTracer("mugline.web");
const meter = metrics.getMeter("mugline.web");
const requests = meter.createCounter("mug.copy.generated");

export async function POST(request: Request) {
  const tenantId = request.headers.get("x-tenant-id") ?? "tenant_demo";
  return await tracer.startActiveSpan("mug.copy.generate", async (span) => {
    try {
      span.setAttribute("tenant.id", tenantId);
      requests.add(1, { "tenant.id": tenantId, outcome: "success" });
      return Response.json({ ok: true });
    } catch (error) {
      span.recordException(error as Error);
      span.setAttribute("outcome", "error");
      throw error;
    } finally {
      span.end();
    }
  });
}
```

If a route has an LLM call, use a child span for the LLM operation and LLM token
/ cost counters tagged by tenant, provider, model, use case, call site, and
outcome.

## Env And Smoke

Document server-side env vars:

```text
OTEL_EXPORTER_OTLP_ENDPOINT=https://intake.superlog.sh
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <key>
OTEL_SERVICE_NAME=mugline-web
```

Smoke checks should use tools already in the repo, e.g. `npm run typecheck` or
`npm run build`. Do not assume `ts-node` exists unless it is already installed.
