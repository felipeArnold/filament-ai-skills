---
name: filament-infolist-development
description: >-
  Creates and maintains Infolists Filament v5 para visualização read-only de registros.
  Activate when: infolist, view page, visualização, página de
  detalhes, exibir dados, read-only, ver registro, tela de detalhes, entries,
  TextEntry, IconEntry, RepeatableEntry, layout de visualização.
---

# Filament Infolist Development — v5

Infolists são a contraparte read-only dos Forms. Usados na `ViewRecord` page para exibir dados de forma organizada e visual.

---

## Namespaces obrigatórios

```php
// Entries (componentes de dado)
use Filament\Infolists\Components\TextEntry;
use Filament\Infolists\Components\IconEntry;
use Filament\Infolists\Components\ImageEntry;
use Filament\Infolists\Components\ColorEntry;
use Filament\Infolists\Components\RepeatableEntry;
use Filament\Infolists\Components\KeyValueEntry;

// Enums de suporte
use Filament\Support\Enums\TextSize; // Para ->size()

// Layout (MESMOS namespaces do Form)
use Filament\Schemas\Components\Section;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Tabs;
use Filament\Schemas\Components\Tabs\Tab;
use Filament\Schemas\Components\Fieldset;
use Filament\Schemas\Schema;
```

> **CRÍTICO:** Entries ficam em `Filament\Infolists\Components\`. Layout fica em `Filament\Schemas\Components\`. **Nunca use** `Filament\Forms\Components\TextInput` em um infolist.

---

## Padrão deste projeto

```php
final class XInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema
            ->components([
                Tabs::make('tabs')
                    ->columnSpanFull()
                    ->tabs([
                        Tab::make('general')
                            ->label('Geral')
                            ->icon('heroicon-o-information-circle')
                            ->schema([
                                Grid::make(2)
                                    ->schema([
                                        Section::make('Seção A')->compact()->schema([...]),
                                        Section::make('Seção B')->compact()->schema([...]),
                                    ])->columnSpanFull(),
                            ]),
                    ]),
            ]);
    }
}
```

> **COMPONENTES INEXISTENTES — NUNCA USE:**
> - `Filament\Schemas\Components\Split` — **não existe**, use `Grid::make(N)`
> - `->grow(false)` — **não existe em Grid**, não tem equivalente direto
> - `Filament\Forms\Components\*` — são para forms, não infolists

---

## TextEntry — Common Patterns

### Texto simples
```php
TextEntry::make('name')
    ->label('Nome')
    ->weight('bold')
    ->placeholder('—'),
```

### Badge com Enum (HasColor + HasLabel)
```php
TextEntry::make('status')
    ->label('Status')
    ->badge()
    ->color(fn (StatusEnum $state): string => $state->getColor()),
```

### Valor monetário (usando FormatterHelper)
```php
TextEntry::make('total_value')
    ->label('Total')
    ->formatStateUsing(fn ($state) => FormatterHelper::money((float) $state, currency: true))
    ->weight('bold')
    ->size(TextSize::Large)
    ->color('success'),
```

### Data e Hora
```php
TextEntry::make('created_at')->label('Criado em')->dateTime('d/m/Y H:i'),
TextEntry::make('opening_date')->label('Abertura')->date('d/m/Y'),
```

### HTML (de RichEditor)
```php
TextEntry::make('description')->label('Descrição')->html()->placeholder('Não informado.'),
```

### Copyable
```php
TextEntry::make('number')->label('Número')->copyable(),
```

### URL / Link
```php
TextEntry::make('person.name')
    ->label('Cliente')
    ->url(fn ($record) => $record->person_id
        ? route('filament.app.resources.creates.people.edit', ['record' => $record->person_id, 'tenant' => $record->tenant_id])
        : null),
```

> Use `route()` com o nome exato da rota. Verifique rotas disponíveis com `php artisan route:list --name=<resource>`. Muitos resources não têm rota `view`, apenas `edit`.

### Estado calculado (virtual)
```php
TextEntry::make('vehicle_info')
    ->label('Modelo')
    ->state(fn ($record) => $record->vehicle
        ? "{$record->vehicle->brand} {$record->vehicle->model}"
        : null)
    ->placeholder('—'),
```

### Formatação customizada
```php
TextEntry::make('document')
    ->label('CPF / CNPJ')
    ->formatStateUsing(fn ($state) => $state ? FormatterHelper::cpfCnpj($state) : '—'),

TextEntry::make('mileage')
    ->label('Km')
    ->formatStateUsing(fn ($state) => $state ? number_format($state, 0, ',', '.') . ' km' : '—'),
