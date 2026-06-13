---
name: filament-action-development
description: >-
  Creates Filament v5 Actions — record, bulk, header, page actions — com modais, forms,
  confirmações e notificações. Activate: create action, action customizada, botão, modal,
  confirmação, dialog, bulk action, ação em lote, ação de tabela, ExportAction,
  ImportAction, gerar PDF a partir de botão, enviar email via action, atualizar status,
  agrupar ações.
---

# Filament Action Development

## When to Apply

- Nova action customizada (record/bulk/header/page)
- Modal forms, confirmações, wizards
- Operações em lote em registros selecionados
- Geração de PDF, envio de email, update de status via botão
- Visibilidade/autorização/lógica condicional
- Agrupar actions em dropdowns

## Docs

`search-docs` com queries `['action overview', 'action modals', 'grouping actions']`.

## Project Conventions (Crítico)

### Estrutura

Actions ficam dentro do resource:
```
app/Filament/Resources/{Group}/{ResourceName}/Actions/
├── {ActionName}Action.php
├── {ActionName}BulkAction.php
```

### Class conventions

- Todas `final`
- `declare(strict_types=1);`
- Estendem `Filament\Actions\Action`
- Config em `setUp()` (sempre `parent::setUp()` primeiro)
- Static `make()` com default name
- Form schemas em `getFormSchema()` separado
- Lógica de negócio em `execute()` separado
- Return types explícitos sempre

### Nomes

- Classes: PascalCase + `Action` (`MarkAsReceivedAction`)
- Bulk: sufixo `BulkAction` (`UpdateStatusBulkAction`)
- `make()` name: snake_case (`'mark_as_received'`)

## Record Action — Confirmação Simples

```php
declare(strict_types=1);

namespace App\Filament\Resources\Financial\AccountsReceivables\Actions;

use App\Models\Accounts\AccountsInstallments;
use Filament\Actions\Action;
use Filament\Notifications\Notification;

final class MarkAsReceivedAction extends Action
{
    protected function setUp(): void
    {
        parent::setUp();

        $this
            ->label('Marcar como Recebido')
            ->icon('heroicon-o-check-circle')
            ->color('success')
            ->hiddenLabel()
            ->tooltip('Marcar como recebido')
            ->visible(fn ($record) => ! $record->status->value)
            ->requiresConfirmation()
            ->modalHeading('Confirmar Recebimento')
            ->modalDescription('Deseja marcar esta parcela como recebida?')
            ->modalSubmitActionLabel('Confirmar')
            ->action(fn () => $this->execute());
    }

    public static function make(?string $name = 'mark_as_received'): static
    {
        return parent::make($name);
    }

    protected function execute(): void
    {
        /** @var AccountsInstallments|null $record */
        $record = $this->getRecord();

        if (! $record) {
            Notification::make()->title('Erro')->body('Registro nao encontrado.')->danger()->send();
            return;
        }

        $record->update(['status' => 1, 'paid_at' => now()]);

        Notification::make()
            ->title('Parcela recebida!')
            ->body('A parcela foi marcada como recebida com sucesso.')
            ->success()
            ->send();
    }
}
```

## Record Action — Modal Form

```php
declare(strict_types=1);

namespace App\Filament\Resources\Orders\Actions;

use App\Enum\Template\TemplateContext;
use App\Models\EmailTemplate;
use App\Models\Order;
use Filament\Actions\Action;
use Filament\Forms;
use Filament\Notifications\Notification;

final class SendOrderEmailAction extends Action
{
    protected function setUp(): void
    {
        parent::setUp();

        $this
            ->label('Enviar por Email')
            ->color('success')
            ->icon('heroicon-o-envelope')
            ->modalHeading('Enviar por Email')
            ->modalDescription('Selecione o template para enviar:')
            ->modalSubmitActionLabel('Enviar')
            ->modalWidth('md')
            ->form($this->getFormSchema())
            ->action(fn (array $data) => $this->execute($data));
    }

    public static function make(?string $name = 'send_email'): static
    {
        return parent::make($name);
    }

    protected function getFormSchema(): array
    {
        return [
            Forms\Components\Select::make('email_template_id')
                ->label('Template de E-mail')
                ->searchable()
                ->preload()
                ->required()
                ->options(fn (): array => EmailTemplate::query()
                    ->where('context', TemplateContext::Order->value)
                    ->where('is_active', true)
                    ->orderBy('name')
                    ->pluck('name', 'id')
                    ->all()
                )
                ->placeholder('Selecione um template'),
        ];
    }

    protected function execute(array $data): void
    {
        /** @var Order|null $record */
        $record = $this->getRecord();

        if (! $record) {
            Notification::make()->title('Erro')->body('Registro nao encontrado.')->danger()->send();
            return;
        }

        // Business logic...

        Notification::make()->title('E-mail enfileirado')->body('Envio adicionado à fila.')->success()->send();
    }
}
```

## Bulk Action

Múltiplos registros via `accessSelectedRecords()`:

