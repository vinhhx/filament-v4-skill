# Filament testing

Source: `docs/10-testing/*`. All examples use Pest + the Livewire Pest plugin (`livewire()` helper); swap in `Livewire::test()` for plain PHPUnit.

## What is (and isn't) a Livewire component

Test these by passing the class to `livewire()` / `Livewire::test()`:
- Panel pages, including a resource's `Pages/` classes (List/Create/Edit/View).
- Relation managers.
- Widgets.

These are **not** Livewire components — test them as plain PHP classes, or indirectly through a Livewire component that uses them:
- Resource classes.
- Schema components (form fields, infolist entries).
- Actions.

## Setup: authenticate the test user

```php
use App\Models\User;

beforeEach(function () {
    $user = User::factory()->create();
    actingAs($user);
});
```

## Resource pages

### List page

```php
use App\Filament\Resources\Users\Pages\ListUsers;
use App\Models\User;

it('can load the page', function () {
    $users = User::factory()->count(5)->create();

    livewire(ListUsers::class)
        ->assertOk()
        ->assertCanSeeTableRecords($users);
});
```

Search / sort / filter on a list page (delegates to the table testing API below):

```php
livewire(ListUsers::class)
    ->assertCanSeeTableRecords($users)
    ->searchTable($users->first()->name)
    ->assertCanSeeTableRecords($users->take(1))
    ->assertCanNotSeeTableRecords($users->skip(1));

livewire(ListUsers::class)
    ->sortTable('name')
    ->assertCanSeeTableRecords($users->sortBy('name'), inOrder: true);

livewire(ListUsers::class)
    ->filterTable('locale', $users->first()->locale)
    ->assertCanSeeTableRecords($users->where('locale', $users->first()->locale))
    ->assertCanNotSeeTableRecords($users->where('locale', '!=', $users->first()->locale));
```

Bulk actions use `TestAction`:

```php
use Filament\Actions\Testing\TestAction;
use function Pest\Laravel\assertDatabaseMissing;

livewire(ListUsers::class)
    ->selectTableRecords($users)
    ->callAction(TestAction::make(DeleteBulkAction::class)->table()->bulk())
    ->assertNotified()
    ->assertCanNotSeeTableRecords($users);

$users->each(fn (User $user) => assertDatabaseMissing($user));
```

### Create page

```php
it('can create a user', function () {
    $newUserData = User::factory()->make();

    livewire(CreateUser::class)
        ->fillForm(['name' => $newUserData->name, 'email' => $newUserData->email])
        ->call('create')
        ->assertNotified()
        ->assertRedirect();

    assertDatabaseHas(User::class, ['name' => $newUserData->name, 'email' => $newUserData->email]);
});
```

Validation — use a Pest dataset to avoid repeating boilerplate:

```php
it('validates the form data', function (array $data, array $errors) {
    livewire(CreateUser::class)
        ->fillForm([...User::factory()->make()->toArray(), ...$data])
        ->call('create')
        ->assertHasFormErrors($errors)
        ->assertNotNotified()
        ->assertNoRedirect();
})->with([
    '`name` is required' => [['name' => null], ['name' => 'required']],
    '`email` is a valid email address' => [['email' => Str::random()], ['email' => 'email']],
]);
```

### Edit page

```php
livewire(EditUser::class, ['record' => $user->id])
    ->assertOk()
    ->assertSchemaStateSet(['name' => $user->name, 'email' => $user->email]);

livewire(EditUser::class, ['record' => $user->id])
    ->fillForm(['name' => $newName])
    ->call('save')
    ->assertNotified();

livewire(EditUser::class, ['record' => $user->id])
    ->callAction(DeleteAction::class)
    ->assertNotified()
    ->assertRedirect();
```

### View page

```php
livewire(ViewUser::class, ['record' => $user->id])
    ->assertOk()
    ->assertSchemaStateSet(['name' => $user->name, 'email' => $user->email]);
```

### Relation managers

Pass both `ownerRecord` and `pageClass`:

```php
livewire(EditUser::class, ['record' => $user->id])
    ->assertSeeLivewire(PostsRelationManager::class);

livewire(PostsRelationManager::class, [
    'ownerRecord' => $user,
    'pageClass' => EditUser::class,
])
    ->assertOk()
    ->assertCanSeeTableRecords($user->posts);
```

### `getFormActions()` custom actions

Target the `form-actions` key in the `content` schema:

```php
use Filament\Actions\Testing\TestAction;

livewire(CreateUser::class)
    ->fillForm(['name' => 'Test User', 'email' => 'test@example.com'])
    ->callAction(TestAction::make('createAndVerifyEmail')->schemaComponent('form-actions', schema: 'content'));
```

