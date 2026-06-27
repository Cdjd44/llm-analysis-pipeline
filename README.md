# llm-analysis-pipeline

A production-pattern pipeline for running structured LLM analysis via the Anthropic API. Built to demonstrate the full operational story: secure key handling, prompt versioning, automated evals, and per-call cost observability.

---

## Architecture

```
┌─────────────────┐        ┌──────────────────┐        ┌─────────────────┐
│   client/       │  HTTP  │   proxy/          │  SDK   │  Anthropic API  │
│   index.html    │───────▶│   index.js        │───────▶│  claude-*       │
│                 │◀───────│   (Azure Fn)      │◀───────│                 │
│  no API key     │  JSON  │   holds the key   │  usage │                 │
└─────────────────┘        └────────┬─────────┘        └─────────────────┘
                                    │
                         ┌──────────┴──────────┐
                         │  observability/     │
                         │  costLogger.js      │
                         │  (tokens + $)       │
                         └─────────────────────┘
```

**Key design decisions:**

- **Proxy holds the key.** The client never touches the Anthropic key. The proxy is the only trust boundary between user input and the API.
- **Prompts are config, not code.** `prompts/` holds versioned JSON files. Swapping prompt versions requires no code change — just a config update.
- **Evals are first-class.** The `evals/` layer runs on every push via GitHub Actions. This is what separates a demo from a production system.
- **Observability from day one.** Every API call logs token usage and estimated cost. You know what the pipeline costs before it surprises you.

---

## Operational Story

### Local development

```bash
# 1. Install dependencies
npm install

# 2. Copy env template and fill in your key
cp .env.example .env

# 3. Start the proxy (Azure Functions local runtime)
cd proxy && func start

# 4. Open the client
open client/index.html
```

### Running evals locally

```bash
node evals/runEvals.js
```

Evals score model output against `evals/dataset.json`. Each scenario defines expected bounds (tone, length, required terms). A passing run prints a score ≥ the threshold defined in `runEvals.js`.

### CI

Every push triggers `.github/workflows/evals.yml`, which runs the full eval suite against the current prompt version. A failing eval blocks merge — this is intentional.

---

## Prompt versioning

`prompts/analysis.v1.json` is the baseline. `prompts/analysis.v2.json` introduces chain-of-thought scaffolding. The proxy loads the version specified by `PROMPT_VERSION` in `.env`. To A/B test, deploy two instances with different env values.

---

## Cost model

`observability/costLogger.js` reads `input_tokens` and `output_tokens` from the Anthropic response and applies the published per-token rates. Logs go to stdout in structured JSON so they can be forwarded to any log aggregator.

---

## Directory reference

| Path | Purpose |
|---|---|
| `proxy/` | Azure HTTP Function — the only process that holds the API key |
| `proxy/anthropicClient.js` | Thin wrapper around the Anthropic SDK; extracts usage |
| `prompts/` | Versioned prompt configs loaded at call time |
| `evals/` | Dataset + runner + scorers; CI-enforced quality gate |
| `observability/` | Per-call token and cost logging |
| `client/` | Minimal HTML UI; calls proxy, never the API directly |
| `.github/workflows/` | Eval CI — the standout operational piece |
