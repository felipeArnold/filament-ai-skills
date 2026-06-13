---
name: filament-resource-development
description: >-
  Creates and maintains Filament v5 Resources (Resource, Form Schema, Table, Pages,
  RelationManager, Widget, Infolist). Activate when: create resource, filament resource,
  new CRUD, form schema, table, pages, relation manager, widget, panel navigation,
  register resource, PanelProvider, Filament plugin.
  Covers Filament v5 (Livewire 4 + Tailwind 4). For version upgrades use filament-upgrade.
---

# Filament v5 — Resource Development

> Skill for Filament **v5** (Livewire 4 + Tailwind 4). Verify before applying:
> ```bash
> composer show filament/filament | grep versions
> ```
> If `5.x.x` → ok. If `^4.0` or `^3.0` → activate `filament-upgrade` first.

## Requirements

| Dep | Version |
|---|---|
| PHP | 8.2+ |
| Laravel | 11.28+ |
| Livewire | **4.x** |
| Tailwind | **4.x** |

## Modular Structure (v5 — mandatory pattern)

```
app/Filament/Resources/
└── Posts/
    ├── PostResource.php       # delegates form/table
    ├── Pages/                 # Create/Edit/List
    ├── Schemas/PostForm.php   # extracted form
    └── Tables/PostsTable.php  # extracted table
```

Convention: `{Model}Form` (singular, `Schemas/`), `{Model}sTable` (plural, `Tables/`).

## Namespaces v5 — Cheatsheet

| Item | Namespace |
|---|---|
| Schema base | `Filament\Schemas\Schema` |
| Layout (Section, Grid, Tabs, Wizard…) | `Filament\Schemas\Components\*` |
| Form fields (TextInput, Select…) | `Filament\Forms\Components\*` |
| Tables | `Filament\Tables\*` |
| Actions (all) | `Filament\Actions\*` |
| Login | `Filament\Auth\Pages\Login` |
| Set/Get | `Filament\Schemas\Components\Utilities\{Set,Get}` |
| Heroicons | `Filament\Support\Icons\Heroicon` (enum) |
| Width | `Filament\Support\Enums\Width` (enum) |

**Critical**:
- `Form` → `Schema` in `form()` and `infolist()`.
- Layout components: `Schemas\Components`, **not** `Forms\Components`.
- Actions: `Filament\Actions\*` (centralized). **Do not use** `Filament\Tables\Actions\*`.
- Login moved to `Filament\Auth\Pages\Login` (v3/v4 was `Filament\Pages\Auth\Login`).
- `$view` on Page: `protected string` (NOT static).

## Resource Class — Pattern (delegates to Schemas/Tables)

```php
<?php

declare(strict_types=1);

namespace App\Filament\Resources\Posts;

use App\Filament\Resources\Posts\Pages;
use App\Filament\Resources\Posts\Schemas\PostForm;
use App\Filament\Resources\Posts\Tables\PostsTable;
use App\Models\Post;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Tables\Table;

final class PostResource extends Resource
{
    protected static ?string $model = Post::class;
    protected static string|\BackedEnum|null $navigationIcon = Heroicon::OutlinedDocumentText;
    protected static string|\UnitEnum|null $navigationGroup = 'Content';

    public static function form(Schema $schema): Schema
    {
        return PostForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return PostsTable::configure($table);
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListPosts::route('/'),
            'create' => Pages\CreatePost::route('/create'),
            'edit' => Pages\EditPost::route('/{record}/edit'),
        ];
    }
}
```

## Form Schema — Mandatory conventions

```php
<?php

declare(strict_types=1);

namespace App\Filament\Resources\Posts\Schemas;

use Filament\Forms\Components\TextInput;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;

final class PostForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Section::make('Content')
                ->schema([
                    TextInput::make('title')
                        ->label('Title')
                        ->required()
                        ->maxLength(255),
                ])->columns(2),
        ]);
    }
}
```

Rules:
- `declare(strict_types=1);` always
- `final class` always
- `->label('...')` on all fields
- `->required()` — **NEVER** `->rules(['required'])`
- Explicit imports (no generic `use Filament\Forms;`)

## Artisan commands

```bash
php artisan make:filament-resource Post                # modular structure
php artisan make:filament-resource Post --generate     # auto-generate from DB
php artisan make:filament-resource Post --simple       # modal
php artisan make:filament-resource Post --soft-deletes # with soft-deletes
```

## TDD — mandatory

Write/update tests before creating/modifying a resource.

```bash
php artisan make:test --pest Feature/Posts/PostResourceTest
php artisan test --compact --filter=PostResourceTest
```

## Common pitfalls

- Wrong namespace: `Filament\Tables\Actions\` instead of `Filament\Actions\`
- `Form` instead of `Schema` in method signature
- Static `$view` on Page (must be non-static `protected string`)
- Layout components from `Forms\Components` (must be `Schemas\Components`)
- Missing `declare(strict_types=1);` or `final class`
