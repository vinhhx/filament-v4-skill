# Filament best practices: structuring resource code

Source: `docs/03-resources/13-code-quality-tips.md` and general v4 resource scaffolding conventions.

## The problem

Filament methods like `form()`, `table()`, and `infolist()` define both UI and behavior in one method. It's easy to end up with a 300-line `configure()` method that is hard to review or reuse, even with clean formatting.

## 1. Extract dedicated Schema/Table/Infolist classes

Filament v4's resource generator already scaffolds these. Each has a `configure()` method with no enforced parent class/interface (so you're free to add extra parameters for reuse):

```php
// app/Filament/Resources/Customers/Schemas/CustomerForm.php
namespace App\Filament\Resources\Customers\Schemas;

use Filament\Forms\Components\TextInput;
use Filament\Schemas\Schema;

class CustomerForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name'),
            // ...
        ]);
    }
}
```

Wire it into the resource:

```php
use App\Filament\Resources\Customers\Schemas\CustomerForm;
use Filament\Schemas\Schema;

public static function form(Schema $schema): Schema
{
    return CustomerForm::configure($schema);
}
```

Do the same for `CustomersTable::configure($table)` (used from `table()`) and `CustomerInfolist::configure($schema)` (used from `infolist()`).

## 2. Extract component classes when a single field/column/action needs heavy configuration

```php
// app/Filament/Resources/Customers/Schemas/Components/CustomerNameInput.php
namespace App\Filament\Resources\Customers\Schemas\Components;

use Filament\Forms\Components\TextInput;

class CustomerNameInput
{
    public static function make(): TextInput
    {
        return TextInput::make('name')
            ->label('Full name')
            ->required()
            ->maxLength(255)
            ->placeholder('Enter your full name')
            ->belowContent('This is the name that will be displayed on your profile.');
    }
}
```

```php
use App\Filament\Resources\Customers\Schemas\Components\CustomerNameInput;

public static function configure(Schema $schema): Schema
{
    return $schema->components([
        CustomerNameInput::make(),
        // ...
    ]);
}
```

Suggested (not enforced) locations/naming inside a resource directory:

| Kind | Directory | Naming |
|---|---|---|
| Schema field | `Schemas/Components/` | `CustomerNameInput`, `CustomerCountrySelect` |
| Table column | `Tables/Columns/` | `CustomerNameColumn`, `CustomerCountryColumn` |
| Table filter | `Tables/Filters/` | `CustomerCountryFilter`, `CustomerStatusFilter` |
| Action | `Actions/` | `EmailCustomerAction`, `UpdateCustomerCountryBulkAction` |

## 3. Extract reusable actions the same way

```php
// app/Filament/Resources/Customers/Actions/EmailCustomerAction.php
namespace App\Filament\Resources\Customers\Actions;

use App\Models\Customer;
use Filament\Actions\Action;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;
use Filament\Support\Icons\Heroicon;

class EmailCustomerAction
{
    public static function make(): Action
    {
        return Action::make('email')
            ->label('Send email')
            ->icon(Heroicon::Envelope)
            ->schema([
                TextInput::make('subject')->required()->maxLength(255),
                Textarea::make('body')->autosize()->required(),
            ])
            ->action(function (Customer $customer, array $data) {
                // ...
            });
    }
}
```

Reuse it in `getHeaderActions()` of a page, or as a `recordActions()` entry in a table — same class, different context.

## Package/architecture map (for orientation, not something you usually need to change)

```
support (base utilities)
  → schemas (layout/UI primitives)
    → forms, infolists, tables, actions, notifications, widgets
      → panels (full admin framework: Resources, Pages, Panel config)
```

- Resources (CRUD for an Eloquent model) live under `app/Filament/Resources/`.
- Pages (Livewire components) live under a resource's `Pages/` directory, or standalone for custom panel pages.
- Relation managers are Livewire components nested inside a resource's edit/view page.
- Widgets (stats, charts) can be dashboard-level or resource-page-level.

## General hygiene

- Prefer the dot-notation relationship syntax (`TextColumn::make('author.name')`) over manually joining/loading relationships — see `references/performance.md`.
- Keep `getEloquentQuery()` overrides on the Resource class, not scattered across pages, unless a specific page genuinely needs different scoping.
- When a feature has its own security caveats (file uploads, rich editor attachments, inline-editable columns), read that component's docs section before shipping — don't assume it's automatically safe. See `references/security.md` for the roll-up.
