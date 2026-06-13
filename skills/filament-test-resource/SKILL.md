---
name: filament-test-resource
description: >-
  Generates complete Pest suite for Filament v5 Resources — Pages (Create/Edit/List/View),
  Form Schemas, Tables, RelationManagers, Actions. Uses Livewire::test() (required pattern).
  Activate when: create tests for resource, test suite, generate Filament tests,
  test form/table/page, resource coverage, test relation manager, validate Filament form.
---

# Filament Test Suite Generator

Pest suites for Filament v5 Resources.

## When to apply

- Create tests for a new Filament Resource
- Requests: "create tests for [resource]", "test suite", "generate tests"
- Full Resource coverage

## Structure

```
tests/Feature/[ResourceName]/
├── [ResourceName]ResourceTest.php
├── Pages/{Create,Edit,List}[ResourceName]Test.php
├── Schemas/[ResourceName]FormTest.php
├── Tables/[ResourceName]TableTest.php
└── RelationManagers/[RelationName]RelationManagerTest.php
```

## Critical Patterns

### 1. `Livewire::test()` — NEVER `livewire()`

```php
use Livewire\Livewire;
// ✅ Livewire::test(ListResources::class)->assertSuccessful();
// ❌ use function Pest\Livewire\livewire; livewire(ListResources::class)
```

### 2. beforeEach Setup

```php
use Illuminate\Foundation\Testing\RefreshDatabase;
uses(RefreshDatabase::class);

beforeEach(function (): void {
    $this->user = User::factory()->create();
    $this->actingAs($this->user);
    // Multi-tenant projects: also set up tenant
    // $this->tenant = Tenant::factory()->create();
    // Filament::setTenant($this->tenant);
});
```

### 3. Enums

```php
// ✅ instanceof + value
$weight = $column->getWeight();
expect($weight)->toBeInstanceOf(Filament\Support\Enums\FontWeight::class)
    ->and($weight->value)->toBe('bold');
// ❌ expect($column->getWeight())->toBe('bold');
```

### 4. Hidden Columns — test config, not rendering

```php
// ✅
test('chassis column is hidden by default', function (): void {
    $component = Livewire::test(ListResources::class);
    $column = $component->instance()->getTable()->getColumn('chassis');
    expect($column)->not->toBeNull()
        ->and($column->isToggleable())->toBeTrue()
        ->and($column->isToggledHiddenByDefault())->toBeTrue();
});
// ❌ ->assertCanRenderTableColumn('chassis');  // hidden doesn't render
```

### 5. Methods with Parameters — pass instance

```php
// ✅ expect($column->isCopyable($record))->toBeTrue();
// ❌ expect($column->isCopyable())->toBeTrue();
```

### 6. Unique Constraints — create separate related records

```php
test('table displays different records', function (): void {
    $firstType = Category::factory()->create();
    $secondType = Category::factory()->create();

    $r1 = Model::factory()->create(['category_id' => $firstType->id]);
    $r2 = Model::factory()->create(['category_id' => $secondType->id]);

    Livewire::test(ListResources::class)->assertCanSeeTableRecords([$r1, $r2]);
});
```

### 7. Functional Test > Direct Method Call

```php
// ✅ Via real page usage
test('form has all required fields', function (): void {
    Livewire::test(CreateResource::class)
        ->assertFormFieldExists('name')
        ->assertFormFieldExists('email');
});
// ❌ Direct schema calls may fail
```

### 8. Group Sequential Assertions of Same Method (CRITICAL)

```php
// ✅ Same method grouped
test('has all required fields', function (): void {
    Livewire::test(CreateResource::class)
        ->assertFormFieldExists('name')
        ->assertFormFieldExists('email')
        ->assertFormFieldExists('phone');
});

test('displays all columns', function (): void {
    Livewire::test(ListResources::class)
        ->assertCanRenderTableColumn('name')
        ->assertCanRenderTableColumn('email')
        ->assertCanRenderTableColumn('created_at');
});

// ❌ Multiple tests doing the same thing
test('has name field', function (): void { ... });
test('has email field', function (): void { ... });
```

**Group ONLY if:** same assertion method, no intermediate logic/data, same behavior.

**Do NOT group if:** different behaviors (validation, filter, search), different data per assertion, distinct scenarios/edge cases.

## Templates

### ResourceTest

```php
<?php
declare(strict_types=1);

use App\Filament\Resources\[ResourceName];
use App\Models\[Model];
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);
beforeEach(function (): void { /* auth setup */ });

test('resource exists', fn () => expect([ResourceName]::class)->toBeString());
test('resource has correct model', fn () => expect([ResourceName]::getModel())->toBe([Model]::class));

test('can create [resource]', function (): void {
    $data = [Model]::factory()->make()->toArray();
    [ResourceName]::create($data);
    $this->assertDatabaseHas('[table]', $data);
});
```

### Create Page