```php
declare(strict_types=1);

namespace App\Filament\Resources\Orders\Actions;

use App\Enum\OrderStatus;
use App\Models\Order;
use Filament\Actions\Action;
use Filament\Forms;
use Filament\Notifications\Notification;
use Illuminate\Support\Collection;

final class UpdateStatusBulkAction extends Action
{
    protected function setUp(): void
    {
        parent::setUp();

        $this
            ->label('Atualizar Status')
            ->icon('heroicon-o-arrow-path')
            ->modalWidth('md')
            ->requiresConfirmation()
            ->color('info')
            ->accessSelectedRecords()
            ->form($this->getFormSchema())
            ->action(fn (array $data, Collection $records) => $this->execute($data, $records));
    }

    public static function make(?string $name = 'update_status'): static
    {
        return parent::make($name);
    }

    protected function getFormSchema(): array
    {
        return [
            Forms\Components\Select::make('status')
                ->label('Status')
                ->options(OrderStatus::toSelectArray())
                ->native(false)
                ->required(),
        ];
    }

    protected function execute(array $data, Collection $records): void
    {
        $records->each(fn (Order $r) => $r->update(['status' => $data['status']]));

        Notification::make()
            ->title('Status atualizado com sucesso')
            ->success()
            ->body("O status de {$records->count()} registro(s) foi atualizado.")
            ->send();
    }
}
```

## Action com Transaction

Múltiplas tabelas → `DB::transaction()`:

```php
declare(strict_types=1);

namespace App\Filament\Resources\MaintenancePlans\Actions;

use App\Models\Order;
use Filament\Actions\Action;
use Filament\Notifications\Notification;
use Illuminate\Support\Facades\DB;

final class CreateBudgetAction extends Action
{
    protected function setUp(): void
    {
        parent::setUp();

        $this
            ->label('Criar Orcamento')
            ->icon('heroicon-o-document-text')
            ->color('success')
            ->visible(fn ($record) => ! $record->order_id)
            ->requiresConfirmation()
            ->modalHeading('Criar Orcamento de Manutencao')
            ->modalDescription(fn ($record) => "Deseja criar um orcamento para '{$record->maintenanceType->name}'?")
            ->modalSubmitActionLabel('Criar Orcamento')
            ->action(fn () => $this->execute());
    }

    public static function make(?string $name = 'create_budget'): static
    {
        return parent::make($name);
    }

    protected function execute(): void
    {
        $record = $this->getRecord();

        DB::transaction(function () use ($record): void {
            $order = Order::create([/* ... */]);
            $record->update(['order_id' => $order->id]);

            Notification::make()
                ->success()
                ->title('Orcamento criado!')
                ->body("Pedido #{$order->id} criado com sucesso.")
                ->send();
        });
    }
}
```

## Registering Actions

### Record (linha da tabela) — `recordActions()`

```php
->recordActions([
    ViewAction::make(),
    EditAction::make(),
    MarkAsReceivedAction::make(),
    SendOrderEmailAction::make(),
])
```

### Bulk (selecionados) — `toolbarActions()` + `BulkActionGroup`

```php
use Filament\Actions\BulkActionGroup;

->toolbarActions([
    BulkActionGroup::make([
        UpdateStatusBulkAction::make(),
        ExportPdfBulkAction::make(),
    ]),
])
```

> ⚠️ `->bulkActions([...])` era v3. **v5: `->toolbarActions([BulkActionGroup::make([...])])`**.

### Header (toolbar da tabela) — `headerActions()`

```php
->headerActions([
    Action::make('view_all')
        ->label('Ver Todos')
        ->url(SomeResource::getUrl('index'))
        ->icon('heroicon-o-eye'),
])
```

### Page — `getHeaderActions()`

```php
protected function getHeaderActions(): array
{
    return [
        EditAction::make(),
        DeleteAction::make(),
        SendEmailAction::make(),
    ];
}
```

## Agrupar em Dropdown

```php
use Filament\Actions\ActionGroup;

->recordActions([
    ViewAction::make(),
    EditAction::make(),
    ActionGroup::make([
        SendEmailAction::make(),
        DownloadPdfAction::make(),
        DeleteAction::make(),
    ])
        ->icon('heroicon-m-ellipsis-vertical')
        ->tooltip('Mais acoes'),
])
```

## Referência

### Trigger

| Método | Descrição |
|---|---|
| `->button()` | Botão com fundo (default) |
| `->link()` | Link inline |
| `->iconButton()` | Botão circular só icon |
| `->badge()` | Badge com fundo colorido |
| `->hiddenLabel()` | Esconde label, só icon |

### Modal

| Método | Descrição |
|---|---|
| `->requiresConfirmation()` | Diálogo confirmação |
| `->modalHeading('Title')` | Título |
| `->modalDescription('Text')` | Descrição |
| `->modalSubmitActionLabel('OK')` | Texto submit |
| `->modalWidth('md')` | Largura (sm, md, lg, xl, 2xl) |
| `->slideOver()` | Slide panel em vez de modal |
| `->form([...])` | Campos do modal |
| `->stickyModalHeader()` / `->stickyModalFooter()` | Header/footer fixos |
| `->closeModalByClickingAway(false)` | Previne fechar acidental |

