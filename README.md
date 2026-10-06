# Filament Rich Links

> **Documentation only.** Filament Rich Links is a commercial plugin: this public repository holds its
> documentation, changelog and licence terms. The package itself is installed from the private Composer
> repository you get with a licence — [buy one on Anystack](https://checkout.anystack.sh/filament-rich-links).
> Questions: support@asign.in.ua.

[![Stand With Ukraine](https://raw.githubusercontent.com/vshymanskyy/StandWithUkraine/main/badges/StandWithUkraine.svg)](https://stand-with-ukraine.pp.ua)

<img class="filament-hidden" src="https://raw.githubusercontent.com/asignua/filament-rich-links-docs/main/art/cover.jpg" alt="Filament Rich Links">

Internal links for the [Filament](https://filamentphp.com) 5 `RichEditor`: the editor **picks a record**, the content stores
the **record**, not a URL, and the address is filled in when the page is rendered.

Filament's rich editor links to URLs only. An editor who wants to link to another page of the site has to paste an
address, and that address silently rots: someone renames the slug, unpublishes the page, or the site gets a second
language, and the link is dead. This plugin fixes the cause instead of the symptom:

- **IDs, not URLs.** The saved `href` is a marker (`/rich-link/post/42` with a morph map, `/rich-link/App%5CModels%5CPost/42`
  without one). A changed slug never breaks a link; with an enforced morph map a renamed or moved model class does not either.
- **Unpublished targets become text.** A deleted, hidden or not-yet-translated target is replaced by its link text at render
  time, so the site never serves a link to a 404.
- **Per language.** The marker is resolved for the language of the page being rendered; a target that has no address in that
  language disappears from that language's page only.
- **Cheap.** One DOM pass for all markers, one query per target type, an early exit when the HTML has none.
- **Safe.** Absolute and protocol-relative URLs are foreign links and are never touched, even when their path looks like a
  marker. A resolved address is written only when it is a root-relative path or an `http(s)` URL — a `javascript:` or
  `//other-host` value from a resolver loses the link instead. A marker naming a class that is not an instantiable Eloquent
  model is never loaded; with the default resolver the class must also implement `HasRichLink`, with the
  [table resolver](#table-resolver) it must have rows in the slug table (which only your code writes).

- [Purchase](#purchase)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Table resolver](#table-resolver)
- [Targets and search](#targets-and-search)
- [Extra link kinds](#extra-link-kinds)
- [Rendering](#rendering)
- [Fallback route](#fallback-route)
- [Multilingual sites](#multilingual-sites)
- [Configuration](#configuration)
- [Gotchas](#gotchas)
- [Translations](#translations)
- [AI agents](#ai-agents)
- [Testing](#testing)
- [Screenshots](#screenshots)

## Purchase

Filament Rich Links is a commercial plugin. Both tiers include one year of updates; after that the
plugin keeps working on the last release you received, and you can renew for further updates.

| Tier | Projects | Activations | Price | Renewal |
| --- | --- | --- | --- | --- |
| Single Project | 1 | up to 3 (production, staging, local) | €49 | €25 / year |
| Unlimited | any number, SaaS included | unlimited | €119 | €59 / year |

Refunds are available within 14 days of purchase.

**[Buy a licence on Anystack](https://checkout.anystack.sh/filament-rich-links)** — after the purchase you receive a licence key and
access to the private Composer repository. See [LICENSE.md](https://github.com/asignua/filament-rich-links-docs/blob/main/LICENSE.md) for the licence terms.

## Requirements

- PHP 8.3+ with the `dom` and `libxml` extensions
- Laravel 12 or 13
- Filament 5
- Optional: [`spatie/laravel-translatable`](https://github.com/spatie/laravel-translatable), so the picker can search translatable
  titles in every language

## Installation

1. **Add the private repository** (the same for every customer):

   ```bash
   composer config repositories.filament-rich-links composer https://filament-rich-links.composer.sh
   ```

2. **Add your credentials.** The username is the e-mail address you bought the licence with; the
   password is your licence key. A **Single Project** licence is bound to a fingerprint, so the password
   is the key followed by a colon and the fingerprint you activated — usually the production domain:

   ```bash
   # Single Project
   composer config --auth http-basic.filament-rich-links.composer.sh you@example.com "LICENCE-KEY:shop.example.com"

   # Unlimited
   composer config --auth http-basic.filament-rich-links.composer.sh you@example.com "LICENCE-KEY"
   ```

   This writes `auth.json` next to your `composer.json` — keep it out of git (add `auth.json` to `.gitignore`).
   On CI and servers pass the same JSON through the `COMPOSER_AUTH` environment variable instead.
   A Single Project licence allows three activations (for example production, staging and local); manage
   them in your Anystack account.

3. **Require the package:**

   ```bash
   composer require asignua/filament-rich-links
   ```

Publish the config file if you want to change a default (everything in it can also be set in code):

```bash
php artisan vendor:publish --tag=filament-rich-links-config
```

## Quick start

**1. Tell the plugin which records can be linked to.** Put it in `AppServiceProvider::boot()`:

```php
use Asignua\FilamentRichLinks\RichLinks;
use Asignua\FilamentRichLinks\Support\LinkTarget;

RichLinks::targets([
    LinkTarget::model(Post::class)->label('Posts')->titleColumn('title'),
]);
```

**2. Make the model say where it lives.** The default resolver asks the model:

```php
use Asignua\FilamentRichLinks\Contracts\HasRichLink;

class Post extends Model implements HasRichLink
{
    public function richLinkUrl(string $locale): ?string
    {
        return route('posts.show', $this); // or null: "no address in this language"
    }

    public function richLinkVisible(string $locale): bool
    {
        return $this->published_at?->isPast() ?? false;
    }
}
```

**3. Put the button in the editor.**

```php
use Asignua\FilamentRichLinks\RichLinks;
use Asignua\FilamentRichLinks\RichLinksPlugin;

RichEditor::make('body')
    ->plugins([RichLinksPlugin::make()])
    ->toolbarButtons([['bold', 'italic', 'link', RichLinks::getToolName()]]);
```

To give every editor in the application the plugin, use `RichEditor::configureUsing(fn (RichEditor $editor) => $editor->plugins([RichLinksPlugin::make()]))`
and add the button name to the toolbars you want it in.

**4. Sanitize, then resolve the markers when you render** — see [Rendering](#rendering):

```php
{!! RichLinks::resolve(RichContentRenderer::make($post->body)->toHtml()) !!}
```

`toHtml()` runs Filament's HTML sanitizer; never pass raw editor content to `{!! !!}`. `RichLinks::resolve()` does not sanitize.

The stock "Link" button stays; use it for external URLs.

## Table resolver

When the address is not something the model can compute — you already keep a table of slugs, one row per record and
language — resolve through the table instead:

```php
use Asignua\FilamentRichLinks\Resolvers\TableLinkResolver;

RichLinks::resolveUsing(
    TableLinkResolver::make()
        ->table('links')
        ->columns(type: 'entity_type', id: 'entity_id', locale: 'language', slug: 'slug')
        ->href(fn (string $locale, string $slug): string => localized_url($locale, $slug))
        ->visible(fn (Model $model, string $locale): bool => $model->isPublished($locale)),
);
```

- `href()` builds the URL from the language and the stored slug. Default: `/{slug}`, and `/{locale}/{slug}` for every language
  except the unprefixed one (see [Multilingual sites](#multilingual-sites)).
- `visible()` is the visibility rule. Default: the model's own `isVisible($locale)` when it has one, otherwise visible.
- The type column may hold a class name or a morph alias; both are understood.

The plugin runs **one query** when visibility does not matter (menus, banners) and **two queries per target type** (the table
and the models) when it does (rendered content), however many links the page holds.

**No such table?** Publish the optional migration and keep it filled:

```bash
php artisan vendor:publish --tag=filament-rich-links-migrations
php artisan migrate
```

```php
use Asignua\FilamentRichLinks\Support\RichLinkSlugs;

RichLinks::resolveUsing(TableLinkResolver::make()); // the defaults match the rich_links table

// whenever a model is saved / deleted:
RichLinkSlugs::sync($page, ['en' => 'about', 'uk' => 'pro-nas']); // a language that is missing or empty is removed
RichLinkSlugs::forget($page);
```

## Targets and search

Every picker entry is a `LinkTarget`:

```php
LinkTarget::model(Page::class)
    ->label('Pages')                                       // the entry in the type select
    ->titleColumn('title')                                 // default: title
    ->query(fn (Builder $query) => $query->where('team_id', auth()->user()->team_id))
    ->search(fn (Builder $query, string $term) => $query->where('title', 'like', "%{$term}%")->orWhere('slug', 'like', "%{$term}%"))
    ->optionLabel(fn (Model $page, string $term): string => "{$page->title} ({$page->slug})");
```

By default the picker searches the title column with `LIKE` (the term is matched literally: `%` and `_` are not wildcards), newest first, 50 at a time (`search_limit` in the config). When the
column is declared translatable by `spatie/laravel-translatable` it is searched in **every** language of
`RichLinks::locales()`, so a query in English finds a record whose current-language title is Ukrainian; the option then says
which language matched (`[uk] Онлайн-курси · [en] Online courses`).

### Tenancy and access

The picker lists the titles of every record its query returns, drafts included, and the record field rejects a submitted id
that is outside that query. In a panel **with tenancy** the query is scoped to the current tenant by default, through the
panel's ownership relationship — the same rule Filament applies to resources:

```php
LinkTarget::model(Page::class)->tenantOwnershipRelationship('team');   // when the relationship has another name
LinkTarget::model(Country::class)->scopeToTenant(false);               // a model shared by every tenant
```

A target with no relationship to the tenant throws a `LogicException` rather than leaking another tenant's titles. Anything
finer — drafts, a policy, a role — goes into `->query()`: every lookup of the dialog (search, the label of the chosen record,
the check of a submitted id) goes through it.

The tenant scope applies to the **picker** only. Rendering and the fallback route resolve whatever marker the HTML holds, so a
marker for another tenant's record (content copied between tenants, hand-edited HTML) still becomes a link whenever that record
is visible. If a site needs tenant isolation at render time, put the rule into `visible()` or the model's `richLinkVisible()`.

When the editor leaves the link text empty, the title of the record is inserted — without a language marker, so a `[en] `
label never ends up in published content.

## Extra link kinds

A link target that is not a record (the value of a setting, a phone number from the site config…) is an extra *kind*. It
appears in the same type select, brings its own fields, and writes its own `href`:

```php
use Asignua\FilamentRichLinks\Support\LinkKind;

RichLinks::kind(
    LinkKind::make('setting')
        ->label('Setting')
        ->fields(fn (): array => [Select::make('setting_key')->options(fn () => Setting::linkable()->pluck('key', 'key'))->searchable()])
        ->encode(fn (array $data): ?string => filled($data['setting_key'] ?? null) ? '/setting/'.rawurlencode($data['setting_key']) : null)
        ->decode(fn (string $href): ?array => str_starts_with($href, '/setting/') ? ['setting_key' => rawurldecode(substr($href, 9))] : null)
        ->defaultText(fn (array $data, string $href): ?string => Setting::valueOf($data['setting_key']))
        ->handler(new SettingLinkHandler),
);
```

- the fields are shown only while the kind is chosen, and the record picker is hidden then;
- `encode()` returning `null` means "nothing chosen" and removes the link;
- `decode()` pre-fills the dialog when an existing link of this kind is edited;
- `handler()` takes part in the render pass — see below.

A marker type that needs no entry in the dialog (a link to a file from your own file picker, say) is just a handler:

```php
use Asignua\FilamentRichLinks\Contracts\SentinelHandler;

final class SettingLinkHandler implements SentinelHandler
{
    public function prefix(): string { return '/setting/'; }
    public function collect(string $path): bool { /* remember the key; false = broken marker */ }
    public function resolveBatch(string $locale): void { /* one lookup for everything collected */ }
    public function replace(DOMElement $anchor, string $path): void { /* setAttribute('href', …) or HtmlLinkRewriter::unwrap($anchor) */ }
    public function reset(): void { /* forget the state of the render */ }
}

RichLinks::handler(new SettingLinkHandler);
```

All handlers share **one** DOM pass with the plugin's own: `collect()` is called per anchor under your prefix (only relative
paths ever reach it), `resolveBatch()` once if anything valid was collected, then `replace()` per anchor. `reset()` runs before
and after each render. `HtmlLinkRewriter::unwrap()` removes the link and keeps its text.

## Rendering

```php
use Filament\Forms\Components\RichEditor\RichContentRenderer;

{!! RichLinks::resolve(RichContentRenderer::make($post->body)->toHtml()) !!}         // the application locale
{!! RichLinks::resolve(str($post->body)->sanitizeHtml(), 'uk') !!}                   // an explicit language
```

> **`resolve()` does not sanitize.** It only rewrites the markers; the HTML around them goes out as it came in. Always feed it
> sanitized HTML — `RichContentRenderer::toHtml()` or `str()->sanitizeHtml()` — and never the raw column.

**Call it after the HTML sanitizer.** A marker is a relative `href`, which any sanitizer keeps; the resolved address is not
necessarily one the sanitizer would let through. (If you resolve first and sanitize second, the sanitizer sees finished URLs,
which is a different — and for some values a stricter — question.) The same goes for anything that turns saved HTML into
something else, such as a Markdown export for `llms.txt`: resolve first, convert second.

A link whose target cannot be resolved is *unwrapped*: the `<a>` disappears and its text stays. Query strings and fragments
on a marker are dropped. Because the resolved address never meets the sanitizer, the plugin checks it itself: only a path
starting with a single `/` (not `//` or `/\`) or an absolute `http`/`https` URL is written; anything else unwraps the link, and
the fallback route answers 404 instead of redirecting.

HTML without a marker is returned byte for byte. HTML with one goes through a DOM round trip; foreign `href`/`src` values and
`&nbsp;` in text are written back verbatim, and content after a stray closing tag (`</div>`) is kept. Well-formed (sanitized)
HTML comes back unchanged apart from the rewritten links; malformed nesting is repaired the way a browser's parser would repair
it (an element closed out of order can move text out of its list or paragraph), and `&nbsp;` inside an attribute comes back as a
raw U+00A0.

## Fallback route

HTML occasionally leaks past the renderer — an e-mail template, a stale cache, a manual export — and a visitor follows a raw
marker. The plugin registers a redirect so that still works:

- `GET /rich-link/{type}/{id}` — the unprefixed language;
- `GET /{locale}/rich-link/{type}/{id}` — a prefixed language.

The language comes **strictly from the URL**, with no fallback to another one, and an unpublished target is a 404 for
everybody, including logged-in panel users. Absolute markers on your own host (`https://site.test/rich-link/…`) end up on the
same route.

A site with a catch-all route must register this one before it: set `routes.register` to `false` in the config and call
`RichLinks::routes()` yourself, before the catch-all. The route names (`rich-links.redirect`, `rich-links.redirect.locale`) are
configurable with `RichLinks::routeNames(plain:, locale:)`.

## Multilingual sites

```php
RichLinks::locales(default: 'uk', all: ['uk', 'en'], unprefixed: 'uk');
```

- `all` is the list the picker searches in and the redirect route knows about;
- `unprefixed` is the language whose URLs have no `/{locale}` prefix. `RichLinks::localizedUrl($locale, $path)` applies the
  rule and is the default URL builder of the table resolver;
- without this call the application locale is the only language.

Call it in `AppServiceProvider::boot()`, before the routes are registered (they are registered once the application has booted).

## Configuration

Each value can be set in `config/filament-rich-links.php` or in code; the code wins.

| Config key | Code | Default |
| --- | --- | --- |
| `prefix` | `RichLinks::prefix('/rich-link/')` | `/rich-link/` |
| `tool` | `RichLinks::tool('richLink')` | `richLink` |
| `form_keys.type` / `.id` | `RichLinks::formKeys(type: 'entity_type', id: 'entity_id')` | `entity_type` / `entity_id` |
| `table` | `RichLinks::table('rich_links')` | `rich_links` |
| `search_limit` | | `50` |
| `routes.register` / `.middleware` | | `true` / `['web']` |
| `routes.names.plain` / `.locale` | `RichLinks::routeNames(plain:, locale:)` | `rich-links.redirect(.locale)` |

The **prefix** must start and end with a slash. Changing it later orphans every link already saved, so pick it once. The
**tool name** is both the toolbar button and the modal action behind it; your `toolbarButtons([...])` refers to it. The
**form keys** are the keys of the dialog state — change them only to stay compatible with code that already refers to them.

### Toolbar icon

The button uses its own icon (a page with a link), not the stock chain: with the cursor inside one
of these links the stock **Link** button lights up too, and two identical chains side by side cannot
be told apart. Use your own with `RichLinks::toolIcon('heroicon-o-document-text')` or an SVG string.

## Gotchas

These cost real debugging time while building the plugin; they are all handled for you, and each is pinned by a test. They
matter if you copy the tool or write your own.

- **`->action()` needs `arguments:`.** `RichEditorTool::action()` without it gives the dialog an empty `$arguments`: the
  *editing* of an existing link silently never pre-fills, while a unit test that calls the PHP method directly stays green.
  The stock `link` button passes `arguments:` for the same reason.
- **`$getEditor()?.`, always with `?.`.** On the first render the TipTap editor is still `undefined` (it is created after an
  `await` in `init()`), so a bare `$getEditor().isActive(...)` in `activeJsExpression` throws a `TypeError` in Alpine.
- **The `link` mark drops `data-*` attributes.** Only attributes declared by its JS extension survive a re-render of the editor,
  so the marker lives in `href`, which is stored verbatim.
- **The link text has no language marker.** The picker labels translatable records with the language they were read in
  (`[en] Title`) because the editor must know what they are looking at; that label must never be inserted into content. The
  default link text is the clean title.
- **Custom toolbar buttons only come from a `RichContentPlugin`.** `RichEditor::getDefaultActions()` is a hard-coded list; there
  is no public `extraActions()`. That is why the button is a plugin.
- **Resolve after sanitizing,** not before — see [Rendering](#rendering).
- **Markers carry the morph class.** With a morph map the marker holds the alias, otherwise the class name. Markers written
  before a morph map was added keep working; both spellings resolve.
- **IDs are numeric.** A marker is `{prefix}{type}/{digits}`; models with UUID/ULID keys need a numeric key to be linkable.

## Translations

The interface ships in English, Ukrainian, German, Spanish, French, Italian, Dutch, Polish, Brazilian Portuguese and Turkish
under the `filament-rich-links::rich-links` namespace. A test keeps every language in step with the English keys. Override a
string by publishing the translations (`--tag=filament-rich-links-translations`) and editing the copy in
`lang/vendor/filament-rich-links`.

## AI agents

The package ships [Laravel Boost](https://laravel.com/docs/boost) guidelines (`resources/boost/guidelines/core.blade.php`) that
describe the registry, the resolvers, the render call and the traps above, so a coding agent wires it up correctly.

## Testing

```bash
composer install
vendor/bin/phpunit
vendor/bin/phpstan analyse --memory-limit=1G
vendor/bin/pint --test
```

The suite runs on [Orchestra Testbench](https://packages.tools/testbench) with a `workbench/` panel, a plain `Post` model and a
translatable `Article`. The registry is static: call `RichLinks::flush()` in your own tests' `setUp()` and `tearDown()` if you
configure it there.

## Screenshots

<!-- Relative paths, so the images render on the repository page at any commit. A public listing (Anystack, the Filament plugin directory) cannot read a private repository: upload the files from art/ there. -->

The internal-link dialog: pick a type, search a record.

![The internal link dialog](https://raw.githubusercontent.com/asignua/filament-rich-links-docs/main/art/link-dialog.jpg)

The same dialog in dark mode.

![The internal link dialog in dark mode](https://raw.githubusercontent.com/asignua/filament-rich-links-docs/main/art/link-dialog-dark.jpg)

The toolbar button lights up while the cursor is inside an internal link.

![The editor toolbar with the internal link button active](https://raw.githubusercontent.com/asignua/filament-rich-links-docs/main/art/editor-toolbar.jpg)

What is stored versus what the page renders: the content keeps a marker, the site shows the current address; an unpublished target becomes plain text.

![Stored markers next to the rendered addresses](https://raw.githubusercontent.com/asignua/filament-rich-links-docs/main/art/rendered.jpg)

## License

Commercial: Single Project or Unlimited, each with one year of updates. See [LICENSE.md](https://github.com/asignua/filament-rich-links-docs/blob/main/LICENSE.md).
