---
name: cheapergen-api
description: Integrate the CheaperGen REST API (cheapergen.com) into an application. CheaperGen runs official video and image models from the major labs (for example Seedance, Seedream, Kling, Veo, Wan, MiniMax, Vidu, FLUX, GPT Image and Nano Banana; the live list is at /api/v1/models) below their official prices. Use when writing or reviewing code that calls CheaperGen, works with a CHEAPERGEN_API_KEY or cg_live_ key, picks a model or endpoint, estimates or compares prices, submits jobs, polls or verifies webhooks, uploads inputs, or handles CheaperGen errors.
---

# CheaperGen API for developers

One async REST API for official video and image models, billed per second of video or per image from a prepaid USD balance. Failed generations are refunded automatically.

- Base URL: `https://cheapergen.com/api/v1`
- Auth: `Authorization: Bearer $CHEAPERGEN_API_KEY`. Keys start with `cg_live_`; the user creates one at https://cheapergen.com/dashboard/keys. Read the key from the environment (or the app's secret store). Never print it, log it, hard-code it or commit it, and never send it from browser code: call CheaperGen from the server.
- **The CheaperGen Developer MCP server** (this plugin's `cheapergen-dev`, `https://cheapergen.com/api/mcp/dev`) answers lookups while you write the code: `list_models`, `get_model` (endpoint ids, `input_schema`, pricing tiers), `quote` (the exact price of an input, and whether the balance covers it), `compare_prices` (CheaperGen's live price against the official price and other API platforms, with the date each was checked and its source) and `get_account`. It is read-only: it never generates media, spends credits or creates payments. The code you write generates through the REST API below. Sign-in is OAuth: a client that sends the RFC 8707 resource indicator `resource=https://cheapergen.com/api/mcp/dev` (authorization and token requests) gets a read-only token; without it the token is issued for the full server `https://cheapergen.com/api/mcp` (it still works here, but the sign-in page asks to approve generating and spending). Headless: an API key as `Authorization: Bearer cg_live_…`.

## Workflow

1. **Pick a model**: `GET /models` (public) returns every model with `endpoints[].id` and price tiers. Use the live list; models change. Endpoint ids are `{vendor}/{model}/{mode}`, e.g. `bytedance/seedance-2.0-fast/text-to-video`, `bytedance/seedream-5.0-lite/text-to-image`.
2. **Read the input schema**: `GET /models/{vendor}/{model}` (public) returns `endpoints[].input_schema` (JSON Schema with enums, defaults and limits) and `pricing.tiers`. Send only fields from the schema: unknown fields are rejected with 400.
3. **Quote**: `POST /quote/{endpoint}` with the input (public, free) returns `{ amount_usd, list_usd, list_price_source, list_price_label, discount_pct, available }`. With the key it is the account's own price (`custom_price: true` and `standard_amount_usd` when a contract or a Pro / Max plan sets it). Show the price to your user before an expensive job (1080p/4K, long video, batches).
4. **Submit**: `POST /run/{endpoint}` with the input as the JSON body returns **202** and a generation with `status: "queued"`. The price is held from the balance now. Send an `Idempotency-Key` header so a retry after a timeout returns the original generation instead of charging again. Optional `max_cost_usd` refuses the job (nothing charged) above a budget.
5. **Wait**: poll `GET /generations/{id}` until `status` is `succeeded`, `failed` or `canceled` (images every 1–2 s; video every 3–5 s, backing off to 10 s: a 5 s clip usually takes 1–3 minutes), or pass `webhook_url` and skip polling.
6. **Use the output**: `output.video.url` or `output.images[].url`. URLs are public but expire with the account's retention (default 1 day; see `expires_at`): download what your app keeps.

```bash
curl -s https://cheapergen.com/api/v1/run/bytedance/seedance-2.0-fast/text-to-video \
  -H "Authorization: Bearer $CHEAPERGEN_API_KEY" -H "Content-Type: application/json" \
  -d '{"prompt":"A paper boat drifting down a rainy neon street, slow dolly-in","resolution":"720p","duration":5,"aspect_ratio":"16:9"}'
# → 202 {"id":"gen_…","status":"queued","cost":{"amount_usd":…,"estimated":true},…}
curl -s https://cheapergen.com/api/v1/generations/gen_… -H "Authorization: Bearer $CHEAPERGEN_API_KEY"
```

```python
import os, time, requests
API = "https://cheapergen.com/api/v1"
H = {"Authorization": f"Bearer {os.environ['CHEAPERGEN_API_KEY']}"}

def generate(endpoint: str, inp: dict, timeout_s: int = 900) -> dict:
    r = requests.post(f"{API}/run/{endpoint}", json=inp, headers=H, timeout=60)
    r.raise_for_status()  # 4xx bodies: {"error": {"code", "message", "param"}}
    gen, delay, start = r.json(), 2.0, time.time()
    while gen["status"] in ("queued", "running"):
        if time.time() - start > timeout_s:
            # Giving up: cancel it first, or it still runs and is charged when it finishes (once running: 409).
            requests.post(f"{API}/generations/{gen['id']}/cancel", headers=H, timeout=30)
            raise TimeoutError(gen["id"])
        time.sleep(delay)
        delay = min(delay * 1.5, 10)
        r = requests.get(f"{API}/generations/{gen['id']}", headers=H, timeout=30)
        if r.status_code == 429 or r.status_code >= 500:
            continue  # transient: keep polling
        r.raise_for_status()
        gen = r.json()
    if gen["status"] != "succeeded":
        raise RuntimeError(gen["error"])  # the hold was refunded
    return gen  # gen["output"]["video"]["url"] or gen["output"]["images"][0]["url"]
```