### Visibilidade & Autorização

| Método | Descrição |
|---|---|
| `->visible(fn ($record) => ...)` | Condicional |
| `->hidden(fn ($record) => ...)` | Esconder condicional |
| `->disabled(fn ($record) => ...)` | Desabilitar condicional |
| `->authorize('update')` | Policy |
| `->tooltip('Hint text')` | Tooltip hover |

### Cores

`'primary'`, `'secondary'`, `'success'`, `'danger'`, `'warning'`, `'info'`, `'gray'`.
Custom: `Filament\Support\Colors\Color` (ex: `Color::Emerald`).

### Injeção de Utilities

| Param | Descrição |
|---|---|
| `$record` | Model atual (record actions) |
| `$data` | Form data (com `form()`) |
| `Collection $records` | Selecionados (bulk com `accessSelectedRecords()`) |
| `$action` | Action instance |
| `$livewire` | Componente Livewire pai |
| `$arguments` | Args custom passados |

## Built-In Actions

Namespace `Filament\Actions\`:

| Action | Uso |
|---|---|
| `CreateAction` | Criar |
| `EditAction` | Editar |
| `ViewAction` | Ver |
| `DeleteAction` | Deletar |
| `ReplicateAction` | Duplicar |
| `ForceDeleteAction` | Force delete (soft-deleted) |
| `RestoreAction` | Restaurar soft-deleted |
| `ImportAction` | Import bulk |
| `ExportAction` | Export |

## New Action Checklist

1. Criar em `Actions/` do resource
2. Estender `Filament\Actions\Action`
3. `final` + `declare(strict_types=1);`
4. `setUp()` com `parent::setUp()`
5. Static `make()` com snake_case default
6. `getFormSchema()` se tiver modal form
7. Lógica em `execute()`
8. Erro com `Notification::make()->danger()`
9. Sucesso com `Notification::make()->success()`
10. `DB::transaction()` em multi-tabela
11. Registrar (recordActions / bulk via BulkActionGroup / headerActions / page getHeaderActions)
12. `vendor/bin/pint --dirty`

## Money Formatting

**OBRIGATÓRIO formatação consistente** — nunca `number_format()` direto.

```php
use Illuminate\Support\Number;

// ❌ number_format($value, 2, ',', '.')
// ✅
Number::currency($value, 'BRL')                          // "R$ 1.234,56"
number_format($value, 2, ',', '.')                       // "1.234,56"
number_format($value, 2, ',', '.') . '%'                 // "12,50%"

->formatStateUsing(fn ($state) => $state ? Number::currency($state, 'BRL') : '—')
```

## TDD Obrigatório

**REGRA:** teste primeiro, sem exceção.

```
1. Teste falha (RED)
2. Implementar action
3. Teste passa (GREEN)
4. vendor/bin/pint --dirty
```

### O que testar

```php
// 1. Efeito esperado
test('action X faz Y', function (): void {
    Livewire::test(ViewFoo::class, ['record' => $record->id])
        ->callAction('action_name', data: ['field' => 'value'])
        ->assertNotified();
    expect($record->refresh()->field)->toBe('value');
});

// 2. Visibilidade
test('action oculta quando condição não satisfeita', function (): void {
    Livewire::test(ViewFoo::class, ['record' => $record->id])
        ->assertActionHidden('action_name');
});

// 3. Action de tabela com data escolhida (não now() automático)
test('action de tabela salva data escolhida', function (): void {
    Livewire::test(ListFoo::class)
        ->callTableAction('action_name', $record, data: ['paid_at' => '2024-01-15'])
        ->assertNotified();
    expect($record->refresh()->paid_at->format('Y-m-d'))->toBe('2024-01-15');
});

// 4. Bulk com accessSelectedRecords
test('bulk processa todos selecionados', function (): void {
    Livewire::test(ListFoo::class)
        ->callBulkAction('bulk_action_name', $records, data: ['field' => 'value']);
    $records->each(fn ($r) => expect($r->refresh()->field)->toBe('value'));
});
```

Arquivo: `tests/Feature/{Domain}/{Name}/Actions/{ActionName}Test.php`
```bash
php artisan make:test --pest Feature/{Domain}/{Name}/Actions/{ActionName}Test
php artisan test --compact --filter={ActionName}Test
```

---

## Common Pitfalls

- Esquecer `parent::setUp()`
- Namespaces errados (`Filament\Tables\Actions\` em vez de `Filament\Actions\`)
- Bulk sem `accessSelectedRecords()`
- Lógica em closures de `setUp()` em vez de `execute()` separado
- Sem tratar `$this->getRecord()` null
- Sem `DB::transaction()` em multi-tabela
- Select em modal sem `->native(false)`
- `->bulkActions()` (v3) em vez de `->toolbarActions([BulkActionGroup::make([...])])` (v5)
- Sem notificação ao final
- Sem `->requiresConfirmation()` em destrutivas
- `number_format()` em vez de `Number::currency()`
