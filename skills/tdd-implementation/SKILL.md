---
name: tdd-implementation
description: >-
  Implements new features following the TDD Red → Green → Refactor cycle with Pest 4,
  Laravel 12 and Filament v5. Activate when: implement feature, TDD, test-driven,
  red-green cycle, test first, create feature with tests, new functionality,
  write test before, "implement X with tests". Includes step-by-step protocol
  (write test → red → implement → green → pint).
---

# TDD Implementation Skill

## When to Apply

Activate this skill **every time you implement a new feature**, following the Red → Green → Refactor cycle. This skill works with Pest 4, Laravel 12 and Filament v5.

---

## TDD Cycle

```
1. RED   — Write a failing test
2. GREEN — Write the minimum code to make it pass
3. PINT  — Run vendor/bin/pint --dirty
4. NEXT  — Repeat for the next behaviour
```

**Never skip the Red phase.** Run the test before writing the implementation to confirm it fails for the right reason.

---

## Step-by-Step Protocol

### Step 1 — Understand Scope

Before writing any test:
- Read sibling models/resources to understand existing conventions
- Check if a factory/migration already exists for the model
- Identify which layers to test: unit (pure logic) vs feature (DB + Filament)

### Step 2 — Create Test Files

```bash
# Feature test (most tests should be feature tests)
php artisan make:test --pest Feature/{Domain}/{NameTest}

# Unit test (pure logic — no DB, no factories)
php artisan make:test --pest --unit Unit/{Domain}/{NameTest}
```

**Fix double-nesting** from artisan: if artisan creates `tests/Feature/Feature/...`, move the file manually.

### Step 3 — Write Failing Tests First

#### Unit Test Pattern (pure logic — no DB)
```php
test('model has correct mass-assignment protection', function (): void {
    $model = new MyModel();
    // Check per your project convention: $guarded or $fillable
    expect($model->getGuarded())->toBe([]);
});

test('model has method doSomething', function (): void {
    expect(method_exists(MyModel::class, 'doSomething'))->toBeTrue();
});
```

#### Feature Test Pattern (DB + Filament)
```php
uses(Illuminate\Foundation\Testing\RefreshDatabase::class);

beforeEach(function (): void {
    $this->user = User::factory()->create();
    $this->actingAs($this->user);
    // Multi-tenant: also set up tenant and Filament::setTenant($this->tenant)
});

test('action creates record', function (): void {
    Livewire::test(ViewPage::class, ['record' => $this->record->id])
        ->callAction('myAction', ['field' => 'value'])
        ->assertNotified();

    expect(MyModel::query()->where('field', 'value')->exists())->toBeTrue();
});
```

### Step 4 — Run Tests (Confirm Red)

```bash
php artisan test --compact --filter=MyFeatureTest
```

Confirm tests fail with **the expected error** (class not found, method not found, assertion failure — not a PHP syntax error).

### Step 5 — Implement Minimum Code

Create only what's needed to turn tests green:
- Migration
- Model with mass-assignment protection per project convention, casts, relations, scopes
- Factory with realistic defaults and named states
- Filament form/infolist/action changes

### Step 6 — Run Tests (Confirm Green)

```bash
php artisan test --compact --filter=MyFeatureTest
```

All tests must pass. If a test fails, fix the implementation — never change the test to fit wrong code.

### Step 7 — Run Pint

```bash
vendor/bin/pint --dirty
```

Then run tests again to confirm pint didn't break anything.

### Step 8 — Run Regression Check

```bash
php artisan test --compact tests/Feature/{Domain}/
```

Confirm no existing tests broke.

---

## File Conventions

| File type | Location | Command |
|-----------|----------|---------|
| Model | `app/Models/` | `php artisan make:model ModelName --factory` |
| Migration | `database/migrations/` | `php artisan make:migration create_table_name_table` |
| Factory | `database/factories/` | auto-created with `--factory` |
| Enum | `app/Enum/ClassName.php` | `php artisan make:class App/Enum/ClassName` → convert to enum |
| Filament Page | `app/Filament/{Panel}/Pages/` | manual |
| Feature Test | `tests/Feature/{Domain}/` | `php artisan make:test --pest Feature/...` |
| Unit Test | `tests/Unit/{Domain}/` | `php artisan make:test --pest --unit Unit/...` |

---

## Model Checklist

Every new model must have:
- [ ] `declare(strict_types=1)`
- [ ] `final class`
- [ ] `/** @use HasFactory<XFactory> */`
- [ ] Mass-assignment protection per project convention (`$guarded = []` or `$fillable = [...]`)
- [ ] `casts()` method (not `$casts` property) with explicit types
- [ ] Return type hints on all relationship methods
- [ ] Scopes as `scopeName(Builder $query): Builder`

---

## Filament Integration Tests

### Asserting form fields exist
```php
Livewire::test(EditPage::class, ['record' => $record->id])
    ->assertFormFieldExists('field_name');
```

### Asserting actions exist and work
```php
Livewire::test(ViewPage::class, ['record' => $record->id])
    ->assertActionExists('actionName')
    ->callAction('actionName', ['field' => 'value'])
    ->assertNotified();
```

### Asserting infolist content
```php
Livewire::test(ViewPage::class, ['record' => $record->id])
    ->assertSee('Expected text');
```

---

## What NOT to do

- Do not write implementation before the test
- Do not make tests pass by weakening assertions
- Do not put DB tests in Unit test files (no DB in Unit)
- Do not skip `RefreshDatabase` on feature tests that use factories
- Do not use `assertStatus(200)` — use `assertSuccessful()`
- Do not run `pint --test` — run `pint` to fix directly

---

## Running Tests

```bash
# Single filter
php artisan test --compact --filter=TestName

# Full feature directory
php artisan test --compact tests/Feature/{Domain}/

# All tests
php artisan test --compact
```
