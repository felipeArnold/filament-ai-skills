---
name: eloquent-query-check
description: >-
  Verifies and fixes direct static calls on Eloquent models, ensuring they
  always go through ::query() before methods like find, where, create, all, etc.
  Activate when: query(), Model::find, Model::create, Model::where,
  Model::all, eloquent query, static model call, model without query, fix model.
---

# Eloquent Query Check

## Rule

All data access via Eloquent model **must go through `::query()`** before any method.

```php
// ❌ Wrong
Product::find($id)
Product::create([...])
Product::where('tenant_id', $id)->get()
Product::all()
Product::count()
Product::pluck('name', 'id')
Product::first()
Product::firstOrCreate([...])
Product::updateOrCreate([...])
Product::exists()

// ✅ Correct
Product::query()->find($id)
Product::query()->create([...])
Product::query()->where('tenant_id', $id)->get()
Product::query()->get()
Product::query()->count()
Product::query()->pluck('name', 'id')
Product::query()->first()
Product::query()->firstOrCreate([...])
Product::query()->updateOrCreate([...])
Product::query()->exists()
```

## Why use ::query()

- Returns `Builder` with correct type hints — PHPStan and IDEs infer the model type
- Allows clean chaining without static context ambiguity
- Total consistency — any reader knows it is a database query
- Avoids conflicts with Eloquent magic static methods

## Patterns that should NOT be converted

```php
// Keep as-is — these are not database queries
Model::factory()          // Test factories
Model::query()            // Already correct
Model::class              // Class constant
Model::make(...)          // Filament Action, not Eloquent
Model::observe(...)       // Observer registration
Model::getSelectComponent() // Custom static method
Model::from(...)          // Rarely, accepted
new Model()               // Direct instantiation
```

## Verification Process

### 1. Find files to check

```bash
# Staged files (pre-commit)
git diff --name-only --cached | grep '\.php$'

# Modified files (working tree)
git diff --name-only | grep '\.php$'
```

### 2. Grep for forbidden patterns

Run for each identified PHP file:

```
# Direct calls without ::query()
::(find|findOrFail|findMany|where|whereIn|whereNotIn|whereHas|with|all|count|create|first|firstOrCreate|firstOrNew|updateOrCreate|pluck|exists|sum|avg|min|max|get|paginate|simplePaginate|cursor|chunk|each|latest|oldest|orderBy|select|groupBy|having|join|leftJoin|rightJoin|insert|upsert)\(
```

Use the Grep tool with `output_mode: "content"` to see the exact lines.

**Additional filter:** Ignore lines where the pattern is already preceded by `::query()`.

### 3. Verify each match

For each line found, confirm:
- Is it a static call on an Eloquent Model? (`Model::method(`)
- Does it already have `::query()` before it? (e.g. `Model::query()->where(` — OK)
- Is it a custom non-Eloquent method? (check if it exists in the model as `static`)
- Is it in the context of a factory, observer, or provider? (skip)

### 4. Fix with Edit tool

```php
// Substitution pattern:
// Model::method(  →  Model::query()->method(

// Examples:
Contact::where('tenant_id', $id)    →  Contact::query()->where('tenant_id', $id)
Account::create([...])               →  Account::query()->create([...])
Order::find($orderId)                →  Order::query()->find($orderId)
Product::all()                       →  Product::query()->get()
```

### 5. Format after fixing

```bash
vendor/bin/pint --dirty
```

## Quick Checklist

- [ ] Grep for `Model::(find|create|where|all|count|first|pluck)` in modified PHP files
- [ ] Each match verified: is it Eloquent? Already has `::query()`?
- [ ] All direct calls fixed to `Model::query()->method()`
- [ ] `vendor/bin/pint --dirty` executed

## Common Examples

### Filament Actions (common error pattern)

```php
// ❌ Frequent in Actions and Forms
->options(fn () => User::pluck('name', 'id'))
->options(fn () => Category::all()->pluck('name', 'id'))
$product = Product::find($productId);

// ✅ Correct
->options(fn () => User::query()->pluck('name', 'id'))
->options(fn () => Category::query()->pluck('name', 'id'))
$product = Product::query()->find($productId);
```

### Dashboard / Computed Properties

```php
// ❌
$count = Order::count();
$total = Account::where('type', 'payables')->sum('amount');

// ✅
$count = Order::query()->count();
$total = Account::query()->where('type', 'payables')->sum('amount');
```

### Observers and Listeners

```php
// ❌
$existing = Product::where('barcode', $barcode)->first();

// ✅
$existing = Product::query()->where('barcode', $barcode)->first();
```

## What NOT to do

- ❌ Convert `Model::factory()` — test factories are not queries
- ❌ Convert `Model::class` — it is a PHP constant, not a method
- ❌ Convert custom static methods unrelated to queries
- ❌ Modify code that already uses `::query()` correctly
- ❌ Forget to run Pint after the fixes
