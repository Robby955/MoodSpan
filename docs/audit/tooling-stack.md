# Tooling Stack Additions (April 2026)

## Install Order (strict)
1. **Sentry** — Errors, perf tracing, session replay. `npx @sentry/wizard@latest -i nextjs`
2. **PostHog** — Analytics, feature flags, experiments, surveys. `npm install posthog-js posthog-node`
3. **Langfuse** — LLM traces, evals, prompt management, cost tracking. `npm install @langfuse/tracing @langfuse/otel @opentelemetry/sdk-node @langfuse/client`
4. **Trigger.dev** — Long-running TypeScript jobs (nightly evals, failure clustering, canary reports). `npx trigger.dev@latest init`

## Later Additions
- **Label Studio or Argilla** — When human review starts seriously. Label Studio for quick A/B comparison, Argilla for longer-term dataset/feedback pipeline with clinician review
- **LiteLLM or Helicone** — Only when provider routing complexity grows (2+ providers in production)
- **Playwright in CI** — Crisis-flow tests, answer-render tests, citation-click tests, tool-required UI regressions

## Do NOT Add Yet
- LangChain/LangGraph (Kira is simple enough for hand-rolled control)
- Separate vector DB (Postgres is fine at current scale)
- Fine-tuning infrastructure (no strong human-labeled evals yet)
- Memory stack (fix grounding/verifier first)

## Environment Variables
```
# Sentry
NEXT_PUBLIC_SENTRY_DSN=...
SENTRY_AUTH_TOKEN=...
SENTRY_ORG=...
SENTRY_PROJECT=...

# PostHog
NEXT_PUBLIC_POSTHOG_KEY=phc_xxx
NEXT_PUBLIC_POSTHOG_HOST=https://us.i.posthog.com

# Langfuse
LANGFUSE_PUBLIC_KEY=pk-lf-xxx
LANGFUSE_SECRET_KEY=sk-lf-xxx
LANGFUSE_BASE_URL=https://cloud.langfuse.com

# Trigger.dev
TRIGGER_SECRET_KEY=tr_dev_xxx
```

## File Layout
```
next.config.ts                              — withSentryConfig wrapper
instrumentation-client.ts                   — Sentry client init + PostHog browser init
sentry.server.config.ts                     — Sentry server init
sentry.edge.config.ts                       — Sentry edge init
instrumentation.ts                          — Langfuse via OpenTelemetry (NodeSDK + LangfuseSpanProcessor)
src/lib/posthog-server.ts                   — PostHog server-side (posthog-node, flushAt:1, flushInterval:0)
src/lib/flags.ts                            — PostHog feature flag helpers (getKiraFlags)
src/lib/tracing.ts                          — OpenTelemetry span helper (withSpan)
src/lib/sentry.ts                           — captureRouteError helper
trigger.config.ts                           — Trigger.dev config (dirs: ["./src/trigger"])
src/trigger/nightly-eval.ts                 — Nightly eval replay task
src/app/api/admin/run-nightly-eval/route.ts — Trigger task from route
```

## PostHog Feature Flags to Create
- `forced_tools` — Force search tool for clinical queries
- `inline_citations` — Inline evidence linking in responses
- `verifier_v1` — Second-pass groundedness verifier
- `reranker_long_tail_only` — Selective reranker for rare queries
- `differential_card_v2` — Updated differential UI with disclaimers

## Success Criteria (first pass)
- Sentry: uncaught route errors + p95 chat latency visible
- PostHog: feature flags live for forced_tools and inline_citations
- Langfuse: one full trace per chat turn with model/tool metadata and cost
- Trigger.dev: nightly replay job running against eval set

## Key Rule
No new modeling work until tracing + flags + one nightly eval job are live. Biggest bottleneck is observability, not model quality.

## PostHog Server-Side Note
In short-lived Next.js server functions: use flushAt: 1, flushInterval: 0, and await posthog.shutdown() so events actually send on Vercel.
