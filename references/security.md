# Filament security

Source: `docs/09-advanced/06-security.md` (read the full file for edge cases), plus `packages/forms/docs/09-file-upload.md`.

Filament trusts values passed into flexible configuration methods (`url()`, `icon()`, `html()`, `extraAttributes()`, etc.) — validating/sanitizing user-supplied data before it reaches Filament is the application developer's responsibility.

## 1. Authorization

### What Filament checks automatically

Filament checks Laravel model policies for standard resource CRUD: `viewAny()`, `create()`, `update()`, `view()`, `delete()`. This only covers built-in resource operations.

**You must authorize yourself:**
- Custom actions — use `visible()`, `hidden()`, `authorize()`. Not applied automatically.
- Custom pages, custom Livewire components, API endpoints, other business logic.
- Inline editable table columns (`ToggleColumn`, `TextInputColumn`, `SelectColumn`, `CheckboxColumn`) — these do **not** check model policies before saving; they only respect `disabled()`. Add your own authorization logic via `disabled()` if needed.

### Authorization re-runs on every Livewire request

Filament re-checks authorization on every Livewire update (search, filter, pagination, action call, form interaction) — not just on initial mount — across resource pages, custom panel pages, relation managers, widgets, and tenancy pages. If a user's permissions change mid-session, the *next* interaction is checked against current policy state.

**Practical implication:** Livewire's synth step (deserializing public properties), `boot()`/`boot{Trait}()` hooks, and the `mount()` body all run *before* Filament's authorization hooks fire on the very first request. Do not put side effects (DB writes, audit logging on SELECT, dispatched events, external calls) in those early hooks if an unauthorized user should never trigger them — put that work after an explicit `$this->authorizeAccess()` call, or inside an action method invoked via `wire:click` (which always runs post-authorization).

### Test authorization end-to-end

Don't rely solely on Filament's built-in policy checks — write Pest/Livewire tests asserting different user roles get the expected access for every custom action, page, and route, not just resource CRUD. See `references/testing.md`.

## 2. Validating user input passed into configuration methods

Methods like `url()`, `icon()`, `html()` render whatever you give them. If the value comes from user input or untrusted DB content, sanitize it first.

### URLs — use `Str::sanitizeUrl()`

```php
use Filament\Tables\Columns\TextColumn;
use Illuminate\Support\Str;

TextColumn::make('website')
    ->url(fn (string $state): ?string => Str::sanitizeUrl($state))
```

- Returns the URL unchanged if it's schemeless/relative or `http`/`https`; returns `null` otherwise.
- Defeats common obfuscation tricks (`&#x09;`, `%0A`, mixed-case schemes, embedded control characters) used to sneak `javascript:` past naive checks.
- To allow more schemes (e.g. `mailto:`, `tel:`), pass them explicitly — the default allowlist is replaced, not extended:

```php
Str::sanitizeUrl($state, allowedSchemes: ['http', 'https', 'mailto', 'tel'])
```

- It is **only** a scheme allowlist. It does NOT protect against open redirects, SSRF, or guarantee the URL is safe outside an `href`-style HTML attribute context. Layer your own host allowlist on top if you need that:

```php
->url(function (string $state): ?string {
    $sanitized = Str::sanitizeUrl($state);
    if (blank($sanitized)) return null;

    $host = parse_url($sanitized, PHP_URL_HOST);
    return in_array($host, ['example.com', 'cdn.example.com'], true) ? $sanitized : null;
})
```

### Icons

`icon()` accepts a Blade icon name or an image URL (any string containing `/`). URL strings are escaped before being rendered as `src`; icon-name strings resolve via Blade's icon system and an invalid name causes a rendering error — validate against a known allowlist if the icon name is user-controlled.

### `extra*Attributes()` methods (`extraAttributes()`, `extraInputAttributes()`, `extraCellAttributes()`, etc.)

These render into HTML **unescaped by design** (so Alpine/Livewire directives work). Never pass user-controlled data as attribute names or values here — an attacker can break out of the attribute and inject markup.

**Rule of thumb:** treat any user-controlled value passed to a Filament configuration method with the same caution as rendering it raw in a Blade template.

## 3. HTML sanitization (`html()` / `markdown()` on TextColumn/TextEntry)

Filament auto-sanitizes with Symfony's `HtmlSanitizer`, stripping dangerous elements like `<script>`. Default config allows safe elements, relative links/media, and a specific allowlist of attributes (`class`, `data-color`, `data-cols`, `data-col-span`, `data-from-breakpoint`, `data-id`, `data-type`, `style`, `width`/`height` on `img`), input capped at 500,000 chars.

