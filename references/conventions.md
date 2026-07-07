# Filament coding conventions

These conventions come from Filament's own contribution guide (`CLAUDE.md`). They matter most when you are **extending or overriding core Filament classes** (custom components, custom actions, plugins published for others to use). For ordinary "use the fluent API in a resource" application code, following the same patterns still produces more idiomatic, readable code — but the strict naming/no-`final` rules are primarily about code that other developers will extend.

## Variable names

Never abbreviate. Use full descriptive names:

```php
// GOOD
$exception, $component, $response, $configuration, $record, $livewire

// BAD
$e, $comp, $res, $cfg, $rec, $lw
```

Only exception: universally understood short names like `$id`, `$url`.

## The fluent API pattern

Components are built with a static `make()` constructor and chainable methods. Nullable properties get nullable setters so a caller can "unset" a previously configured value:

```php
TextInput::make('name')
    ->label('Full name')
    ->icon('heroicon-o-user')

// Property and setter share a name; nullable to allow unsetting later
protected string | Closure | null $icon = null;

public function icon(string | Closure | null $icon): static
{
    $this->icon = $icon;

    return $this;
}

// Getter is prefixed `get`, and resolves Closures via evaluate()
public function getIcon(): ?string
{
    return $this->evaluate($this->icon);
}
```

## Boolean methods

```php
// Property: `is`/`should`/`can`/`has` prefix, defaults false, supports Closure
protected bool | Closure $isDisabled = false;

// Setter: verb form, defaults true, pass false to undo
public function disabled(bool | Closure $condition = true): static
{
    $this->isDisabled = $condition;

    return $this;
}

// Getter: cast to bool, prefixed `is`/`should`/`can`/`has`
public function isDisabled(): bool
{
    return (bool) $this->evaluate($this->isDisabled);
}
```

This is why you see `->required()` / `->required(false)`, `->disabled()` / `->disabled(false)`, etc. everywhere — assume any boolean-flag method follows this pattern and can be toggled off with `false` or a closure.

## Closures

- Filament methods overwhelmingly accept `Closure` in place of a static value. Reach for `fn (...) => ...` before writing an if/else around a call site.
- Use `static fn` when the closure body doesn't reference `$this`:

```php
->placeholder(static fn (Select $component): ?string => $component->isDisabled() ? null : 'Select...')
->visible(fn (): bool => $this->canView()) // uses $this, can't be static
```

- Closures can request many different injectable parameters by name (`$record`, `$state`, `$livewire`, `$component`, `$table`/`$schema`, `$rowLoop`, plus anything resolvable from Laravel's container like `Request`). See the "Column/Field utility injection" sections of the component docs for the full list per component type.

## Container resolution

Prefer `app()` over `new` so applications can bind their own implementation:

```php
app(RelationshipJoiner::class)->prepareQuery($relationship); // good
(new RelationshipJoiner())->prepareQuery($relationship);     // avoid
```

## Extensibility

Do not mark classes `final` or properties/classes `readonly` in code meant to be extended (custom base components, published plugins) — application developers need to be able to subclass Filament classes.

## Concerns and contracts

- Traits living in a `Concerns/` directory: `Can*` traits express capabilities, `Has*` traits express properties/state.
- Interfaces live in `Contracts/` directories.

## Deprecations

Keep old public methods that are referenced in docs; mark them deprecated rather than deleting:

```php
/** @deprecated Use `newMethod()` instead. */
public function oldMethod(): void
{
    return $this->newMethod();
}
```

## PHPDoc

Only add PHPDoc when it conveys type information beyond native PHP types:

```php
/** @var array<string, array{label: string, icon: string}> */  // Good — shape info
/** @param string $name The name */                            // Redundant — skip
```

## Code comments referencing code

Wrap method/class names in backticks in comments, same as in prose docs:

```php
// Uses `evaluate()` to resolve the `Closure`
// Returns `null` if the `$record` is not set
```

## CSS in Blade views (only relevant if editing/publishing Blade views)

Never write Tailwind utility classes directly in a Blade view. Put them in a CSS file behind a hook class using `@apply`:

```css
.fi-fo-field {
    @apply grid gap-y-2;
}
```

Hook class naming: `fi-` prefix + package code (`fi-fo-` forms, `fi-ta-` tables, `fi-ac-` actions, etc.), using abbreviations like `btn`, `col`, `ctn`, `wrp`.
