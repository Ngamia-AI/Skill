---
name: ngamia-integrations
description: Integrate developer applications with Ngamia models and AI capabilities using a pre-provisioned API key. Use when building or troubleshooting a client for Ngamia chat, embeddings, audio, voice, image, document, or video APIs, including model discovery, streaming, uploads, retries, and response handling.
---

# Ngamia developer integrations

Use this skill to connect an application or service to Ngamia's model catalog and AI capabilities. Assume the developer already has an API key (`ngm_...`) from their organization. Treat it as a secret and send it as `Authorization: Bearer <API_KEY>` over HTTPS. Load it from the host's secret store or environment; never hardcode it in source, browser code, logs, or examples committed to a repository.

This is a capability integration guide. The caller should not need Ngamia account-management or admin flows. Do not introduce those flows when the task is only to call models or capabilities.

## Integration sequence

1. **Read the current contract.** Start with the public [Ngamia developer docs](https://docs.ngamia.cc). For implementation detail, use the API repository's [developer integration guide](https://github.com/Ngamia-AI/ngamia_api/blob/main/docs/integrations.md), [OpenAPI spec](https://github.com/Ngamia-AI/ngamia_api/blob/main/docs/openapi.yaml), and [models and gateway examples](https://github.com/Ngamia-AI/ngamia_api/blob/main/docs/frontend/models-and-gateway.md). These define current paths, fields, limits, response types, and capability support.
2. **Configure the client.** Use `https://api.ngamia.cc` in production or the developer's supplied base URL. Keep the API key in server-side configuration. For OpenAI-compatible SDKs, use `https://api.ngamia.cc/v1` as `base_url` and the Ngamia key as `api_key`.
3. **Discover models at runtime.** Call `GET /v1/models` with the API key. Select the `model` identifier whose `input_modalities`, `output_modalities`, and supported parameters match the feature. Use that `model` value verbatim in requests. Avoid hardcoded model lists because availability and pricing can change.
4. **Call the capability.** Use the endpoint and wire format from the integration guide. Chat, embeddings, and several media endpoints use OpenAI-compatible payloads; analysis helpers and asynchronous video jobs use the documented Ngamia envelope. Check the endpoint's content type and response shape before parsing.
5. **Handle operations safely.** Surface authentication, insufficient-credit, validation, rate-limit, and upstream errors distinctly. For supported non-streaming billable POSTs, send a stable `Idempotency-Key` for retries of the same logical request. Use a new key for a new operation. Do not retry a partially consumed stream as if it were a fresh request.
6. **Check the finished integration.** Confirm that the request uses the selected catalog model, the right modality and content type, handles the documented response format, and does not expose or log credentials or private media.

When working in a developer's codebase, do the integration work end to end: inspect the existing app and its HTTP/SDK conventions, implement the smallest complete path, connect configuration through the project's secret mechanism, handle loading/errors/output in the caller, and document any setup the developer must perform. Don't stop after giving a sample request if the task asks for a working integration. If the user only wants guidance, provide runnable code and identify exactly where the key and model identifier come from.

## Capability routing

Use the task's actual input and output to choose a route:

- Text generation or tool calls: `POST /v1/chat/completions`.
- Embeddings: `POST /v1/embeddings`.
- Transcription and speech generation: `POST /v1/audio/transcriptions` and `POST /v1/audio/speech`.
- Combined Kiswahili speech workflow: `/v1/bonga` (or documented aliases and voice response routes).
- Private PDF, image, or video uploads: `POST /v1/documents/analyze`, `POST /v1/vision/analyze`, or `POST /v1/video/analyze`.
- Image/video already represented as a URL or data URI: use supported `image_url` or `video_url` content parts on chat completions when the chosen model advertises that input modality.
- Asynchronous video generation: `POST /v1/videos`, poll `GET /v1/videos/{job_id}`, then fetch completed output from `/content`.

The catalog and OpenAPI guide determine whether a specific route/model combination is enabled. Do not infer that a capability is supported merely because an upstream provider supports it.

## Minimal request examples

Use the API key supplied to the integration in place of `$NGAMIA_API_KEY`:

```sh
curl https://api.ngamia.cc/v1/models \
  -H "Authorization: Bearer $NGAMIA_API_KEY"
```

```sh
curl https://api.ngamia.cc/v1/chat/completions \
  -H "Authorization: Bearer $NGAMIA_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-summary-2026-001" \
  -d '{"model":"MODEL_FROM_CATALOG","messages":[{"role":"user","content":"Summarize this text."}],"stream":false}'
```

For an OpenAI-compatible SDK, configure the Ngamia `/v1` base URL and API key, then call the SDK's normal chat or embeddings method. Keep the model identifier sourced from `GET /v1/models`.

Python example with the OpenAI SDK:

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["NGAMIA_API_KEY"],
    base_url="https://api.ngamia.cc/v1",
)

response = client.chat.completions.create(
    model="MODEL_FROM_CATALOG",
    messages=[{"role": "user", "content": "Say hello in Swahili."}],
)
print(response.choices[0].message.content)
```

JavaScript example with the OpenAI SDK:

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.NGAMIA_API_KEY,
  baseURL: "https://api.ngamia.cc/v1",
});

const response = await client.chat.completions.create({
  model: "MODEL_FROM_CATALOG",
  messages: [{ role: "user", content: "Say hello in Swahili." }],
});
console.log(response.choices[0].message.content);
```

These examples show the standard SDK setup. Replace `MODEL_FROM_CATALOG` with a model returned by `GET /v1/models`; for other capabilities, inspect the endpoint contract and use that endpoint's SDK method or HTTP request format.

## Response and reliability rules

- OpenAI-compatible endpoints return OpenAI-shaped JSON, SSE, multipart-related outputs, or raw media bytes as documented. They do not all use Ngamia's standard JSON envelope.
- Convenience analysis endpoints and video job control use the envelope `{status, code, data, request_id}` on success and include `error_code` and `message` on errors.
- Treat `401` as a missing or invalid key, `402` as insufficient credits for a billable request, `429` as a signal to back off, and `5xx` as a provider or service failure. Respect each endpoint's retry and idempotency behavior.
- Keep private prompts, uploads, generated media, and credentials out of logs. Treat model output as untrusted input and validate it before using it for consequential actions.
- For streaming, consume the response incrementally using SSE or the documented media streaming format. Do not parse it as one JSON document.

## Source of truth

The public entry point is [docs.ngamia.cc](https://docs.ngamia.cc). The API repository's [developer integration guide](https://github.com/Ngamia-AI/ngamia_api/blob/main/docs/integrations.md) is the short route map; its [OpenAPI spec](https://github.com/Ngamia-AI/ngamia_api/blob/main/docs/openapi.yaml) defines request and response contracts, and [models and gateway](https://github.com/Ngamia-AI/ngamia_api/blob/main/docs/frontend/models-and-gateway.md) has detailed examples. When those disagree, inspect the implementation and update API documentation before copying a stale example into a client.
