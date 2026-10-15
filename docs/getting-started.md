# Getting Started

Install → configure an LLM endpoint → build a page from a prompt → fill language variants.

## 1. Install

```bash
dotnet add package uTPro.Feature.AIPageBuilder
```

Supports Umbraco 16, 17 and 18 (multi-targets `net9.0` for Umbraco 16 and `net10.0` for Umbraco 17 & 18). The package self-wires: composer, Management API and backoffice UI — no `Program.cs` changes.

## 2. Configure the LLM endpoint

Add the section to `appsettings.json`:

```json
{
  "uTPro": {
    "Feature": {
      "AIPageBuilder": {
        "Enabled": true,
        "Endpoint": "https://api.openai.com/v1/",
        "ApiKey": "<your-api-key>",
        "Model": "gpt-4o-mini"
      }
    }
  }
}
```

Zero-cost provider presets (any OpenAI-compatible endpoint works):

| Provider | Endpoint | Example model |
|---|---|---|
| OpenAI (default) | *(empty)* | `gpt-4o-mini` |
| Azure OpenAI | `https://<resource>.openai.azure.com/openai/deployments/<deployment>/` | your deployment name |
| Gemini free tier | `https://generativelanguage.googleapis.com/v1beta/openai/` | `gemini-2.0-flash` |
| Groq | `https://api.groq.com/openai/v1/` | `llama-3.3-70b-versatile` |
| OpenRouter | `https://openrouter.ai/api/v1/` | `openai/gpt-4o-mini` |
| Ollama (local) | `http://localhost:11434/v1/` | `llama3.1:8b` |

Any OpenAI-compatible chat-completions endpoint works — these presets are examples, not an allow-list.
For the full option list (including `VisionInput` and `ImageGeneration`) see [configuration.md](configuration.md).

Notes:

- Endpoints must be `https://` — plain `http://` is allowed only for localhost, or anywhere with `"AllowInsecureEndpoint": true` (e.g. Ollama on a trusted LAN).
- Free tiers are rate-limited and may use requests for training; prefer a paid key or local models for client content.
- Image generation is a separate, optional section (`ImageGeneration`) and is **off by default** — each generated image costs API money.

## 3. Teach the AI your components

The planner reads the component catalog from each page's Block Grid / Block List configuration at runtime — nothing to register. What it cannot know is what each component is *for*. Add hints for the components that matter:

```json
"ComponentHints": {
  "heroBanner": "Hero banner at the top: big heading, short description, call to action.",
  "contactForm": "Contact form section, use at most once per page."
}
```

Components without a hint fall back to their element type description. Property names the AI may fill are derived from the editors: text, number, boolean and picker editors (see `AllowedPropertyEditors`); anything else is left untouched.

## 4. Build a page

1. Create (or open) a node whose doc type has a Block Grid / Block List property. Build needs a block editor property; Media and Dictionary translation (section 6) work on **any** content type. On a doc type with no block editor, the AI Builder opens with the Build tab disabled and a note, while Media/Dictionary translation keep working.
2. Click **AI Builder** in the workspace footer.
3. In **Build**, write a prompt — page description or just a topic with tone/audience/length.
4. Click **Generate**, review the plan (operations, values, notes), click **Apply as draft**.
5. Review the content in the editor and publish.

**Rewrite a single block:** switch *Scope* to *Rewrite one block*, pick the block, write what to change, Generate → Apply. Only that block is touched.

### Build from a design image

Attach a design screenshot (exported from Figma or any mock) and a vision-capable model reproduces the layout using **only the components the page already allows** — it never invents components, keys, HTML, or areas.

This is **off by default** because it needs a vision model. Enable it and point `Model` at a vision-capable model (for example `gemini-2.0-flash`):

```json
"AIPageBuilder": {
  "Model": "gemini-2.0-flash",
  "VisionInput": {
    "Enabled": true,
    "MaxImages": 3,
    "MaxImageBytes": 4000000,
    "AllowedContentTypes": [ "image/png", "image/jpeg", "image/webp" ]
  }
}
```

