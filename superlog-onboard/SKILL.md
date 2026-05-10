---
name: superlog-onboard
description: "Onboard a project to Superlog by installing OpenTelemetry traces, logs, and metrics across every app and service in the repo. Triggers on requests like 'install Superlog', 'set up Superlog', 'add Superlog telemetry', 'onboard this repo to Superlog', 'instrument with OpenTelemetry for Superlog'."
---

# Superlog onboarding

Wire OpenTelemetry traces, logs, and metrics into the user's project so telemetry streams to Superlog. Cover **every app and service in the repo** — not just the one the user is currently sitting in.

Prefer native OpenTelemetry APIs and the framework's documented bootstrap over custom helper layers. If a specific stack stumps you, search the OTel docs for that language; don't guess.

Before editing, read the applicable companion skills:

- `otel-onboarding-style` for general OTel taste.
- `otel-python-style` for Python services.
- `otel-fastapi-style` for FastAPI services.
- `otel-livekit-style` for LiveKit agents.
- `otel-nextjs-style` for Next.js/Vercel apps.
- `otel-expo-style` for Expo / React Native apps.
- `otel-supabase-edge-style` for Supabase Edge Functions.
- `otel-generic-style` for any other language (Go, Java/Kotlin, Ruby, Rust, .NET/C#, PHP, Elixir, plain Node, …) — use this as the fallback when none of the above match.

## Step 0 — Endpoint and key handling

The **OTLP endpoint** is always `https://intake.superlog.sh`.

The **ingest API key** starts with `superlog_live_` and is project-scoped + write-only. It's a secret: **never embed it as a literal in source files.** It only ever goes into env files (`.env.local`, `.env.superlog`, the language equivalent) read at runtime.

Two paths, no questions asked:

### Key in the prompt

If the invoking prompt or env already contains a `superlog_live_…` key, use it. Validate the prefix, write it into env files, never echo it back to the chat. Done — move on to Step 1.

### No key

Kick off the device flow immediately, then keep working in parallel — don't block install on signup.

1. `POST https://api.superlog.sh/oauth/device` with `Content-Type: application/json` and body `{"flow":"skill"}`. Response includes `device_code`, `user_code`, `verification_uri_complete` (a `https://superlog.sh/activate?code=…&flow=skill` URL), `expires_in` (seconds), `interval` (poll interval seconds).
2. Open `verification_uri_complete` in the user's default browser (`open` / `xdg-open` / `start ""`). Print the URL too so they can copy it if the open command silently fails. Tell the user *briefly* what's happening: signup is open in their browser, the key flows back here automatically, you're going to keep working.
3. While the user signs up, **do not block.** Keep going with Steps 1–4. Use the literal sentinel `SUPERLOG_TEST` in env files as a placeholder — Superlog's ingest accepts it from anyone (returns 200 without forwarding anywhere), so the user's app can boot and exercise the OTel bootstrap path while signup is in flight.
4. At Step 5, poll `POST https://api.superlog.sh/oauth/token` with `{"device_code":"…"}` every `interval` seconds. `428` = `authorization_pending`, keep waiting. `200` returns `{ingest_key, project_id, user, org, flow:"skill"}`. `410` = expired (user took longer than `expires_in`).
5. On 200: rewrite every env file you wrote to replace `SUPERLOG_TEST` with the real `ingest_key`. Never print the key back to chat (transcripts get logged). The web `/activate` page already confirms hand-off to the user.
6. On 410 / user closed the tab: leave `SUPERLOG_TEST` in place and tell the user to sign up at https://superlog.sh/ and swap it later.

Sign-up is **not** elicited. Deploy is — see Step 4 / Step 5 below for when to ask the user which target to deploy to.

## Step 1 — Map every app/service in the repo

Before instrumenting anything, enumerate what's here. Check workspace manifests (`pnpm-workspace.yaml`, root `package.json` `workspaces`, `go.work`, Cargo workspace, Python `pyproject.toml` workspace setups, `apps/*` and `services/*` conventions). Identify each service: web frontend, API, workers, background jobs, CLIs, sample/demo apps, mobile apps, Supabase and/or server functions. Mobile and serverless/edge functions are in scope; do not skip them merely because they are client-side or short-lived. Skip pure type/config packages with no runtime entry point. Do not skip any runnable services or leave them "out of scope": instrument absolutely everything in this run; there may be no follow-up.

Show the user the list before you start, so they can correct it.

## Step 2 — For each service, install native OTel and bootstrap

**Use the language's native OpenTelemetry SDK.** Don't reach for vendor wrappers or hand-rolled helpers when an official package exists. Examples of what "native" means here: `@opentelemetry/sdk-node` for Node servers, `@vercel/otel` for normal Next.js/Vercel apps (sdk-node breaks Next's webpack and misses the framework bootstrap), `@opentelemetry/sdk-trace-web` + browser/mobile-compatible exporters for Vite/SPA/Expo, `opentelemetry-instrumentation-*` + `opentelemetry-sdk` for Python, `go.opentelemetry.io/otel` for Go.

No broad wrapper APIs. Avoid reusable helpers like `sendSuperlogSpan`, `recordCounter`, `recordLog`, `startTelemetrySpan`, or `withTelemetry`. Acquire native tracers/meters/loggers at module scope and use the SDK's own APIs directly. In TypeScript/JavaScript, use the published `@superlog/otel-helpers` `withSpan` helper for bounded business spans and add `@superlog/otel-helpers` to `package.json`; this is required when the package can be installed because it avoids expanding a whole function into `startActiveSpan` plus `try` / `catch` / `finally`. Do not use helpers around provider SDK calls that OpenInference/provider instrumentation can observe directly. If an edge runtime genuinely cannot load an upstream OTel SDK, keep the shim tiny, provider-neutral, and OTel-shaped: `tracer.startActiveSpan`, `span.setAttributes`, `SpanStatusCode`, `meter.createCounter`, `histogram.record`.

Wire all three signals — traces, logs, metrics. **Logs go through OTLP, not just stdout** — set up the OTel log bridge for the language so app logs (with their existing log levels and structured fields) carry the active `trace_id` / `span_id` automatically. The user's existing logger keeps working; you're just adding an OTLP handler/processor underneath.

Bootstrap rules:
- The bootstrap file must run before any framework imports. Use the language/framework's documented hook (`--import` flag, `instrumentation.ts`, top-of-`main.py` import, etc.).
- Read the endpoint and key from env. If the auth header is missing, log one clear warning and leave the app running in no-op/disabled mode rather than silently dropping telemetry or crashing local dev.
- Use HTTP OTLP exporters, not gRPC. gRPC pulls in native bindings that break bundlers and complicate containers.
- Use the project's existing package manager (detect via lockfile).
- Prefer idempotent edits. If a config file already exists, edit don't overwrite.
- Set resource attributes on the OTel resource for every service: `service.name`, `service.version`, `deployment.environment.name`, and **`vcs.repository.url.full`** — the canonical https URL of the repo (e.g. `https://github.com/acme/api`). The repo URL is the important one and is fine to hardcode alongside `service.name` in the SDK init; if the build platform exposes the slug (Vercel `VERCEL_GIT_REPO_OWNER`/`VERCEL_GIT_REPO_SLUG`, Railway `RAILWAY_GIT_REPO_OWNER`/`RAILWAY_GIT_REPO_NAME`), prefer reading from env. Also set **`vcs.ref.head.revision`** (commit SHA) on a best-effort basis from whatever env the runtime already injects (`VERCEL_GIT_COMMIT_SHA`, `RAILWAY_GIT_COMMIT_SHA`, `GITHUB_SHA`, `SOURCE_COMMIT`, `GIT_COMMIT`, `HEROKU_SLUG_COMMIT`, …). Do not shell out to `git` from the running process. Skipping the SHA is fine, skipping the URL is not. Use the OTel semantic-convention keys exactly — do not invent `git.repo` / `app.repo_url`.

Framework rules:
- **Next.js/Vercel:** use `instrumentation.ts` with `@vercel/otel` `registerOTel(...)` as the bootstrap. Do not substitute a raw `@opentelemetry/sdk-node` / `NodeSDK` bootstrap unless the repo already uses that architecture and you are extending it. Use `@opentelemetry/api` tracers/meters inside route handlers only where auto-instrumentation is blind.
- **Expo/React Native:** preserve existing Expo Go / unsupported-runtime guards. In supported builds, call `initObservability()` before Sentry and before app registration/user code. Use public env vars like `EXPO_PUBLIC_OTEL_EXPORTER_OTLP_ENDPOINT`, `EXPO_PUBLIC_OTEL_EXPORTER_OTLP_HEADERS`, and `EXPO_PUBLIC_OTEL_SERVICE_NAME`.
- **Supabase Edge Functions:** native Deno OpenTelemetry does not work in hosted Supabase Edge today. Use the tiny OTel-shaped shim pattern above; keep exporter endpoint/headers in one setup area and avoid Superlog-specific function/file names.
- **Python/FastAPI:** use native instrumentation such as `FastAPIInstrumentor.instrument_app(app)` rather than replacing request handling with manual middleware.
- **Python/LiveKit:** lifecycle spans that cross shutdown callbacks may use `start_span` + `trace.use_span(..., end_on_exit=False)` and end in the shutdown callback. Bounded work should still use decorators or context managers.

**Coexist with existing observability vendors. Don't remove Sentry, Datadog, New Relic, Honeycomb, Logtail, Pino transports, etc.** OTel sits alongside them. The user explicitly wants both signals flowing during migration; ripping out the incumbent is not your call.

## Step 3 — Add custom spans, metrics, and logs around business operations

Auto-instrumentation gets you HTTP in/out, DB queries, framework lifecycle. That's the floor, not the ceiling. Read the project to find the operations a human operator would actually want to see when something looks wrong.

### Traces

Wrap **every critical business operation** with an active span. Auto-instrumented spans are fine where they exist — but if an operation isn't already getting a span, add one.

- Naming: `domain.verb` (`order.process`, `payment.charge`, `email.send`, `agent.run`, `interview.create`, `job.<type>`).
- Attributes: entity IDs (order.id, user.id, workspace.id, tenant.id), counts, key boolean branch outcomes, model name / provider for LLM calls.
- Record exceptions: `span.recordException(err)` + `span.setStatus({ code: ERROR })` on failure paths.
- For Python functions with clear boundaries, prefer `@tracer.start_as_current_span("operation.name")`. Use a context manager when a decorator does not fit. Do not use detached `start_span()` + manual `end()` for bounded work.
- Skip trivial getters, pure transforms, internal helpers — anything with no real latency or failure mode.
- **Never put PII in attributes** (emails, passwords, tokens, full request bodies).

### Logs

Make sure logs are **structured and carry operation context**. Concretely: every log line emitted inside a span should arrive at Superlog with `trace_id` / `span_id` populated and any structured fields (orderId, userId, etc.) preserved as attributes. Trace/span context may be added natively by the log bridge or integration, or may require additional work.

Use logs for narrative ("starting batch reconcile", "retrying after 3xx") and exceptional events. An error log must only be emitted if the operation cannot recover and manual intervention is required.

### Metrics

Cover **business + performance + cost**. Three categories to look for:

- **Business logic counters.** Every meaningful state transition: created, started, completed, failed, retried. Per-tenant, per-channel, per-status — low-cardinality dimensions only (never user/order IDs).
- **Performance histograms.** Latency of operations the user cares about, queue depth, batch sizes, payload sizes. Reuse existing timing instrumentation if the project already has any (`time.perf_counter` blocks, custom `LatencyTracker`s, "[TIMING]" log lines) — emit a histogram from those measurements rather than measuring twice.
- **Costs — especially LLM costs.** If the project calls OpenAI / Anthropic / Google / any LLM provider, prefer provider instrumentation such as OpenInference where available so native SDK calls stay readable. Do not add pricing constants or LLM cost math in product handlers; Superlog computes estimated cost centrally in the UI/query layer from captured provider/model/token attributes. Avoid duplicating token counters already captured by provider instrumentation.

Get the meter once at module level, create instruments at module level, increment in the hot path. Don't create a fresh meter per call.

## Step 4 — Verify the app still works

Per service:

1. **Run the project's own dev or build command** (whatever its `package.json` / `pyproject` / `Makefile` already wires up). Confirm it starts cleanly with no errors that trace back to your OTel install. Also run a telemetry bootstrap smoke that imports or starts the app with the OTel env vars present, so provider setup, exporter construction, log bridging, and framework instrumentation all initialize. For a Python server this can be an import/startup command such as `uv run python -c 'from app.main import app; print(app.title)'`; for Node/Next use the repo's build/start path. For a server, hit at least one route with curl so traffic flows through the instrumentation; choose a route that exercises an instrumented operation when practical, not only a static health route. For a CLI, invoke a real command. **Don't ship if the app's own startup is now broken** — that's a regression.
2. **Confirm telemetry leaves the process.** With `SUPERLOG_TEST` (or a real key) in env, OTLP POSTs from the dev server should return 2xx — that proves the bootstrap is reaching the network. As an extra sanity check, you can `curl -X POST https://intake.superlog.sh/v1/traces -H "authorization: Bearer SUPERLOG_TEST" -H "content-type: application/json" -d '{}'` and confirm 200. The real signal is the running app's own POSTs succeeding by the time the dev server shuts down — if they don't, the bootstrap is wrong (most often: SDK loaded too late, or shutdown not flushing).

A bootstrap that loads but never POSTs is not a partial success. Fix it before moving on.

## Step 5 — Hand-off (final message to the user)

If you started a device flow in Step 0, collect the key first. Print one line that you're waiting for sign-up to finish (so the user knows the terminal isn't frozen), then poll `POST https://api.superlog.sh/oauth/token` with `{"device_code":"…"}` at the `interval` returned earlier. Cap the wait at `expires_in` (default 600s).

- On 200: rewrite every env file you wrote, replacing the `SUPERLOG_TEST` placeholder with the real `ingest_key`. Touch `.env.local`, `.env.superlog`, `.env.production`, etc. — whatever you created or edited. Never print the `ingest_key` back to chat (transcripts get logged); the web `/activate` page already confirms hand-off to the user.
- On 410 / poll timeout: leave `SUPERLOG_TEST` in env, tell the user "sign-up didn't finish in time — sign up at https://superlog.sh/ when you're ready and swap the key." Continue with the rest of the closing message.

If the key was already supplied in the prompt, no polling needed — it's been in env from the start.

### What changed

3–7 short factual bullets covering: packages installed, files created/modified, env vars written, business spans/metrics added. Per service if changes differed, grouped if uniform. Mention any existing observability vendor (Sentry, Datadog, Logtail, Pino transports, etc.) you intentionally left in place so the coexistence is explicit.

### Set these env vars to launch

Two env vars need to reach every runtime that runs the instrumented code (local dev, CI, prod):

- `OTEL_EXPORTER_OTLP_ENDPOINT=https://intake.superlog.sh`
- `OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer <key>` — `<key>` is the real `superlog_live_…` once pairing/signup completed; `SUPERLOG_TEST` if it didn't (user can swap later).

For local dev, point at the `.env.local` / `.env.superlog` file you wrote — already done. For deploy, give **concrete instructions for the targets you actually detected in the repo**:

- `Dockerfile` / `docker-compose.yml` → `docker run -e OTEL_EXPORTER_OTLP_ENDPOINT=… -e OTEL_EXPORTER_OTLP_HEADERS=…` (or service-level `environment:` in compose). Mention multi-stage builds shouldn't bake the key in.
- `vercel.json` / `.vercel/` → `vercel env add OTEL_EXPORTER_OTLP_HEADERS production` (and same for `OTEL_EXPORTER_OTLP_ENDPOINT`).
- `railway.json` / Railway already wired → `railway variables --set 'OTEL_EXPORTER_OTLP_HEADERS=…' --service <name>`.
- `fly.toml` → `fly secrets set OTEL_EXPORTER_OTLP_HEADERS=…`.
- `.github/workflows/*.yml` → repo Settings → Secrets → add as Actions secrets, then reference as `${{ secrets.OTEL_EXPORTER_OTLP_HEADERS }}` in the workflow.
- `Procfile` / Heroku → `heroku config:set OTEL_EXPORTER_OTLP_HEADERS=…`.
- Kubernetes manifests → add to a Secret and mount via `envFrom` / `valueFrom.secretKeyRef`.
- Otherwise → the generic shell-export form for whatever shell is in use.

Only mention targets that actually exist in the repo. Don't list every option as a wall of text.

### Deploy + finish setup

Once env vars are wired into the deploy target, the user deploys (or runs locally) so first events start flowing. Offer to run the platform CLI for them when one is detected and likely authed (`vercel`, `railway`, `flyctl`, `gh secret set` for Actions). When in doubt, ask before running — deploy commands are observable and not always reversible.

**This is the place to elicit a choice.** If the repo has signals for *multiple* deploy targets (e.g. `Dockerfile` + `vercel.json`), or *no* clear target (no platform config + no Procfile + no infra dir), ask the user which one to wire up rather than guessing. One question, one answer:

> "I see signals for `<X>` and `<Y>` here — which is this project actually deployed to?" (or, if none: "How do you deploy this — Vercel, Railway, Fly, Docker, something else?")

Then write the env vars to that target (or those targets) and stop.

Then point the user at **https://superlog.sh/**. The first time they land there, Superlog's onboarding wizard takes over: it polls for the first events, then walks them through connecting GitHub (so the fix-agent can open PRs) and Slack (so investigations can ping a channel). The skill does **not** drive these integrations — the wizard does, and it's already designed for that.

Tell the user, in one line: "Deploy your app, then open https://superlog.sh/ — the dashboard will detect your first events and walk you through GitHub + Slack from there."

Open the dashboard for them:

- macOS: `open "https://superlog.sh/"`
- Linux: `xdg-open "https://superlog.sh/"`
- Windows: `start "" "https://superlog.sh/"`

If `open` / `xdg-open` isn't available or returns non-zero, just print the URL — don't fail the skill.

### Suggest the Superlog MCP for next time

Before ending the skill, suggest installing the Superlog MCP server so the agent (Claude Code, Codex, Cursor, etc.) can query their telemetry directly the next time they're debugging — search logs, pull traces, check error rates from inside the chat without context-switching to the dashboard.

For Claude Code (which is what the user is running this skill in, by far the most likely case):

```
claude mcp add --transport http superlog https://api.superlog.sh/mcp
```

Mention but don't run the equivalents for other agents in case they use more than one:

- Codex: `codex mcp add superlog --url https://api.superlog.sh/mcp && codex mcp login superlog`
- Cursor / others: copy the `mcpServers` snippet from https://superlog.sh/ → Connect.

Frame this as a suggestion, not a step you execute. The user has to opt in to MCP servers and may not want one yet.

## Hard rules

- Never modify files outside the project root.
- Never commit, push, or open PRs.
- Never embed the ingest key as a literal in source — env files only.
- Never remove an existing observability vendor unless the user asks for it.
- Use the project's existing package manager and existing logger.
- Prefer native OTel packages for the language; don't reinvent telemetry plumbing the SDK already provides.
- If the dev/build command errors out *because of* your instrumentation, that's a failure — fix it or report it, don't paper over it.