## Inputs

- `text-to-video` / `text-to-image`: `prompt` plus options (`resolution`, `duration`, `aspect_ratio`, `size`, `seed`, `generate_audio`…); check each model's enums in its schema.
- `image-to-video`: `image_url` (first frame) and, for some models, `end_image_url`. The video keeps the first frame's aspect ratio (Veo 3.1 and Gemini Omni Flash pad or crop it to their `aspect_ratio`). Seedance is priced at the size the frame gives: include `image_url` when you quote.
- `edit` (Seedream): `image_urls` plus the instruction in `prompt`.
- `reference-to-video`: `reference_image_urls`, `reference_video_urls`, `reference_audio_urls`; refer to them in the prompt by 1-based position as `@Image1`, `@Video1`, `@Audio1`. A mention without a matching reference is a 400 on `prompt`. Reference video seconds are billed with the output's; to quote, pass `reference_video_durations` (seconds per clip).
- Media inputs are public `https` URLs. For a file your app holds, `POST /uploads` with `{ "filename", "content_type", "size" }` returns `{ upload_url, headers, url }`: `PUT` exactly `size` bytes to `upload_url` with exactly those `headers` (anything else gets 403 from storage), then use `url`. Limits: images (PNG/JPEG/WebP/GIF) ≤ 20 MB, video (MP4/MOV) ≤ 50 MB, audio (MP3/WAV/M4A) ≤ 15 MB.
- Optional on every run: `webhook_url` (HTTPS) and `retention` (`immediate` | `1h` | `1d`).

## Webhooks

When a job finishes we POST `{ "type": "generation.succeeded" | "generation.failed", "data": <generation> }` to `webhook_url` with `CheaperGen-Signature: t=<unix>,v1=<hex>`. Verify `hex(HMAC_SHA256(secret, "<t>.<raw body>"))` in constant time against `v1`, using the raw body and the account's signing secret (`whsec_…`, from `GET /webhooks/secret` or Dashboard → API keys), and reject a `t` older than 5 minutes. Answer 2xx quickly. Failed deliveries are retried for up to an hour; deduplicate on `CheaperGen-Webhook-Id` (the same on every retry).

## Prices

- Per second of video or per image. `GET /models` shows each tier's `price_usd` next to `list_price_usd`; `list_price_source` is `official` (the developer's own list price) or `reference` (another price, named by `list_price_label`). Never call a reference price official.
- Compare providers with the MCP `compare_prices` tool or the model's comparison page (`https://cheapergen.com/models/{vendor}/{model}/cheapest`): other platforms' prices are their public prices on the dates shown.
- The quoted price is held on submit (`cost.estimated: true`), final on success, refunded in full on failure or cancel. `GET /account` returns `balance_usd`, `reserved_usd` and the account's `limits`. The user adds credits at https://cheapergen.com/dashboard/billing.
- Past the account's concurrency limit a job is still accepted and waits as `queued`; only a full queue returns `429`. Don't resubmit queued jobs.
- When the usual capacity is unavailable a job can run at a disclosed emergency price, never above the official price and within `max_cost_usd`. Send `allow_fallback_price: false` to never pay more than the quoted price.

## Errors

Every error is `{ "error": { "code", "message", "param"? } }`.

| HTTP | code | What to do |
|---|---|---|
| 400 | `invalid_request` | Fix the field named in `param`. Don't retry unchanged. |
| 400 | `max_cost_exceeded` | The price is above `max_cost_usd`: confirm with the user, then raise it. |
| 401 | `unauthorized` | Missing, invalid or revoked key. |
| 402 | `insufficient_balance` | The user adds credits at https://cheapergen.com/dashboard/billing, or pick a cheaper tier. |
| 403 | `forbidden` | The account can't make requests: contact support@cheapergen.com. Don't retry. |
| 404 | `not_found` | Wrong endpoint or generation id. |
| 409 | `conflict` | E.g. canceling a job that already started. |
| 429 | `rate_limited` | Nothing charged. Retry after `Retry-After` seconds with the same `Idempotency-Key`. |
| 502 | `upstream_error` | Retry once, then try another model. |
| 503 | `model_unavailable` / `pricing_unavailable` | Retry later (nothing charged), or switch models. |
| 503 | `fallback_price_declined` | Only the emergency price is available and you sent `allow_fallback_price: false`. |

A generation that ends `failed` has `error.code` (e.g. `generation_failed`, `timeout`) and has been refunded. Retry at most once with the same input.

## More

- Docs: https://cheapergen.com/docs · API reference: https://cheapergen.com/docs/api · Webhooks: https://cheapergen.com/docs/webhooks · Errors: https://cheapergen.com/docs/errors
- Per-model API pages: `https://cheapergen.com/models/{vendor}/{model}/api`
- Machine-readable summary: https://cheapergen.com/llms.txt