- The `style` attribute is allowed (needed for rich text formatting) — this means `background: url(...)` and `position: fixed` are **not** stripped. If you render HTML from fully untrusted users, tighten this.
- Customize via `HtmlSanitizerConfig` service-container binding + `extend()` in a service provider: `allowAttribute()` to add, `dropAttribute()` to remove. Don't remove attributes the rich editor depends on (`data-color`, `data-cols`, `data-id`, `style`) unless you understand the breakage.
- **In your own Blade views**, sanitize manually — never `{!! $content !!}` with unsanitized content:

```blade
{!! str($record->content)->sanitizeHtml() !!}
{!! str($record->content)->markdown()->sanitizeHtml() !!}
```

## 4. Panel access

- Outside `local` env, you **must** implement `FilamentUser` on your User model and define `canAccessPanel()`, or nobody can log in.
- Multi-panel apps: check the `$panel` argument inside `canAccessPanel()` and return the right result per panel.
- MFA (TOTP/email codes) is opt-in and only enforced on Filament's own auth flow — implement it separately for any other auth path (API routes, non-Filament login pages).

## 5. Model attribute exposure

Filament exposes all non-`$hidden` model attributes to JS via Livewire binding (needed for dynamic forms; only fields with a matching form field are actually editable — not a mass-assignment hole). Sensitive attributes (API keys, internal flags) that shouldn't be visible in the browser should be added to the model's `$hidden`, or stripped via `mutateFormDataBeforeFill()` on the Edit/View page.

## 6. File uploads

- `FileUpload::preserveFilenames()` re-exposes the client's original filename via `getClientOriginalName()` — don't enable this for untrusted users; by default Livewire generates a random name + infers extension from MIME type, which is safer. Preserving names also risks filename collisions unless scoped to a directory.
- Review the "Authorizing existing file paths" section of the file-upload docs whenever a field lets a client reference an existing stored file path.
- Every Livewire component using `InteractsWithSchemas` exposes Livewire's `_startUpload`/`_finishUpload` RPCs for **any** property name by default — not just declared `FileUpload` fields. On any component reachable by users who shouldn't be able to upload arbitrary files (unauthenticated pages, or pages with no upload field at all), add:

```php
use Filament\Schemas\Concerns\InteractsWithSchemas;
use Filament\Schemas\Concerns\RestrictsFileUploadsToSchemaComponents;
use Filament\Schemas\Contracts\HasSchemas;
use Livewire\Component;

class ViewProduct extends Component implements HasSchemas
{
    use InteractsWithSchemas;
    use RestrictsFileUploadsToSchemaComponents;
}
```

This rejects uploads that don't target a real, currently-visible `FileUpload`/attachment-capable field (hidden fields don't count as valid targets).

## 7. Scoping queries (multi-tenant / permission-scoped data)

`getEloquentQuery()` returns all records by default. For any custom query, action, or page you build outside Filament's built-in tenancy feature, you must scope it yourself (`modifyQueryUsing()` on a table, or override `getEloquentQuery()` on the resource) — otherwise users can see data belonging to other tenants/users. Built-in tenancy scopes resource queries automatically; hand-rolled queries do not get this for free.

## Security checklist before shipping a feature

- [ ] Every custom action has explicit `visible()`/`hidden()`/`authorize()`.
- [ ] Every custom page/Livewire component authorizes access itself.
- [ ] Any `url()`/`icon()` fed by user or DB data goes through `Str::sanitizeUrl()` (or a stricter host-allowlist wrapper).
- [ ] No `extra*Attributes()` call receives raw user-controlled keys/values.
- [ ] Rich text output uses `html()`/`markdown()` (auto-sanitized) or manual `->sanitizeHtml()` in Blade — never raw `{!! !!}`.
- [ ] `canAccessPanel()` implemented and panel-aware for production.
- [ ] Sensitive model attributes are `$hidden` or stripped via `mutateFormDataBeforeFill()`.
- [ ] `preserveFilenames()` only enabled for trusted users, with directory scoping.
- [ ] `RestrictsFileUploadsToSchemaComponents` added to any reachable component without an upload field.
- [ ] Every custom/manual query is scoped to the current user/tenant.
- [ ] Authorization behavior is covered by tests, not just assumed.
