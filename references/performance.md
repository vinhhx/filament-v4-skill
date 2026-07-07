# Filament performance

Sources: `packages/tables/docs/01-overview.md`, `packages/tables/docs/02-columns/01-overview.md`, `docs/03-resources/01-overview.md`, `docs/13-deployment.md`, `docs/01-introduction/04-optimizing-local-development.md`.

## Avoiding N+1 queries in tables

- **Relationship columns use dot notation and are auto-eager-loaded.** `TextColumn::make('author.name')` makes Filament eager-load the `author` relationship for you — don't manually add `->with('author')` duplicated logic, and don't fetch relationship data through a closure/accessor that triggers a query per row.
- **Use the built-in relationship aggregate helpers instead of accessors:**

```php
TextColumn::make('users_count')->counts('users');                 // COUNT
TextColumn::make('users_exists')->exists('users');                 // EXISTS
TextColumn::make('users_avg_age')->avg('users', 'age');             // AVG
TextColumn::make('users_sum_age')->sum('users', 'age');             // SUM
// min() / max() also available
```

  The column name must follow Laravel's convention (`{relationship}_count`, `{relationship}_avg_{column}`, etc.) because these compile to `withCount()`/`withAvg()` under the hood — a single query, not one per row. Scope the relationship before aggregating by passing an array: `counts(['users' => fn (Builder $query) => $query->where('is_active', true)])`.
- If you must compute something per-row that can't be expressed as a query aggregate, eager-load the relationship explicitly in `getEloquentQuery()`/`modifyQueryUsing()` rather than lazy-loading inside a column's `state()`/formatting closure.

## Deferring table loading

For tables backed by large or slow datasets, load the table asynchronously after the rest of the page has rendered:

```php
public function table(Table $table): Table
{
    return $table->deferLoading();
}
```

When testing a deferred table, call `->loadTable()` before `assertCanSeeTableRecords()`.

## Search and sort cost

- `searchable()` and `sortable()` on a computed/aliased column (not a real DB column) require you to pass real column names or a `query` closure — see `packages/tables/docs/02-columns/01-overview.md` — otherwise every row triggers PHP-side evaluation instead of a SQL `WHERE`/`ORDER BY`.
- By default, table search splits the query into words and searches each independently, which is more flexible but costs more on large tables. Disable it if it's a bottleneck:

```php
$table->splitSearchTerms(false);
```

- Filament auto-adds a primary-key tiebreaker sort for consistent pagination. If your table has no real primary key or you've profiled this as unnecessary overhead, disable with `defaultKeySort(false)`.

## Badges and other deferred UI

Tab/badge counts computed with `deferBadge()` must be passed as a closure, not a pre-computed value — passing a raw value (`badge(Customer::query()->count())`) runs the query immediately when the tab is built, defeating the purpose of deferral:

```php
Tab::make('Active')
    ->badge(fn () => Customer::query()->where('is_active', true)->count())
    ->deferBadge()
```

## Scoping and global scopes

- `getEloquentQuery()` is the single choke point every resource query passes through — put shared constraints there once instead of repeating `->where()` across pages/columns/filters.
- Global scopes are respected by default. Use `withoutGlobalScopes()` / `withoutGlobalScopes([Scope::class])` only when intentionally bypassing them (e.g. viewing soft-deleted records) — don't disable scopes broadly just to "fix" an unrelated query issue.

## Production deployment checklist

Run in your deploy script:

```bash
php artisan filament:optimize        # = filament:cache-components + icons:cache
php artisan optimize                 # Laravel config/route cache
```

- `filament:cache-components` indexes resources/pages/widgets/relation managers so Filament doesn't scan the filesystem to auto-discover components on every request. Skip this while actively developing locally — new components won't be picked up until the cache is cleared (`php artisan filament:clear-cached-components` or `filament:optimize-clear`).
- Cache Blade icons (`php artisan icons:cache`) — re-run whenever you install a new icon package.
- Enable OPcache on the server (`opcache.enable=1`); this is a PHP-level win independent of Filament.
- Keep the `filament:upgrade` `post-autoload-dump` composer script intact so panel assets stay in sync after `composer install`/`update`.

## Local development slowness (not relevant to production)

If the panel feels slow only in local dev, check for these unrelated culprits before assuming a Filament code problem:
- OPcache disabled locally.
- Antivirus/Defender real-time scanning the project directory (common on Windows).
- Debug tooling left on: Laravel Herd view-dump debugging, Laravel Debugbar, Xdebug (`xdebug.mode=off` when not actively debugging).
- Blade icons not cached (`php artisan icons:cache`).

## Widgets and polling

Dashboard/resource widgets can poll on an interval — don't set aggressive polling on widgets that run expensive queries; prefer a sensible interval (seconds, not sub-second) or trigger refresh from user action / Livewire events instead of polling when the underlying data changes infrequently.
