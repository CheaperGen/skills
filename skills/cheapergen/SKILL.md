---
name: cheapergen
description: Generate AI videos and images with the Cheaper Gen. API (cheapergen.com), which runs official ByteDance Seedance and Seedream, Alibaba Wan and MiniMax models below their official prices. Use when the user wants to create a video or image with these models, add text-to-video, image-to-video, reference-to-video, text-to-image or image editing to code, or work with a CHEAPERGEN_API_KEY or cg_live_ key. Covers picking a model, quoting the price, submitting, polling or webhooks, @Image1-style reference mentions, uploads, billing and errors.
---

# Cheaper Gen. API

One async REST API for official video and image models. Billing comes from a prepaid USD balance: you pay per second of video or per image, and failed generations are refunded automatically.

- Base URL: `https://cheapergen.com/api/v1`
- Auth: `Authorization: Bearer $CHEAPERGEN_API_KEY` (keys start with `cg_live_`; create one at https://cheapergen.com/dashboard/keys). Read the key from the environment. Never print it, log it or commit it.
- **Prefer the MCP server when it's connected.** Its tools are `list_models`, `get_model`, `quote`, `generate` (with `wait_seconds` and a `max_cost_usd` budget guard), `get_generation` and `get_account`; use them instead of HTTP calls. The workflow below is the same. To connect it, add the remote server `https://cheapergen.com/api/mcp`; the user signs in and approves in the browser (OAuth), with no key needed. In Claude Code: `claude mcp add --transport http cheapergen https://cheapergen.com/api/mcp`, then `/mcp` → Authenticate. Full setup: https://cheapergen.com/docs/mcp

## Workflow

1. **Pick a model**: `GET /models` (public). It returns every model with `endpoints[].id` and price tiers. Always use the live list; models change.
   - Video: Seedance 2.0 (best quality, 4K, synced audio), Seedance 2.0 Fast / Mini (cheaper and faster), Seedance 2.5 (newest, longest references), Wan 3.0 / Prime (Alibaba), MiniMax H3 / H3 Max.
   - Image: Seedream 5.0 Pro / Lite / Flash (text-to-image and `edit`).
   - Endpoint ids are `{vendor}/{model}/{mode}`, e.g. `bytedance/seedance-2.0-fast/text-to-video`, `bytedance/seedream-5.0-lite/text-to-image`.
2. **Read the input schema**: `GET /models/{vendor}/{model}` (public) returns `endpoints[].input_schema` (JSON Schema with enums, defaults and limits) and `pricing.tiers`. Send only fields from the schema: unknown fields are rejected with 400.
3. **Quote**: `POST /quote/{endpoint}` with the input (public, free) returns `{ amount_usd, list_usd, discount_pct, available }`. Before an expensive job (video at 1080p/4K, long durations, batches), tell the user the price.
4. **Submit**: `POST /run/{endpoint}` with the input as the JSON body returns **202** and a generation with `status: "queued"`. The price is held from the balance at this point.
5. **Wait**: poll `GET /generations/{id}` until `status` is `succeeded`, `failed` or `canceled`. Poll images every 1–2 s; poll video every 3–5 s, backing off to 10 s, since a 5 s clip usually takes 1–3 minutes. Or pass `webhook_url` (see below).
6. **Use the output**: `output.video.url` (video) or `output.images[].url` (images). The URLs are public but expire per the account's retention (default 1 day; see `expires_at`), so download anything worth keeping.

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

## Modes and inputs

- `text-to-video` and `text-to-image`: take `prompt` plus options (`resolution`, `duration`, `aspect_ratio`, `size`, `seed`, `generate_audio`…). Check the schema for each model's enums.
- `image-to-video`: takes `image_url` (first frame) and, for some models, `end_image_url` (last frame).
- `edit` (Seedream): `image_urls` (images to edit or combine; refer to them as "image 1", "image 2" in the prompt) plus the instruction in `prompt`.
- `reference-to-video`: takes `reference_image_urls`, `reference_video_urls` and `reference_audio_urls`. In the prompt, point at them by 1-based position: `@Image1`, `@Image2`, `@Video1`, `@Audio1` (case-insensitive). A mention without a matching reference gives a 400 with `param: "prompt"`. Example: `"@Image1 walks through the market from @Image2, camera moves like @Video1"`.
  - `reference_rules` in the schema sets the maximum number of files and whether you need at least one image or video (audio alone is not enough for Seedance and H3).
  - Reference videos and audio are measured on the server (duration, size, resolution, fps) against `media_limits` / `x-media-limits`. Seedance 2.0 takes MP4/MOV videos of 2–15 s each and 15 s in total; audio must be MP3/WAV. A violation gives a 400 that names the file, e.g. `@Video2 is 16 s long…`.
  - Reference video seconds are billed together with the output seconds. Reference images and audio are free. To quote, pass `reference_video_durations: [6.5]` (seconds per clip).
- Media inputs must be public `https` URLs. For local files, call `POST /uploads` with JSON `{ "filename", "content_type", "size" }`. It returns `{ upload_url, headers, url }`: `PUT` the bytes to `upload_url` with exactly those `headers`, then use `url` as the input. Limits: images (PNG/JPEG/WebP/GIF) ≤ 20 MB, video (MP4/MOV) ≤ 50 MB, audio (MP3/WAV/M4A) ≤ 15 MB.
- Optional on every run: `webhook_url` (HTTPS) and `retention` (`immediate` | `1h` | `1d`; `7d` / `30d` need an account setting).

## Webhooks

When the job finishes (succeeded or failed), we POST `{ "type": "generation.succeeded" | "generation.failed", "data": <generation> }` to `webhook_url`, with the header `CheaperGen-Signature: t=<unix>,v1=<hex>`. To verify it, compute `hex(HMAC_SHA256(secret, "<t>.<raw body>"))` with the account's webhook signing secret (`whsec_…`, in Dashboard → API keys), compare in constant time and reject requests whose `t` is more than 5 minutes old. Use the raw request body, not re-serialized JSON. Return 2xx quickly; failed deliveries are retried (3 attempts in total).

## Pricing and billing

- Prices are per second of output video or per image. `GET /models` shows each tier's `price_usd` next to `list_price_usd` (the official price).
- On submit, the quoted price is held (`cost.estimated: true`). On success it becomes final; on failure or cancel it's refunded in full (`cost.amount_usd: 0`). `cost.official_usd` and `cost.saved_usd` show the savings.
- Balance: `GET /account` returns `balance_usd` and `reserved_usd` (held for jobs in progress). A 402 means the balance is too low; the user adds credits at https://cheapergen.com/dashboard/billing.
- Cancel a queued job with `POST /generations/{id}/cancel` (409 once it's running). Delete a finished one with `DELETE /generations/{id}`.

## Errors

Every error has the shape `{ "error": { "code", "message", "param"? } }`.

| HTTP | code | What to do |
|---|---|---|
| 400 | `invalid_request` | Fix the field named in `param` (check the schema, enums, limits and mentions). Don't retry unchanged. |
| 401 | `unauthorized` | Missing, invalid or revoked key. |
| 402 | `insufficient_balance` | Ask the user to top up, or choose a cheaper tier. |
| 404 | `not_found` | Wrong endpoint or generation id. |
| 409 | `conflict` | E.g. canceling a job that already started. |
| 429 | `rate_limited` | Back off and retry. The limit is about 20 requests in a burst, refilled at 10/s per key, plus a cap on concurrent generations per account (default 20). |
| 502 | `upstream_error` | Retry once, then try another model. |
| 503 | `model_unavailable` | Retry later or switch models. |

A generation that ends `failed` has `error.code` (e.g. `generation_failed`, `timeout`) and has been refunded. Retry at most once with the same input.

## More

- Docs: https://cheapergen.com/docs, MCP setup: https://cheapergen.com/docs/mcp
- Source of this skill and the Claude Code plugin: https://github.com/CheaperGen/skills
- Per-model API pages: `https://cheapergen.com/models/{vendor}/{model}/api`
- Machine-readable summary: https://cheapergen.com/llms.txt
