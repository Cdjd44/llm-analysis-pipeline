# LLM Analysis Pipeline

A small, deliberately focused LLM application built to demonstrate the *operational* layers that separate a production-minded LLM system from a one-off API call. It analyses a stock position and returns a structured `signal` / `score` / `reason`, but the domain is incidental — the point is the scaffolding around the model call.

The whole thing is intentionally tight so that every operational layer is legible in a few minutes of reading:

- **A backend proxy** that holds the API key server-side, so it never reaches the browser.
- **Versioned prompts** kept as configuration files rather than buried string literals.
- **Token-cost accounting** that prices every call from the model's reported usage.
- **An eval harness** with schema, plausibility, and numeric-consistency (hallucination) checks, **wired into CI** so an output regression turns the build red.

> Demonstration project, not financial advice. The analysis output is a vehicle for showing the engineering; it is not a basis for real trading decisions, and the model has no market data beyond the single position passed to it.

## Architecture

```
Browser / CLI ──POST──> Backend Proxy ──────> Anthropic API
  (holds no key)         (Azure Function)        │
                              │                   │
                              ├─ prompts/        (versioned prompt config)
                              ├─ pipeline        (load → call → validate → cost)
                              ├─ costLogger      (usage → USD, per call)
                              └─ structured-output validation

evals/  ── imports the SAME pipeline ──> runs a curated dataset through it,
                                          scores the output, fails CI on regression
```

The single most important design decision is that the **model client, prompt config, and cost logger are injected** into one shared pipeline module. The proxy builds that pipeline with the live Anthropic client; the eval harness builds it with a fixture-backed client. So the code exercised by the tests is the same code that ships, not a parallel reimplementation.

The second decision worth calling out: **evals run in two modes**. Every-push CI uses recorded fixtures — deterministic, free, no key required. A separate, deliberate job (not on every commit) runs against the live model to catch genuine drift. Conflating "test my pipeline logic" with "test the live model" is a common mistake; they are kept apart on purpose.

## Why this structure

This project started life as a single-file browser dashboard that called the model API directly from client-side JavaScript. That approach has one unfixable flaw: any key in front-end code is exposed, because the browser must be able to read it. The rebuild exists to do it correctly — the proxy pattern is the fix, and once there is a server-side component, every operational layer finally has somewhere to live. None of them are possible in a pure client-side file.

## Layout

```
.
├── proxy/
│   ├── index.js          Azure Function HTTP trigger — the proxy, holds the key
│   ├── pipeline.js       core loop, shared by proxy + evals (dependency-injected)
│   └── anthropicClient.js  the Anthropic call; surfaces the usage object
├── prompts/
│   └── analysis.v1.json  prompt-as-config; bump the version to roll forward
├── observability/
│   └── costLogger.js     token → USD accounting per call
├── evals/
│   ├── dataset.json      curated scenarios with expected bounds
│   ├── scorers.js        schema, plausibility, numeric-consistency checks
│   ├── runEvals.js       runs the dataset through the pipeline; non-zero on fail
│   └── fixtures/         recorded model outputs for deterministic CI
├── client/
│   └── index.html        deliberately thin demo client — holds no key
└── .github/workflows/
    └── evals.yml         runs the eval harness on every push
```

## Running it

**Evals (no key needed — this is the interesting part):**

```bash
npm install
npm run eval
```

This runs every scenario through the pipeline against recorded fixtures and scores the output. One fixture carries a deliberately planted inconsistency — the model's prose cites a percentage that matches neither the true profit/loss nor the day's move — to demonstrate the consistency scorer catching an unfaithful figure and failing the run.

**Evals against the live model (drift detection):**

```bash
cp .env.example .env      # then add a real ANTHROPIC_API_KEY
npm run eval:live
```

**The proxy locally** (requires the Azure Functions Core Tools):

```bash
npm start                 # serves the analyze endpoint on :7071
```

Then open `client/index.html` and point `PROXY_URL` at the local endpoint.

## The eval checks, briefly

1. **Schema** — is the output valid JSON, with `signal` in the allowed set and `score` an integer in range? Malformed output is a first-class failure, never silently coerced.
2. **Plausibility** — does the signal/score fall inside loose, per-scenario bounds? These are intentionally wide; there is no single correct answer, so this catches gross errors, not opinions.
3. **Consistency** — the most interesting check. It extracts percentages from the model's prose and reconciles them against the figures actually provided. A number that matches nothing is flagged as a likely hallucination.

The consistency scorer is a smoke detector, not a proof: it reads figures from free text with a regex, so it misses values phrased in words and can occasionally flag a legitimately different number. That limitation is documented in the code, because understanding where a scorer is blind is part of using it responsibly.

## Secret handling

API keys are read from the environment (`.env` locally — gitignored; Application Settings or Key Vault in Azure) and never committed. `.env.example` lists variable names with placeholder values only. The browser client holds no model key at any point.