```

### Tamanhos e pesos
```php
// use Filament\Support\Enums\TextSize;
->size(TextSize::Small)    // small
->size(TextSize::Medium)   // default
->size(TextSize::Large)    // large
->weight('thin' | 'extralight' | 'light' | 'normal' | 'medium' | 'semibold' | 'bold' | 'extrabold' | 'black')
```

> **NUNCA use** `TextEntry\TextEntrySize::*` — não existe. Use `Filament\Support\Enums\TextSize`.

---

## IconEntry — Booleanos e ícones

```php
IconEntry::make('appointment_confirmed')
    ->label('Confirmado')
    ->boolean()
    ->trueIcon('heroicon-o-check-circle')
    ->falseIcon('heroicon-o-x-circle')
    ->trueColor('success')
    ->falseColor('danger'),
```

---

## RepeatableEntry — Listas de relacionamentos

```php
RepeatableEntry::make('serviceOrderServices')
    ->hiddenLabel()
    ->schema([
        TextEntry::make('service_name')->label('Serviço')->weight('semibold'),
        TextEntry::make('quantity')->label('Qtd')->alignCenter(),
        TextEntry::make('total')
            ->label('Total')
            ->formatStateUsing(fn ($state) => FormatterHelper::money((float) $state, currency: true))
            ->color('success'),
    ])
    ->columns(3)
    ->columnSpanFull(),
```

---

## Layout — Grid com Sections lado a lado

`Split` **não existe** em `Filament\Schemas\Components`. Use `Grid` para colocar sections lado a lado:

```php
// 2 sections lado a lado
Grid::make(2)
    ->schema([
        Section::make('Identificação')->compact()->schema([...]),
        Section::make('Detalhes')->compact()->schema([...]),
    ])->columnSpanFull(),

// 3 sections lado a lado
Grid::make(3)
    ->schema([
        Section::make('Cliente')->compact()->schema([...]),
        Section::make('Veículo')->compact()->schema([...]),
        Section::make('Responsáveis')->compact()->schema([...]),
    ])->columnSpanFull(),
```

---

## Layout — Section

```php
Section::make('Título da Seção')
    ->icon('heroicon-o-information-circle')
    ->description('Texto de apoio')
    ->compact()          // menos padding
    ->collapsed()        // começa colapsado
    ->collapsible()      // permite colapsar
    ->schema([...])
    ->columns(2)         // grid interno
    ->columnSpanFull(),  // ocupa toda a largura
```

---

## Layout — Grid

```php
Grid::make(3)
    ->schema([
        TextEntry::make('a'),
        TextEntry::make('b'),
        TextEntry::make('c'),
    ]),
```

---

## Layout — Tabs

```php
Tabs::make('tabs')
    ->columnSpanFull()
    ->tabs([
        Tab::make('geral')
            ->label('Geral')
            ->icon('heroicon-o-clipboard-document-list')
            ->schema([...]),

        Tab::make('financeiro')
            ->label('Financeiro')
            ->icon('heroicon-o-banknotes')
            ->schema([...]),
    ]),
```

---

## Conectar ao Resource

No Resource, o método `infolist()` delega para a classe:

```php
public static function infolist(Schema $schema): Schema
{
    return XInfolist::configure($schema);
}
```

Na página `ViewRecord`, o infolist é renderizado automaticamente.

---

## Padrão de values monetários neste projeto

**NUNCA usar** `->money('BRL')` diretamente (não formata em pt-BR). Usar sempre:

```php
->formatStateUsing(fn ($state) => FormatterHelper::money((float) $state, currency: true))
```

Para exibir sem símbolo `R$`:
```php
->formatStateUsing(fn ($state) => FormatterHelper::money((float) $state, currency: false) . '%')
```

---

## Quality rules

- Todas as classes são `final`
- Usar `->placeholder('—')` em campos nullable
- Usar `->compact()` nas sections para layout mais denso
- Usar `Grid::make(N)` para colocar sections lado a lado (ex: `Grid::make(2)` para Cliente + Veículo)
- Usar `Tabs` para organizar muitos dados em seções temáticas
- Usar `->html()` nos campos vindos de `RichEditor`
- Usar `->badge()` + `->color()` para Enums com `HasColor`
- Usar `->state()` para campos calculados (não persistidos)
- Nunca usar `->money('BRL')` — sempre `FormatterHelper::money()`
- **Nunca usar `Split`** — não existe em `Filament\Schemas\Components\`, use `Grid::make(N)`
