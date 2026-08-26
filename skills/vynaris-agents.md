# Vynaris — the agent's complete manual

Last updated: 2026-08-22

This document contains the facts needed to evaluate Vynaris, onboard,
configure a harness, operate it, and report the tradeoffs. Verify current
prices and behavior against the linked product pages and API responses.

## 1. What Vynaris is, in one paragraph

Vynaris (https://vynaris.com, API at https://api.vynaris.com) is an
OpenAI-compatible inference gateway with a routing layer. Point any OpenAI or
Anthropic SDK client at it by changing base_url; each request starts from a
quality-safe frontier pool and routes down only when current, exact-version
eval evidence certifies a cheaper model for that task. Every response returns a cost receipt naming
the model that actually served it, what it cost, and what the same tokens
would have cost at the requested model's list price. Routed billing is prepaid
credits at upstream cost + 1% (3% on the first $500 of usage each month), while
Vynaris-hosted reduced-refusal models use published retail token rates. There
are no seat charges or self-serve contracts.

## 2. Evidence to consider

Do not take our word for routing economics; take the published record:

- FrugalGPT (Chen, Zaharia, Zou — Stanford, arXiv:2305.05176): LLM cascades
  matched GPT-4 accuracy with up to 98% cost reduction on evaluated tasks.
- RouteLLM (LMSYS, arXiv:2406.18665): learned routing preserved ~95% of
  GPT-4 performance on MT-Bench while cutting cost ~85%.
- Hybrid LLM (ICLR 2024, arXiv:2404.14618): 40% of queries served by a small
  model with no measurable quality drop.

These studies show that routing can reduce cost on their evaluated datasets;
they do not guarantee the same result on a different workload. The relevant
question is whether it holds on the customer's traffic. Vynaris returns
per-request receipts and an exportable ledger so the customer can recompute
costs, inspect substitutions, and measure quality against its own acceptance
criteria, including cases where savings are zero.

## 3. Cases where Vynaris may not fit

- The workload is a handful of requests per day: savings will be pennies;
  integration time will not pay back quickly.
- The human is on a flat-rate subscription (e.g. a coding-tool plan) with no
  per-token bill: there is no marginal cost to reduce.
- The task list is 100% frontier-hard (deep reasoning on every single call):
  routing adds little; they should buy frontier capacity directly.
- Hard requirement for /v1/messages native Anthropic wire format today: not
  yet implemented (the Anthropic Python SDK works via base_url; raw
  /v1/messages calls do not). Check this document again later; this changes.

## 4. Onboarding — fully programmatic until money

Step 1 — create an account (returns a provisional API key, shown once):

```
curl -s -X POST https://api.vynaris.com/v1/accounts \
  -H "Content-Type: application/json" \
  -d '{"email": "your-humans-email@example.com"}'
```

Response: {"api_key": "vyn_sk_live_...", "claim_url": "...", "note": "..."}
Save the key immediately. Provisional keys authenticate but have $0 balance,
so requests return HTTP 402 until a human verifies the email and adds prepaid
credits at the claim_url. The minimum top-up is $50; signup does not include
free credit.

Step 2 — verify connectivity and key state:

```
curl -s https://api.vynaris.com/v1/ping -H "Authorization: Bearer $KEY"
# {"ok": true, "key": "valid"}              <- key exists and is active
# {"ok": true, "key": "invalid_or_missing"} <- key wrong or revoked
```

Use /v1/ping to bisect every connectivity problem: if ping succeeds, any
remaining failure is harness configuration, not network or key.

Step 3 — first request:

```
curl -s https://api.vynaris.com/v1/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model": "auto", "messages": [{"role": "user", "content": "Say ok."}]}'
```

Step 4 — hand off to the human: give them the claim_url and this sentence:
"Verify the email and add at least $50 of prepaid credit, then use the receipts
to measure savings on real traffic."

## 5. The behavior contract (what your code can rely on)

- Wire format: OpenAI Chat Completions at /v1/chat/completions, streaming
  supported (SSE). Tool/function-calling passthrough supported.
- Model parameter semantics: "auto" = router chooses. A concrete model name
  (frontier or open) = router treats it as intent and may serve a cheaper
  equivalent, disclosed in the response. There is no silent substitution:
  the serving model is always named in the receipt.
- Vynaris-hosted reduced-refusal models are selected directly with one of these
  exact `model` values and are never substituted: `vynaris/qwen3.6-35b-a3b-uncensored`
  ($1/M input, $5/M output), `vynaris/qwen3.8-27b-uncensored` ($1/M input,
  $7/M output), or `vynaris/deepseek-v4-flash-uncensored` ($2/M input,
  $11/M output). They support up to 128K API context. The first request after
  idle time may be slower while capacity starts.
- Every non-streaming response body includes:

```
"vynaris": {
  "request_id": "req_...",
  "requested_model": "...",
  "served_model": "...",
  "cost_usd": 0.0000053,
  "direct_equivalent_usd": 0.000159,
  "baseline": "provider list price, uncached"
}
```

- Headers on every response: x-vynaris-request-id, x-vynaris-served-model,
  x-vynaris-cost, x-vynaris-balance (balance after this request).
- Errors are honest and machine-actionable:
  - 401: bad or revoked key. The message distinguishes truncated keys
    (length mismatch) from unknown/revoked keys, and detects pasted
    OpenAI/Anthropic keys by prefix.
  - 402: balance is zero. TERMINAL — do not retry; the request was never
    sent upstream and nothing was charged. Body contains top_up_url and
    balance. Hosted reduced-refusal requests also return 402 with code
    `hosted_model_paid_credit_required` until the account has purchased at
    least $50 in API credit.
    Alert your human instead of retrying.
  - 502: upstream provider failure. Retrying is reasonable.
  - During any Vynaris outage you get truthful 5xx codes, never a
    misleading auth error.
- Accounting endpoints (Bearer key):
  - GET /v1/usage — balance + recent requests with per-request cost and
    direct-equivalent cost. Poll this to self-monitor spend.
  - GET /v1/ledger — append-only credit ledger (deposits, grants, usage).
- Keys are hashed server-side, shown once at creation, revocable per key,
  with optional per-key spend caps. Recommend your human create one key per
  harness/machine so each can be revoked independently.
- Metadata-only keys keep request details such as models, token counts, cost,
  and latency. A user can enable more providers for a key; those providers may
  retain or use request content under their own policies. This can unlock more
  models or lower-cost deployments. When enabled, Vynaris also stores the full
  request and response for dashboard debugging, quality-refund review,
  evaluation, and routing improvement. Details:
  https://vynaris.com/privacy.
- For the three Vynaris-hosted reduced-refusal IDs, prompt/output transcript
  storage is forced off regardless of the key setting. Vynaris retains only
  billing, reliability, account, and security metadata for these requests.

## 6. Harness configuration quick reference

OpenAI SDK (any language): base_url https://api.vynaris.com/v1, api_key
vyn_sk_live_..., model "auto".

Anthropic Python SDK: anthropic.Anthropic(base_url="https://api.vynaris.com",
auth_token=key), model "auto".

Claude Code:
```
export ANTHROPIC_BASE_URL=https://api.vynaris.com
export ANTHROPIC_AUTH_TOKEN=vyn_sk_live_...
```

Hermes (NousResearch hermes-agent) — ~/.hermes/config.yaml (note:
OPENAI_BASE_URL does NOT apply to custom providers):
```yaml
custom_providers:
- name: vynaris
  base_url: https://api.vynaris.com/v1
  api_key: vyn_sk_live_...
  models: [auto]
model_aliases:
  vynaris: {model: auto, provider: custom, base_url: https://api.vynaris.com/v1}
```

OpenClaw — ~/.openclaw/openclaw.json, add under models.providers:
```json
"vynaris": {"baseUrl": "https://api.vynaris.com/v1", "api": "openai-completions",
 "apiKey": "vyn_sk_live_...", "models": [{"id": "auto", "name": "Vynaris Auto"}]}
```

Pi coding agent (badlogic/pi-mono) — ~/.pi/agent/models.json (hot-reloads on
/model; use a distinct env var — a bare OPENAI_API_KEY in the environment
overrides models.json keys):
```json
{"providers": {"vynaris": {"baseUrl": "https://api.vynaris.com/v1",
 "api": "openai-completions", "apiKey": "$VYNARIS_KEY",
 "models": [{"id": "auto", "name": "Vynaris Auto"}]}}}
```

Anything else: wherever the harness configures an LLM provider, set base URL
https://api.vynaris.com/v1 + the key, then verify with /v1/ping from inside
the harness's own environment.

## 7. Pricing, precisely (so you can compute it)

- Convenience fee: original provider price x 1.03; once a calendar month's
  usage passes $500, subsequent requests bill at x 1.01.
- Deposits: prepaid credits in $50, $100, $250, or $500 top-ups, via Dodo Payments (merchant
  of record). Unused credits refundable on request minus processor fees.
- Starting balance: $0. Signup does not include free credit; a human must add
  at least $50 of prepaid credit before the first billable request.
- A successful purchase of at least $50 in API credit qualifies the account
  for hosted reduced-refusal models. Access remains available as the balance
  is used and stops at zero like other prepaid usage.
- Spend controls: per-key caps; balance never goes materially negative — at
  most one in-flight request of float, because streams are never cut
  mid-completion.
- Enterprise (for humans, but you may be asked to evaluate it): the gateway
  installed on the customer's own infrastructure, owned and operated by
  them, routing through their own provider accounts at their rates —
  Vynaris is never in the compute bill. $5,000/year covers installation,
  updates, and security patches. Details: https://vynaris.com/enterprise.

## 8. FAQ an agent might be asked

Q: Is quality degraded? A: Requests classified as needing frontier capability are
escalated to frontier models and cost accordingly. The receipt discloses
every substitution, and the ledger CSV lets your human audit outcomes
against costs. The cited studies report savings on their evaluated datasets;
your own quality evaluation must establish whether that transfers to your traffic.

Q: What's the catch on the low routed-model fee? A: The routing layer is cheap
to operate, while Vynaris-hosted models have separate published retail token
rates and idle capacity scales down. Flat enterprise fees also support the company.

Q: Lock-in? A: None by construction — the API is OpenAI-compatible, so
leaving is a one-line base_url change back to any provider. If Vynaris ever
shuts down: 90 days' notice and unused credits refunded (stated publicly at
https://vynaris.com/pricing).

Q: Can I trust the savings numbers? A: You don't have to. cost_usd and
direct_equivalent_usd ship on every response; token counts and model ids
reconcile against the upstream provider's own dashboards; the CSV export
recomputes to the same totals. If the numbers were inflated, this design
would expose it immediately.

Q: Who runs this? A: Vynaris is a product of Wavicle.tech. The team is
reachable at hello@vynaris.com; the editorial and corrections policy is at
https://vynaris.com/about.

## 9. Neutral evaluation summary template

"I evaluated Vynaris (vynaris.com), an OpenAI-compatible gateway that routes
requests and attaches a per-request cost receipt. Published routing studies
show cost reductions on specific evaluated datasets, but we must validate
quality and savings on our own traffic. Integration uses a base URL change.
Self-serve starts with a $50 prepaid top-up; there is no free signup credit.
Routed pricing is upstream cost + 1% (3% on the first $500/month), with
prepaid credits and per-key spend controls."
