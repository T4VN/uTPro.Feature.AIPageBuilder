# uTPro AI Page Builder for Umbraco

**Documentation:** <https://docs.utpro.dev/uTPro.Feature.AIPageBuilder/>

Build complete pages from your **existing components** with AI — and write every language variant for you. The **AI Builder** workspace action turns a plain prompt or topic into Block Grid content using only the components your page already allows, always as a reviewed preview and always saved as a **draft**. It also translates Dictionary items and Media captions, so one package covers the whole content AI workflow.

The AI never invents HTML, components, or keys: it plans against a server-built catalog of your page's allowed blocks, and every plan is validated against that catalog before anything is written.

## Features

- **Build** — describe the page (or just the topic: tone, audience, length) and the AI plans insert/update/remove operations over your existing Block Grid / Block List components.
- **Rewrite one block** — pick a block and ask for a rewrite; only that block changes.
- **Build from a design image** — attach a Figma/mock screenshot and a vision-capable model reproduces the layout using only your existing components. Off by default (`VisionInput`); needs a vision model.
- **Languages** — fill the current language variant from the source culture: translating and rewriting text, aligning block structures between cultures, and marking every difference for your review.
- **Dictionary & Media translation** — translate dictionary items and culture-variant media text with preview. Neither has a draft state, so both apply live behind an explicit confirmation.
- **Media sourcing & optional image generation** — the AI references media by name (resolved from your own library server-side) or, when explicitly enabled, generates new images into a configured Media folder.
- **Safe by construction** — component whitelist enforced server-side, prompts capped, block counts capped, draft-only saves for page content, per-mode authorization with node-level permission checks, endpoint guarded to https (localhost or explicitly allowed for LAN Ollama).

> **Limitation:** nested Block List / Block Grid values *inside* a component property are not filled or translated — only the page's own block editor property is.
- English and Vietnamese backoffice localizations.

## Install

```bash
dotnet add package uTPro.Feature.AIPageBuilder
```

The package self-wires: a composer registers the services, Management API and backoffice UI — no `Program.cs` changes. Supports **Umbraco 16, 17 and 18** (multi-targets `net9.0` for Umbraco 16 and `net10.0` for Umbraco 17 & 18).

> Works on any content model. Media and Dictionary translation run on **any** content type; **Build** needs a doc type with a Block Grid / Block List property. On a type without one, the AI Builder opens with the Build tab disabled (a friendly note) while Media/Dictionary translation keep working.

## Configuration

All settings live under `uTPro:Feature:AIPageBuilder`. For the full option-by-option reference (including the `VisionInput` and `ImageGeneration` blocks) see [docs/configuration.md](docs/configuration.md).

```json
{
  "uTPro": {
    "Feature": {
      "AIPageBuilder": {
        "Enabled": true,
        "Endpoint": "https://api.openai.com/v1/",
        "ApiKey": "<your-api-key>",
        "Model": "gpt-4o-mini",
        "MaxBlocks": 50,
        "ComponentHints": {
          "heroBanner": "Hero banner at the top: big heading, short description, call to action."
        },
        "ImageGeneration": {
          "Enabled": false
        }
      }
    }
  }
}
```

| Key | Meaning |
|---|---|
| `Endpoint` | Any OpenAI-compatible chat-completions base URL. Empty = OpenAI. |
| `ApiKey` | Server-side only; never sent to the browser. |
| `Model` | Model name served by your endpoint (required). |
| `MaxBlocks` / `MaxBlocksPerCall` | Plan size caps; large variant runs are batched. |
| `MaxInputChars` | Prompt length cap. |
| `AllowedPropertyEditors` | Editors the AI may fill (text/number/bool/pickers by default). |
| `AllowedDocTypeAliases` | Empty = every document type with a block editor. |
| `ComponentHints` | Per-component descriptions for the AI, keyed by element type alias. Falls back to the element type's description. |
| `AllowInsecureEndpoint` | Allow plain http on non-localhost (e.g. LAN Ollama). Off by default. |
| `ImageGeneration` | Off by default. `Endpoint`/`ApiKey`/`Model`/`Size`, `MediaFolder`, `MediaTypeName`, `MaxImagesPerPlan`. |
| `VisionInput` | Off by default. Build-from-image: `Enabled`, `MaxImages`, `MaxImageBytes`, `AllowedContentTypes`. Needs a vision model. |

### Zero-cost setups

The client speaks the OpenAI-compatible protocol, so all of these work without code changes:

```jsonc
// Gemini free tier
{ "Endpoint": "https://generativelanguage.googleapis.com/v1beta/openai/", "Model": "gemini-2.0-flash" }

// Groq
{ "Endpoint": "https://api.groq.com/openai/v1/", "Model": "llama-3.3-70b-versatile" }

// Local Ollama (free + private; localhost http is always allowed)
{ "Endpoint": "http://localhost:11434/v1/", "Model": "llama3.1:8b" }
```

Free tiers are rate-limited and (for some providers) may use your data for training — for client content prefer a paid key or a local model.

## Usage

- **Documents** — the **AI Builder** button in the workspace footer opens two tabs:
  - *Build*: write a prompt, optionally scope it to a single block or attach a design image (when `VisionInput` is enabled), Generate, review the operation list, **Apply as draft**.
  - *Languages*: switch the editor to the target variant, pick the source culture, Generate, review the diff (including blocks that exist in only one culture), **Apply as draft**.
- **Media** — the **AI Translate** button translates culture-variant text fields of the current media item. Media has no draft state, so applied values are live immediately.
- **Dictionary** — the **AI Translate** button plans translations for the whole dictionary; applying is **live** and requires an explicit confirmation checkbox.

## Security

- The API key stays server-side; the backoffice talks only to this package's authenticated Management API (`/umbraco/management/api/v1/utpro/ai-page-builder/`, POST-only, per-mode authorization).
- Plans are validated against the server-built component catalog: unknown components, unknown properties and non-AI-fillable editors are dropped before anything reaches Umbraco.
- Pickers never receive AI-invented keys — references are resolved server-side from your own content/media or left empty with a warning.
- Page content apply is draft-only (`Save`, never publish); media and dictionary have no draft state and apply live, each gated behind a UI confirmation.
- Every call is authorized per mode with node-level permission checks (Content, Media, or the Dictionary tree), so AI actions can never exceed the user's own permissions.
- Endpoints must be https unless localhost, or `AllowInsecureEndpoint` is explicitly enabled.

## Privacy

Prompts, the component catalog and the text you ask to translate are sent to the LLM endpoint you configure. Nothing is sent anywhere else. Review your provider's data policy — free tiers of some providers use requests for training. Choose a local Ollama endpoint when content must not leave your server.

## License

MIT — see [LICENSE](LICENSE).
