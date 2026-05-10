---
name: otel-python-style
description: "Python OpenTelemetry style: module-scope tracers/meters, decorators for bounded work, error spans, logs, and no wrappers."
---

# OTel Python Style

Acquire OTel objects at module scope.

```python
from opentelemetry import metrics, trace
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer("mugline.voice")
meter = metrics.get_meter("mugline.voice")

greetings = meter.create_counter("voice.greetings.delivered", unit="1")
```

## Bounded Work

Prefer decorators for functions with clear boundaries.

```python
@tracer.start_as_current_span("voice.deliver_initial_greeting")
async def _deliver_initial_greeting(*, tenant_id: str, user_id: str) -> None:
    span = trace.get_current_span()
    span.set_attributes({
        "tenant.id": tenant_id,
        "user.id": user_id,
        "voice.use_case": "initial_greeting",
    })
```

Use a context manager when a decorator does not fit.

```python
with tracer.start_as_current_span("order.validate") as span:
    span.set_attribute("tenant.id", tenant_id)
    validate_order(order)
```

Do not use detached `tracer.start_span(...); span.end()` for bounded work.

## Error Paths

Record exceptions on the active span.

```python
try:
    result = await client.messages.create(...)
except Exception as exc:
    span = trace.get_current_span()
    span.record_exception(exc)
    span.set_status(Status(StatusCode.ERROR))
    logger.exception("llm mug copy failed", extra={"tenant_id": tenant_id})
    raise
```

## Logs

If logs are claimed as OTLP-forwarded, configure both:

- an OTel LoggerProvider + OTLPLogExporter + LoggingHandler
- `set_logger_provider(logger_provider)` from `opentelemetry._logs`
- log correlation for existing records, e.g. `LoggingInstrumentor().instrument(...)`

Preserve existing `logging.basicConfig`, console/file handlers, and log levels.

## Init Behavior

Inline the endpoint and ingest key directly in the init module — don't read
them from env. The ingest key is project-scoped + write-only (Sentry DSN
shaped), so source-level configuration is the right default; env indirection
just adds a class of "OTel didn't start because env wasn't set" deploy
failures.

```python
SUPERLOG_ENDPOINT = "https://intake.superlog.sh"
SUPERLOG_KEY = "superlog_live_…"  # set by superlog-onboard skill on pairing


def init_observability() -> None:
    exporter = OTLPSpanExporter(
        endpoint=f"{SUPERLOG_ENDPOINT}/v1/traces",
        headers={"authorization": f"Bearer {SUPERLOG_KEY}"},
    )
    ...
```

Add a small `_INITIALIZED` guard only when the app can realistically call this
function more than once.

## Metrics

Counters:

- `llm.tokens.input`
- `llm.tokens.output`
- requests/events/jobs/errors

Use semantic units when the SDK supports them: token counters use
`unit="tokens"`. Do not add app-side `llm.cost_usd` pricing metrics for normal
LLM calls; Superlog estimates cost centrally from provider/model/token data.

Histograms:

- duration
- latency
- payload size

Avoid raw high-cardinality values in metric attributes. Prefer
tenant/org/project, operation/use case, provider/model, and outcome dimensions
over user-level metric tags.