### Multiple / multi-tenant panels

```php
use Filament\Facades\Filament;

Filament::setCurrentPanel('app'); // non-default panel under test

// Multi-tenant: also boot the panel after setting the tenant
Filament::setTenant($team);
Filament::setCurrentPanel('admin');
Filament::bootCurrentPanel();
```

## Tables

```php
livewire(ListPosts::class)->assertSuccessful();

livewire(ListPosts::class)
    ->assertCanSeeTableRecords($posts)
    ->assertCanNotSeeTableRecords($trashedPosts)
    ->assertCountTableRecords(4);
```

Notes:
- `assertCanSeeTableRecords()` only checks the *current* page of a paginated table — call `->call('gotoPage', 2)` to switch pages.
- If the table uses `deferLoading()`, call `->loadTable()` before asserting records are visible.
- Sort assertions must compare against records ordered with `Post::query()->orderBy(...)->get()`, not `$collection->sortBy()` — DB and PHP sort collations can differ.

Column assertions:

```php
->assertCanRenderTableColumn('title')
->assertCanNotRenderTableColumn('comments')
->searchTable($title) / ->searchTableColumns(['title' => $title])
->assertTableColumnStateSet('author.name', $post->author->name, record: $post)
->assertTableColumnFormattedStateSet('author.name', 'Smith, John', record: $post)
->assertTableColumnExists('author', fn (TextColumn $column) => $column->getDescriptionBelow() === $post->subtitle, $post)
->assertTableColumnVisible('created_at') / ->assertTableColumnHidden('author')
->assertTableColumnHasDescription('author', 'Author! ↓↓↓', $post, 'above')
->assertTableColumnHasExtraAttributes('author', ['class' => 'text-danger-500'], $post)
->assertTableSelectColumnHasOptions('status', ['published' => 'Published'], $post)
```

Filters:

```php
->filterTable('is_published')
->filterTable('author_id', $authorId)
->resetTableFilters()
->removeTableFilter('is_published')
->removeTableFilters()
->assertTableFilterVisible('created_at') / ->assertTableFilterHidden('author')
->assertTableFilterExists('author', fn (SelectFilter $filter) => $filter->getLabel() === 'Select author')
```

Summaries:

```php
->assertTableColumnSummarySet('rating', 'average', $posts->avg('rating'))
->assertTableColumnSummarySet('rating', 'average', $posts->take(10)->avg('rating'), isCurrentPaginationPageOnly: true)
->assertTableColumnSummarySet('rating', 'range', [$posts->min('rating'), $posts->max('rating')])
```

Column visibility toggling:

```php
->toggleAllTableColumns()      // show all
->toggleAllTableColumns(false) // hide all
```

## Actions

```php
livewire(EditInvoice::class, ['invoice' => $invoice])->callAction('send');
```

Table actions (record-scoped) use `TestAction::make($name)->table($record)`:

```php
use Filament\Actions\Testing\TestAction;

livewire(ListInvoices::class)->callAction(TestAction::make('send')->table($invoice));
livewire(ListInvoices::class)->assertActionVisible(TestAction::make('send')->table($invoice));
```

Header actions omit the record: `TestAction::make('create')->table()`.

Bulk actions: select first, then `->bulk()`:

```php
livewire(ListInvoices::class)
    ->selectTableRecords($invoices->pluck('id')->toArray())
    ->callAction(TestAction::make('send')->table()->bulk());
```

Actions inside a schema component (e.g. `belowContent()` on a field) use `schemaComponent()`:

```php
livewire(EditInvoice::class)->callAction(TestAction::make('send')->schemaComponent('customer_id'));
```

Chained actions (an action inside another action's modal) pass an array of `TestAction`s in order.

## Schemas (forms/infolists) and notifications

For deeper coverage, read `docs/10-testing/04-testing-schemas.md` and `docs/10-testing/06-testing-notifications.md` directly — they cover field-level assertions (`assertFormFieldExists`, `assertFormSet`, etc.) and asserting dispatched `Notification::make()` calls beyond the basic `assertNotified()` shown above.

## Custom pages with no special Filament behavior

These are plain Livewire components — test them per the [Livewire testing docs](https://livewire.laravel.com/docs/testing), no Filament-specific helpers needed.

## Minimum bar for "is this tested"

For every resource: a passing-load test for each page (List/Create/Edit/View), a validation test for Create/Edit, a happy-path create/update/delete test, and an authorization test proving an unauthorized user is blocked. For every custom action: a test that it performs its effect when authorized, and a test that it's hidden/blocked when not.
