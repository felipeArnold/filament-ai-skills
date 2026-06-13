---
name: null-safe-check
description: >-
  Verifies and fixes null-safe issues in PHP files before a commit.
  Activate when the user mentions: null safe, null-safe, ?->, commit, pre-commit,
  "before committing", "verify null", "fix null", ErrorException,
  "Attempt to read property", "Call to a member function on null".
---

# Null-Safe Check

## When to Apply

Activate this skill **before every commit** or when an error like this occurs:
- `ErrorException: Attempt to read property "X" on null`
- `Call to a member function X() on null`
- PHPStan: `Access to an undefined property ... ::$X`

## Verification Process

### 1. Identify files to check

```bash
# Staged files (for pre-commit)
git diff --name-only --cached | grep '\.php$'

# Modified files (working tree)
git diff --name-only | grep '\.php$'
```

### 2. Dangerous Patterns to Detect

Use `Grep` to find these patterns in the PHP files:

| Dangerous pattern | Safe pattern |
|-----------------|--------------|
| `->relation->property` | `->relation?->property` |
| `->relation->method()` | `->relation?->method()` |
| `fn ($x) => $x->rel->prop` | `fn ($x) => $x->rel?->prop` |
| `$group->first()->rel->prop` | `$group->first()->rel?->prop ?? 'default'` |
| `filter(fn ($i) => $i->rel->prop)` | `filter(fn ($i) => $i->rel?->prop)` |

**Search regex:**
```
->[a-zA-Z_]+->
```

### 3. High Risk Contexts

Prioritize these contexts — most prone to null:

- **Optional Eloquent relationships** (`nullable()` in migration, `HasOne`, `BelongsTo`)
- **Closures in `filter()`, `map()`, `groupBy()`** over collections
- **`$group->first()->relation->property`** — first item in group may have null relation
- **`getStateUsing(fn ($record) => $record->relation->property)`** in table columns
- **Decoded JSON data** — fields may be absent

### 4. Correction Rules

#### Rule 1 — Simple relation property access
```php
// ❌ Before
$record->author->name
$record->category->label
$record->vehicle->plate

// ✅ After
$record->author?->name
$record->category?->label
$record->vehicle?->plate
```

#### Rule 2 — Multiple chaining
```php
// ❌ Before
$group->first()->order->customer->name

// ✅ After — null-safe at each potentially null level
$group->first()->order?->customer?->name ?? 'N/A'
```

#### Rule 3 — Closures in filter/map
```php
// ❌ Before
->filter(fn ($post) => $post->author->active && $post->author->email)

// ✅ After
->filter(fn ($post) => $post->author?->active && $post->author?->email)
```

#### Rule 4 — getStateUsing in Filament
```php
// ❌ Before
->getStateUsing(fn ($record) => $record->order->status === 'paid' ? $record->amount : null)

// ✅ After
->getStateUsing(fn ($record) => $record->order?->status === 'paid' ? $record->amount : null)
```

#### Rule 5 — Do NOT add unnecessary null-safe
```php
// ⚠️ PHPStan warns: "Using nullsafe property access on non-nullable type"
// If the property is NOT NULL in the database and the relationship is mandatory,
// do not use ?-> — use -> directly.

// Check the migration to determine if the FK is nullable:
// $table->foreignId('author_id')->nullable()  → use ?->
// $table->foreignId('author_id')               → use ->
```

### 5. Verify with PHPStan after correction

```bash
./vendor/bin/phpstan analyse --level=5 {file}
```

### 6. Format with Pint after correction

```bash
vendor/bin/pint --dirty
```

## Full Workflow (Pre-Commit)

```
1. git diff --name-only --cached | grep '\.php$'
2. For each file:
   a. Grep for pattern `->[a-zA-Z_]+->` (non-null-safe chains)
   b. Analyze context of each match
   c. Check migration if FK is nullable
   d. Fix with Edit tool
3. vendor/bin/pint --dirty
4. php artisan test --compact (optional, if relevant tests exist)
5. Confirm no more ErrorException from null
```

## Generic Examples

### Filter closures (common pattern)
```php
// ❌ Causes ErrorException when item does not have 'author'
$posts->filter(fn ($post) => $post->author->active)

// ✅
$posts->filter(fn ($post) => $post->author?->active)
```

### Table columns with getStateUsing
```php
// ❌
->getStateUsing(fn ($record) => $record->order->type === 'sale' ? $record->amount : null)

// ✅
->getStateUsing(fn ($record) => $record->order?->type === 'sale' ? $record->amount : null)
```

### Grouped records with relations (chaining)
```php
// ❌
'author_name' => $group->first()->post->author->name,

// ✅
'author_name' => $group->first()->post?->author?->name ?? 'N/A',
```

## What NOT to do

- **Do not** add `?->` blindly everywhere — PHPStan will warn about unnecessary null-safe
- **Do not** use `?->` on relations defined as `NOT NULL` in the migration
- **Do not** remove `?->` from relations that are already correctly typed
- **Do not** write `?? null` — it is redundant; simply omit the `??`