Then in **Build**: write an optional caption, click **Attach design image(s)**, pick up to `MaxImages` PNG/JPEG/WebP files (each up to `MaxImageBytes` before base64 expansion), **Generate** → review → **Apply as draft**.

The server re-validates every image (count, content type, size) and never trusts the client. Images outside the limits are dropped with a warning and the plan still runs on the text prompt. With vision disabled, any attached image is ignored and the plan records a warning. Image input applies to **Build** mode only.

## 5. Fill language variants

1. Switch the editor to the target language variant (top-right selector).
2. **AI Builder → Languages**. Pick the **source** culture (default: the site's default language) and the **target** culture (default: the open variant). Source and target must differ.
3. **Generate** — the plan updates matched blocks with translated text, inserts blocks missing in the target culture (including blocks nested in a container's area), and flags target blocks with no counterpart for removal.
4. Review the diff and **Apply as draft**.

> The Languages tab is only available on documents that vary by culture; an invariant document shows a note instead. Nested Block List / Block Grid values *inside* a component property are not filled or translated — only the page's own block editor is.

## 6. Translate Dictionary & Media

- **Media workspace → AI Translate**: translates culture-variant text fields of the current item. Media has no draft state, so applied values are **live immediately**.
- **Dictionary workspace → AI Translate**: plans the whole dictionary for a target culture. Applying is **live** (no draft state) — the dialog requires an explicit confirmation.

## 7. Extensibility

| Option | Purpose |
|---|---|
| `ComponentHints` | Per-component descriptions for the AI (keyed by element type alias). |
| `AllowedPropertyEditors` | Which editors the AI may fill. Pickers are filled via server-resolved references only. |
| `AllowedDocTypeAliases` | Restrict the feature to specific document types. |
| `MaxBlocks`, `MaxBlocksPerCall`, `MaxInputChars` | Cost/size guardrails. |
| `ImageGeneration` | Optional AI images: endpoint/key/model, `MediaFolder`, `MediaTypeName`, `MaxImagesPerPlan`. |
| `VisionInput` | Optional "build from a design image": `Enabled`, `MaxImages`, `MaxImageBytes`, `AllowedContentTypes`. Needs a vision model. |

API surface (Management API, POST-only, backoffice auth):

| Route | Body | Result |
|---|---|---|
| `/umbraco/management/api/v1/utpro/ai-page-builder/catalog/{key}` | *(empty POST)* | `PageCatalogResponse` (catalog + structure + culture) |
| `/umbraco/management/api/v1/utpro/ai-page-builder/plan` | `PlanRequest` | `PagePlan` (build/variants) or `TextTranslationPlan` (dictionary/media) — preview only |
| `/umbraco/management/api/v1/utpro/ai-page-builder/apply` | `ApplyRequest` | `ApplyResult` |

All state-changing calls are POST, authorized per mode with node-level permission checks (Content for build/variants, Media for media, the Dictionary tree for dictionary). Content applies as a draft; media and dictionary apply live behind the UI confirmation. See [api.md](api.md) for the field-by-field request/response reference.

## 8. Troubleshooting

Errors surface a short message; full provider detail is logged server-side only. Common cases:

| Symptom | Likely cause |
|---|---|
| HTTP 401 / 403 from the AI endpoint | Wrong or missing `ApiKey`. |
| HTTP 404 from the AI endpoint | Wrong `Endpoint` or `Model`. For Gemini the base URL must end with `/openai/`. |
| HTTP 429 from the AI endpoint | Rate limited — retry later or lower `MaxBlocksPerCall`. |
| "Could not reach the AI endpoint" | Network/DNS/firewall; the endpoint host is unreachable. |
| Empty source-culture list in Languages | The site has no configured languages, or the backoffice could not reach the language API. |

> Verify the Gemini example model name against the current list at
> <https://ai.google.dev/gemini-api/docs/models>; model availability changes over time.
