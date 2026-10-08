# Changelog

All notable changes to `asignua/filament-rich-links` are documented here.

## v1.0.2 - 2026-10-08

- Fix: the toolbar button no longer lights up on handler-only markers (an eEgnith `/ee-file/` link) that the dialog cannot edit; submitting it would have removed the link.
- Fix: the picker's ordering and default title search are qualified with the table name, so a joined `LinkTarget::query()` no longer fails with an ambiguous-column error.
- `scripts/` is export-ignored and no longer ships in the dist.

## v1.0.1 - 2026-10-06

First public release, sold through Anystack. The `v1.0.0` tag (2026-10-02) was an internal pre-release:
it predates the review fixes listed below and still carried the draft licence.

- Commercial licence v1.0 (`LICENSE.md`): Single Project and Unlimited tiers, one year of updates, 14-day refunds,
  Polish law. Sold by Artem Pipenko under an exclusive licence from the author.
- Installation from the private repository `https://filament-rich-links.composer.sh` (README).

- RichEditor toolbar button "Internal link" (`RichLinksPlugin`) and its dialog (`RichLinkAction`): pick a record type and a record, optional link text and "open in new tab". Insert, wrap a selection, update the link under the cursor, or remove it.
- The link stores the record, not a URL: a marker `{prefix}{type}/{id}` in the `href` (the TipTap `link` mark keeps nothing else). The prefix, the button name and the form field keys are configurable.
- `RichLinks::resolve($html, $locale)` replaces the markers with the address of the language at render time. A target that is deleted, unpublished or has no address in the language turns the link into plain text. Foreign and absolute URLs are never touched.
- One DOM pass for every kind of marker: `HtmlLinkRewriter` and the `SentinelHandler` contract, with an early exit when the HTML holds no marker.
- Resolvers: `ModelLinkResolver` (models implementing `HasRichLink`) and `TableLinkResolver` (a slug-per-language table with configurable columns, URL builder and visibility rule), both batching per type. An optional `rich_links` migration and `RichLinkSlugs::sync()`.
- Picker targets: `LinkTarget::model()` with a custom label, base query, search and option label; the default search covers every language of a translatable title column.
- Extra link kinds (`LinkKind`) for targets that are not records, with their own fields and handler.
- Optional redirect route for markers that leaked past the renderer; the language comes strictly from the URL.
- Translations: English, Ukrainian, German, Spanish, French, Italian, Dutch, Polish, Brazilian Portuguese and Turkish.
- Laravel Boost guidelines.
- `LinkTarget::title()` customises the link text inserted when the editor leaves it empty.
- Resolved addresses are checked before they are written into `href` or redirected to (`SafeUrl`): only a root-relative path (not `//…` or `/\…`) or an absolute `http(s)` URL passes; anything else unwraps the link and the fallback route answers 404. `RichLinks::localizedUrl()` strips leading backslashes and control characters from the slug, and `RichLinks::urlFor()` returns `null` for an unsafe address.
- A marker type is untrusted (it comes from stored content and the public redirect route): abstract models, models whose constructor needs arguments and non-model classes are refused by `ModelTypes::modelClass()`, so they give a 404 or plain text, never a 500 — also with an enforced morph map, where a model outside the map is skipped by `TableLinkResolver` instead of throwing `ClassMorphViolationException`.
- In a panel with tenancy every `LinkTarget` is scoped to the current tenant (`->tenantOwnershipRelationship()`, `->scopeToTenant(false)`); the dialog rejects a submitted id that is outside the target's query (`RichLinks::isPickable()`).
- Markers store the model's morph class (`RichLinks::markerType()`), so with a morph map they survive a renamed class; markers holding the class name keep working, and `TableLinkResolver` resolves a model asked for under both spellings in one render.
- The dialog preselects the type when only one is registered, explains an empty registry and flags a link whose type is no longer registered.
- The default search matches `%` and `_` literally. `RichLinkSlugs::sync()` writes with one upsert (the table needs the unique key of the published migration). The migration creates the table `RichLinks::getTable()` names, so a name set in code with `RichLinks::table()` is honoured, and its unique index is named after the table.
- `HtmlLinkRewriter` keeps the content after a stray closing tag and writes foreign `href`/`src` values and `&nbsp;` back verbatim instead of letting libxml percent-encode them.
- `RichLinks::resolve()` reads the application locale only once a marker is found, so HTML with no marker needs no application; `HtmlLinkRewriter::rewrite()` takes a nullable locale.
- The fallback route url-decodes the type once (as Laravel does), not twice.
