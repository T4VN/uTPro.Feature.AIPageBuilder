# API Reference

The AI Page Builder exposes three Management API endpoints under
`/umbraco/management/api/v1/utpro/ai-page-builder/`. Every endpoint is **POST** and served by
`ManagementApiControllerBase` behind backoffice bearer-token auth. Authorization is enforced
**per mode** with node-level permission checks, so a user can only act on items they already have
access to:

| Mode | Section / tree | Node permission |
|---|---|---|
| `build` / `variants` plan | Content | `Browse` on the node |
| `build` / `variants` apply | Content | `Update` on the node |
| `media` plan / apply | Media | access to the media node |
| `dictionary` plan / apply | Dictionary tree | — |

Plans are previews — nothing is written until you call `apply`, which re-validates server-side.
Content is saved as a draft; media and dictionary have no draft state and save live (the UI
confirms before applying).

For installation and a walk-through, see [getting-started.md](getting-started.md).

## Endpoints

| Method | Route | Body | Result |
|---|---|---|---|
| POST | `/catalog/{key}?culture={culture}` | *(empty POST)* | [`PageCatalogResponse`](#pagecatalogresponse) |
| POST | `/plan` | [`PlanRequest`](#planrequest) | [`PagePlan`](#pageplan) or [`TextTranslationPlan`](#texttranslationplan) |
| POST | `/apply` | [`ApplyRequest`](#applyrequest) | [`ApplyResult`](#applyresult) |

- `/catalog/{key}` accepts an optional `?culture=` query to read the structure for a specific
  variant; it defaults to the site's default language for variant content.
- **Graceful degrade:** opening the catalog on a content type **without** a Block Grid / Block List
  property no longer errors. `/catalog/{key}` returns `200` with `hasBlockEditor: false` and an empty
  `catalog`/`structure`, so Media / Dictionary translation and design-image input stay available in
  the backoffice while only Build block-planning is disabled. Actually calling `/plan` or `/apply`
  in `build`/`variants` mode on such a type returns a clean `400` `ProblemDetails`.
- `/plan` returns a `PagePlan` for `build`/`variants` modes and a `TextTranslationPlan` for
  `dictionary`/`media` modes.
- Errors return a `ProblemDetails` body (`title` carries the message) with status `400`
  (bad request / unreadable model reply), `403` (insufficient permission) or `500` (unexpected
  failure). Provider error details and stack traces are logged server-side, never returned to the
  browser.

## PlanRequest

Body for `POST /plan`.

| Field | Type | Description |
|---|---|---|
| `key` | `guid` | Key of the content or media item to plan for. Required for `build`/`variants`/`media`; **optional for `dictionary`** (it targets the whole dictionary). |
| `mode` | `enum` | Which pipeline to run: `build`, `variants`, `dictionary` or `media`. Defaults to `build`. |
| `prompt` | `string?` | User prompt (Build mode). |
| `targetBlockKey` | `guid?` | When set, Build mode rewrites only this block. |
| `sourceCulture` | `string?` | Source culture for Variants mode; defaults to the site's default language. |
| `targetCulture` | `string?` | Target culture for Variants / Dictionary / Media modes. |
| `images` | `string[]?` | **Build mode only.** Design screenshots the vision model reproduces using **only** the page's catalog components. Each item is a data URL (`data:image/png;base64,...`). Ignored unless `uTPro:Feature:AIPageBuilder:VisionInput:Enabled` is `true` server-side; the server re-validates count, content type and size and drops anything outside the limits with a warning. See [getting-started.md § Build from a design image](getting-started.md#build-from-a-design-image). |

## ApplyRequest

Body for `POST /apply`. Send back the plan you previewed; the server re-validates before touching anything.

| Field | Type | Description |
|---|---|---|
| `mode` | `enum` | Which pipeline produced the plan: `build`, `variants`, `dictionary` or `media`. Defaults to `build`. |
| `pagePlan` | [`PagePlan`](#pageplan)`?` | Page plan for Build / Variants modes. |
| `textPlan` | [`TextTranslationPlan`](#texttranslationplan)`?` | Text translation plan for Dictionary / Media modes. |

## PagePlan

A validated, previewable set of changes for one content item. Produced by the planner, cleaned by
the validator, applied (draft-only) by the applier.

| Field | Type | Description |
|---|---|---|
| `itemKey` | `guid` | Key of the content/media/dictionary item this plan targets. |
| `mode` | `enum` | Pipeline that produced this plan. |
| `culture` | `string?` | Culture the plan writes to (content/media modes). |
| `sourceCulture` | `string?` | Source culture for variant plans. |
| `operations` | [`PagePlanOperation[]`](#pageplanoperation) | Ordered operations to apply. |
| `warnings` | `string[]` | Non-fatal notes collected while planning/validating. |

### PagePlanOperation

One change a plan makes to the page. Property values are raw JSON scalars or reference objects
(`{"$ref": ...}`, `{"$image": ...}`) interpreted by the validator and applier.

| Field | Type | Description |
|---|---|---|
| `op` | `enum` | `insert`, `update` or `remove`. |
| `componentAlias` | `string` | Element type alias of the component, e.g. `heroBanner`. |
| `blockKey` | `guid?` | Required for `update`/`remove` (an existing block). On `insert` it is an optional **plan-local** reference: later inserts can nest into it via `parentBlockKey`; the stored block always gets a fresh key. |
| `sourceBlockKey` | `guid?` | Variants inserts only: the source-culture block this insert was translated from. Its non-text values and settings are copied so the new block matches the source. |
| `rootIndex` | `int?` | Zero-based position at the root for inserts; appended when null. |
| `parentBlockKey` | `guid?` | Key of the parent block (or a plan-local insert key) when inserting into one of its areas. |
| `areaAlias` | `string?` | Area alias inside the parent block, used with `parentBlockKey`. |
| `properties` | `object` | Property values keyed by property alias; scalars or reference objects. |
| `originalText` | `string?` | Source-culture text kept alongside translated values for the preview diff. |
| `note` | `string?` | Short human-readable explanation, shown in the preview. |

> Picker values are never supplied as keys by the client. Use a reference placeholder and the
> server resolves it: `{"$ref":{"type":"content","name":"About Us","keywords":"about"}}` or
> `{"$ref":{"type":"media","keywords":"team photo"}}`. When image generation is enabled, a media
> picker may instead use `{"$image":{"prompt":"...","alt":"..."}}`.

## TextTranslationPlan

A previewable set of text translations for Dictionary or Media items. Neither has a draft concept,
so apply saves live for both (`savedAsDraft: false`); the UI confirms before calling.

| Field | Type | Description |
|---|---|---|
| `itemKey` | `guid` | Key of the media item for a Media plan; empty for Dictionary plans. |
| `targetCulture` | `string` | Target culture of the translations. |
| `sourceCulture` | `string?` | Source culture of the translations. |
| `entries` | [`TextTranslationEntry[]`](#texttranslationentry) | Entries to write. |
| `warnings` | `string[]` | Non-fatal notes collected while planning. |

### TextTranslationEntry

| Field | Type | Description |
|---|---|---|
| `key` | `string` | Dictionary item key, or media property alias for Media plans. |
| `displayName` | `string` | Display name shown in the preview (dictionary name / property name). |
| `sourceText` | `string?` | Source-culture text. |
| `translatedText` | `string?` | Translated text written on apply. |

## ApplyResult

Result of applying a plan. Content saves as a draft (`savedAsDraft: true`); media and dictionary
have no draft state and save live (`savedAsDraft: false`), so the UI confirms before apply.

| Field | Type | Description |
|---|---|---|
| `success` | `bool` | Whether the plan applied cleanly (warnings may still be present). |
| `itemKey` | `guid` | Key of the item that was written to. |
| `appliedOperations` | `int` | Number of operations applied. |
| `savedAsDraft` | `bool` | Whether the result was saved as a draft (no publish). |
| `warnings` | `string[]` | Non-fatal notes (unresolved references, skipped batches, ...). |

## PageCatalogResponse

Returned by `POST /catalog/{key}`: the component catalog plus the page's current root structure and
default write culture, for the backoffice structure view and single-block rewrite picker. Also
carries two capability flags (`hasBlockEditor`, `visionInputEnabled`) the backoffice uses to decide
which controls to enable.

| Field | Type | Description |
|---|---|---|
| `catalog` | `ComponentCatalog` | Components the AI may use, derived from the block editor configuration. Empty when `hasBlockEditor` is `false`. |
| `structure` | `StructureItem[]` | Current blocks (root and nested), depth-first. Empty when `hasBlockEditor` is `false`. |
| `culture` | `string?` | Culture the structure was read for and plans write to (null for invariant content). |
| `variesByCulture` | `bool` | Whether the document varies by culture. Languages mode only applies when true. |
| `hasBlockEditor` | `bool` | `false` when the content type has **no** Block Grid / Block List property. The call still returns `200` with an empty `catalog`/`structure` (graceful degrade); the backoffice then shows the Build block-planning UI disabled while Media / Dictionary translation and the design-image control keep working. |
| `visionInputEnabled` | `bool` | Whether design-image input is enabled (`uTPro:Feature:AIPageBuilder:VisionInput:Enabled`). The backoffice only shows the image-upload control when this is `true`. |

### StructureItem

| Field | Type | Description |
|---|---|---|
| `blockKey` | `guid` | Block key from the block value JSON. |
| `componentAlias` | `string` | Element type alias of the block. |
| `rootIndex` | `int` | Zero-based position of the block's root ancestor. |
| `depth` | `int` | Nesting depth: 0 for root blocks, 1 for blocks in a root block's area, ... |
| `parentBlockKey` | `guid?` | Key of the block whose area holds this block; null for root blocks. |
| `areaAlias` | `string?` | Alias of the parent's area holding this block; null for root blocks. |
| `textPreview` | `string?` | Concatenated text values (truncated, HTML stripped) for display. |