```php
test('page can be rendered', fn () =>
    Livewire::test(Create[Resource]::class)->assertSuccessful()
);

test('can create [resource]', function (): void {
    Livewire::test(Create[Resource]::class)
        ->fillForm([/* test data */])
        ->call('create')
        ->assertNotified();
    $this->assertDatabaseHas('[table]', [/* expected data */]);
});

test('validates required fields', function (): void {
    Livewire::test(Create[Resource]::class)
        ->fillForm(['field1' => null, 'field2' => null])
        ->call('create')
        ->assertHasFormErrors(['field1', 'field2']);
});
```

### Edit Page

```php
test('page can be rendered', function (): void {
    $record = [Model]::factory()->create();
    Livewire::test(Edit[Resource]::class, ['record' => $record->id])->assertSuccessful();
});

test('form is populated with data', function (): void {
    $record = [Model]::factory()->create();
    Livewire::test(Edit[Resource]::class, ['record' => $record->id])
        ->assertFormSet(['name' => $record->name]);
});

test('can update [resource]', function (): void {
    $record = [Model]::factory()->create();
    Livewire::test(Edit[Resource]::class, ['record' => $record->id])
        ->fillForm(['name' => 'Updated Name'])
        ->call('save')
        ->assertNotified();
    expect($record->fresh()->name)->toBe('Updated Name');
});
```

### List Page

```php
test('page can be rendered', fn () =>
    Livewire::test(List[Resources]::class)->assertSuccessful()
);

test('can list [resources]', function (): void {
    $records = [Model]::factory()->count(3)->create();
    Livewire::test(List[Resources]::class)->assertCanSeeTableRecords($records);
});

test('can search by field', function (): void {
    $record = [Model]::factory()->create(['name' => 'Searchable']);
    $other = [Model]::factory()->create(['name' => 'Other']);
    Livewire::test(List[Resources]::class)
        ->searchTable('Searchable')
        ->assertCanSeeTableRecords([$record])
        ->assertCanNotSeeTableRecords([$other]);
});
```

### Table Test

```php
describe('table columns', function () {
    test('displays all columns', function (): void {
        Livewire::test(List[Resources]::class)
            ->assertCanRenderTableColumn('name')
            ->assertCanRenderTableColumn('email')
            ->assertCanRenderTableColumn('created_at');
    });
});

describe('table sorting', function () {
    test('can sort all columns', function (): void {
        Livewire::test(List[Resources]::class)
            ->sortTable('name')->sortTable('email')->sortTable('created_at');
    });
});
```

### Form Test

```php
describe('form fields', function () {
    test('has all required fields', function (): void {
        Livewire::test(Create[Resource]::class)
            ->assertFormFieldExists('name')
            ->assertFormFieldExists('email');
    });
});

describe('form validation', function () {
    test('validates required fields', function (): void {
        Livewire::test(Create[Resource]::class)
            ->fillForm(['name' => null, 'email' => null])
            ->call('create')
            ->assertHasFormErrors(['name' => 'required', 'email' => 'required']);
    });
});
```

### Relation Manager

```php
test('can render relation manager', function (): void {
    $owner = [Model]::factory()->create();
    Livewire::test([RelationManager]::class, [
        'ownerRecord' => $owner,
        'pageClass' => Edit[Resource]::class,
    ])->assertSuccessful();
});

test('only shows records for owner', function (): void {
    $owner = [Model]::factory()->create();
    $related = [RelatedModel]::factory()->create(['owner_id' => $owner->id]);
    $otherOwner = [Model]::factory()->create();
    $otherRelated = [RelatedModel]::factory()->create(['owner_id' => $otherOwner->id]);

    Livewire::test([RelationManager]::class, [
        'ownerRecord' => $owner,
        'pageClass' => Edit[Resource]::class,
    ])
        ->assertCanSeeTableRecords([$related])
        ->assertCanNotSeeTableRecords([$otherRelated]);
});
```

## Workflow

1. **Analyze Resource**: model, relations, form fields, table columns, unique constraints
2. **Create test dir**: mirror resource structure
3. **Generate progressively**: ResourceTest → Pages → Schema/Form → Table → RelationManager
4. **Apply grouping**: all `assertFormFieldExists()` in one test, all `assertCanRenderTableColumn()` in one test
5. **Run incrementally**: `php artisan test --compact tests/Feature/[Resource]/` after each file
6. **Format**: `vendor/bin/pint --dirty`

## Expected Coverage

- ResourceTest: ~10
- Create/Edit/ListPage: ~15-20 each
- FormTest: ~20-25
- TableTest: ~35-45
- RelationManagerTest: ~25-35 per manager
- **Total: ~150-175 tests per resource**

## Final Checklist

- [ ] `Livewire::test()` (not `livewire()`)
- [ ] Enums: `instanceof` + `.value`
- [ ] Hidden columns: test config, not rendering
- [ ] Methods with params receive record
- [ ] Unique constraints: separate related records
- [ ] Functional approach via pages
- [ ] beforeEach with auth (+ tenant if multi-tenant)
- [ ] Factory relations configured
- [ ] `$this->assertDatabaseHas()`
- [ ] Forms call right action (`create`/`save`)
- [ ] **Sequential assertions of same method grouped**
- [ ] Tests with different logic/data separated
- [ ] Descriptive names
