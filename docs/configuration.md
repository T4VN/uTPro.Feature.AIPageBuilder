# Configuration

All settings live under the `uTPro:Feature:AIPageBuilder` section of `appsettings.json`. The feature
self-wires on install; you only need an endpoint, an API key and a model for planning to work.

## Minimal setup

```json
{
  "uTPro": {
    "Feature": {
      "AIPageBuilder": {
        "Enabled": true,
        "Provider": "OpenAiCompatible",
        "Endpoint": "https://api.openai.com/v1/",
        "ApiKey": "<your-api-key>",
        "Model": "gpt-4o-mini"
      }
    }
  }
}
```

> `ApiKey` is read server-side only and never reaches the browser. Keep real keys out of source
> control (use user secrets or environment variables in production). Every example here uses the
> `<your-api-key>` placeholder.

## Full reference

| Key | Type | Default | Meaning |
|---|---|---|---|
| `Enabled` | `bool` | `true` | Master switch. When `false` the feature is hidden in the backoffice. |
| `Provider` | `string` | `OpenAiCompatible` | Only `OpenAiCompatible` is built in. The endpoint is configurable, so OpenAI, Azure OpenAI, Gemini, Groq, OpenRouter and local Ollama all work unchanged. |
| `ApiKey` | `string?` | — | API key for the endpoint. Server-side only. |
| `Endpoint` | `string?` | OpenAI (`https://api.openai.com/v1/`) | Chat-completions base URL. |
| `Model` | `string?` | — | Model name requested from the endpoint (e.g. `gpt-4o-mini`, `gemini-2.0-flash`, `llama3.1:8b`). Required for planning. |
| `MaxInputChars` | `int` | `20000` | Upper bound on prompt characters accepted per request. |
| `MaxBlocks` | `int` | `50` | Maximum blocks a single plan may create or leave on the page, so a runaway prompt can't produce an unreviewable page. |
| `MaxBlocksPerCall` | `int` | `10` | Blocks sent to the model per call in Languages mode. Large pages split into several calls; a failed batch is skipped with a warning. |
| `AllowedPropertyEditors` | `string[]` | text/number/bool/picker editors | Property editor aliases the AI may fill. Editors outside the list are left untouched. |
| `AllowedDocTypeAliases` | `string[]` | `[]` (empty) | Document types the feature may act on. Empty means every document type that has a Block Grid / Block List property. |
| `ComponentHints` | `object` | `{}` (empty) | Per-component descriptions written for the AI, keyed by element type alias. Falls back to the element type's own description. |
| `AllowInsecureEndpoint` | `bool` | `false` | Allow plain `http://` endpoints on non-localhost hosts (e.g. LAN Ollama). Off by default because it lets API keys travel unencrypted. |
| `ImageGeneration` | `object` | off | Optional AI image generation to fill Media Picker properties. See below. |
| `VisionInput` | `object` | off | Optional design-image input ("build from a design image"). See below. |

### `ComponentHints`

Component aliases carry no meaning for a language model, so this is where you explain what each
component is for. Keys are element type aliases from your own site:

```json
"ComponentHints": {
  "heroBanner": "Hero banner at the top: big heading, short description, call to action.",
  "contactForm": "Contact form section, use at most once per page."
}
```

### `VisionInput`

Off by default because it needs a vision-capable model and sends uploaded screenshots to the LLM.
When enabled, the Build tab accepts design screenshots and the model reproduces the layout using
**only** the page's existing allowed components. Every limit is re-checked server-side.

```json
"VisionInput": {
  "Enabled": true,
  "MaxImages": 3,
  "MaxImageBytes": 4000000,
  "AllowedContentTypes": [ "image/png", "image/jpeg", "image/webp" ]
}
```

| Key | Type | Default | Meaning |
|---|---|---|---|
| `Enabled` | `bool` | `false` | Master switch. When `false`, any attached images are ignored (the plan runs text-only and records a warning). |
| `MaxImages` | `int` | `3` | Maximum images accepted per plan request. Extra images are dropped with a warning. |
| `MaxImageBytes` | `int` | `4000000` | Maximum decoded size per image, in bytes (roughly 4 MB), before base64 expansion. Larger images are dropped with a warning. |
| `AllowedContentTypes` | `string[]` | `image/png`, `image/jpeg`, `image/webp` | Image content types the model may receive. Others are dropped with a warning. |

> Point `Model` at a vision-capable model (for example `gemini-2.0-flash`) when `VisionInput:Enabled`
> is `true`. Without a vision model the provider rejects image requests.

### `ImageGeneration`

Off by default because every generated image costs money on the configured provider. When enabled,
the planner may fill Media Picker properties with newly generated images saved to a Media folder.

```json
"ImageGeneration": {
  "Enabled": true,
  "Endpoint": "https://api.openai.com/v1/",
  "ApiKey": "<your-api-key>",
  "Model": "gpt-image-1",
  "Size": "1024x1024",
  "MediaFolder": "Generated/AIPageBuilder",
  "MediaTypeName": "Image",
  "MaxImagesPerPlan": 10
}
```

`Endpoint`/`ApiKey` fall back to the core values when omitted; `Model` is required for generation.

## Provider presets

Any OpenAI-compatible chat-completions endpoint works — these are examples, not an allow-list.

| Provider | `Endpoint` | Example `Model` |
|---|---|---|
| OpenAI (default) | *(empty)* | `gpt-4o-mini` |
| Azure OpenAI | `https://<resource>.openai.azure.com/openai/deployments/<deployment>/` | your deployment name |
| Gemini (free tier, vision) | `https://generativelanguage.googleapis.com/v1beta/openai/` | `gemini-2.0-flash` |
| Groq | `https://api.groq.com/openai/v1/` | `llama-3.3-70b-versatile` |
| OpenRouter | `https://openrouter.ai/api/v1/` | `openai/gpt-4o-mini` |
| Ollama (local) | `http://localhost:11434/v1/` | `llama3.1:8b` |

The Gemini free tier is a convenient free, vision-capable option: it supports both text planning and
design-image input, and its base URL must end with `/openai/`.

> Endpoints must be `https://`. Plain `http://` is allowed only for localhost, or anywhere when
> `AllowInsecureEndpoint` is `true` (e.g. Ollama on a trusted LAN). Free tiers are rate-limited and
> may use requests for training — prefer a paid key or a local model for client content.

For the install-to-build walk-through see [getting-started.md](getting-started.md); for the request
and response contract see [api.md](api.md).
