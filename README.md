# Superlog Skills

Agent skills for onboarding projects to Superlog with native OpenTelemetry traces, logs, and metrics.

## Install

```sh
npx skills add superloglabs/skills --all
```

Then ask your agent: **"install Superlog in this project"**.

Install just the main onboarding skill (companion `otel-*-style` skills won't be available):

```sh
npx skills add superloglabs/skills --skill superlog-onboard
```

## Skills

- `superlog-onboard` - install Superlog observability across every runnable app and service in a repo.
- `otel-onboarding-style` - general OpenTelemetry style guidance.
- `otel-generic-style` - fallback style guidance for languages without a dedicated skill (Go, Java, Ruby, Rust, .NET, PHP, Elixir, plain Node, etc.).
- `otel-python-style` - Python OpenTelemetry style guidance.
- `otel-fastapi-style` - FastAPI instrumentation guidance.
- `otel-nextjs-style` - Next.js and Vercel OpenTelemetry guidance.
- `otel-livekit-style` - LiveKit Agents lifecycle, metrics, and LLM telemetry guidance.
- `otel-expo-style` - Expo and React Native OpenTelemetry guidance.
- `otel-supabase-edge-style` - Supabase Edge Function observability guidance.
- `otel-go-style` - Go OpenTelemetry style guidance.

