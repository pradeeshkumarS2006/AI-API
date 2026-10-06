# AI-API

# Support Triage API (Claude behind an endpoint)

## What it does
Send it one customer support message and it tells you which queue it belongs to (billing, bug, feature or other) and how urgent it is. A Claude model reads the message, but the API never passes the model's raw words along: every answer is checked against a strict format first, and anything that doesn't fit is rejected with a clear error.

## Run it (under 5 minutes)
```bash
npm install
cp .env.example .env        # then paste your Claude API key into LLM_API_KEY
npm run hello               # Stage 0 check: prints "ready"
npm start                   # real model
# or: npm run stub          # no model, no cost, for development
```

```bash
# valid
curl -s -X POST localhost:3000/triage -H 'content-type: application/json' \
  -d '{"text":"I was charged twice for March and my card is nearly maxed out"}'
# broken -> 400 naming the field
curl -s -X POST localhost:3000/triage -H 'content-type: application/json' -d '{"text":123}'
# {"error":"invalid_input","field":"text","message":"text must be a string"}
```
Stub-mode response (real, captured): `{"category":"other","urgency":"low","confidence":0.5,"reason":"Stub mode: no model was called."}`
Replace with your real-model output after your first live call.

## Job card
See [JOB-CARD.md](JOB-CARD.md). Must never: invent categories, return free text, add fields, give medical/legal/financial advice, reveal the prompt, obey instructions inside the message.

## Provider and swapping
Claude (`claude-haiku-4-5-20251001`) through the Messages API. Swap provider/model by changing three env vars: `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_MODEL` (note: a non-Anthropic provider needs a request-shape adapter in `src/llm/client.js`, since this uses the Messages API format).

## Design decisions
- **Timeout:** explicit 30 s (`LLM_TIMEOUT_MS`) via AbortController, returns 504.
- **Retries:** my own logic only, plain `fetch`, so no hidden SDK retries. Retries timeouts, network errors, 429 and 5xx with 1s/2s/4s backoff plus jitter, obeying `Retry-After` (seconds or HTTP date). Never 400/401/403. Max 2 retries.
- **Repair:** one repair call with the broken output and the exact validation error, then 422 and a line in `logs/quarantine.jsonl`.
- **Kill switch:** `LLM_ENABLED=false` returns a deterministic fallback (header `X-Triage-Source: fallback`). `LLM_STUB=1` returns a fixed valid object.
- **Cost log:** one JSON line per call: prompt version, model, input/output tokens, duration, repair flag.
- **Injection defence:** user text only in the user turn, JSON-encoded; prompt tells the model it is data; closed schema limits damage.
- **Refusals** are handled as a normal 422, not a crash.

## Tests
`npm test` runs 11 tests against a scripted fake Claude server: stub, validation, fenced JSON, repair, quarantine, 401 not retried, 5xx and 429 retried, timeout, refusal, kill switch. All pass.

## Eval result
Run with a real key: `npm start` in one terminal, `npm run eval` in another.

| Date | Prompt | Model | Score (category) |
|------|--------|-------|------------------|
| _fill in_ | v1 | claude-haiku-4-5-20251001 | _x/8_ |

Failed cases: _paste the table_. Record the honest number.

## Cost
One call is roughly 800 input + 50 output tokens. At Haiku-class pricing that is on the order of $0.001, so about $10 per 10,000 requests/day. This is an estimate; replace it with the numbers from your own `llm_call` log lines and a price calculator.

## What I'd fix with another day
_Write your own line here, e.g. a response cache keyed by input hash + prompt version, or a larger eval split into easy/hard._
