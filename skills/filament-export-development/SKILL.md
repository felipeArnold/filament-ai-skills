---
name: filament-export-development
description: >-
  Adds Excel and PDF export a tabelas Filament. Cria Exporter classes pra Excel
  (via ExportAction + openspout) e BulkAction + Controller + Blade pra PDF (via DomPDF).
  Activate: export, exportação, Excel, planilha, relatório PDF de lista, download de dados,
  export.
---

# Filament Export Development

## When to Apply

- Botão de exportação Excel em tabela Filament
- Relatório PDF de lista (múltiplos registros)
- Bulk action de PDF pra selecionados
- Exportar dados filtrados pra planilha

---

## Arquitetura

Duas abordagens:

| Tipo | Mecanismo | Aciona em |
|---|---|---|
| **Excel** | `ExportAction` + `Exporter` + OpenSpout | Toolbar da tabela (respeita filtros) |
| **PDF** | `BulkAction` → Controller → DomPDF → Blade | Bulk action (selecionados) |

Coexistem no mesmo toolbar:

```php
->toolbarActions([
    ExportAction::make()
        ->label('Exportar Excel')
        ->icon('heroicon-o-table-cells')
        ->color('gray')
        ->exporter(MyModelExporter::class),
    BulkActionGroup::make([
        ExportPdfBulkAction::make(),
        DeleteBulkAction::make(),
    ]),
])
```

---

## PARTE 1 — Excel com ExportAction

### 1.1 Classe Exporter

`app/Filament/Exports/{ModelName}Exporter.php`.

```php
<?php
declare(strict_types=1);

namespace App\Filament\Exports;

use App\Models\MyModel;
use Filament\Actions\Exports\ExportColumn;
use Filament\Actions\Exports\Exporter;
use Filament\Actions\Exports\Models\Export;
use Illuminate\Support\Number;

final class MyModelExporter extends Exporter
{
    protected static ?string $model = MyModel::class;

    public static function getColumns(): array
    {
        return [
            ExportColumn::make('id')->label('ID'),
            ExportColumn::make('name')->label('Nome'),
            ExportColumn::make('amount')
                ->label('Valor (R$)')
                ->formatStateUsing(fn ($state) => number_format((float) $state, 2, ',', '.')),
            ExportColumn::make('due_date')
                ->label('Vencimento')
                ->formatStateUsing(fn ($state) => $state?->format('d/m/Y')),
            ExportColumn::make('status')
                ->label('Status')
                ->formatStateUsing(fn ($state) => $state?->getLabel() ?? '-'),
            ExportColumn::make('relation.field')->label('Campo Relacionado'),
        ];
    }

    public static function getCompletedNotificationBody(Export $export): string
    {
        $body = 'A exportação foi concluída com '.Number::format($export->successful_rows).' '.str('registro')->plural($export->successful_rows).' exportado(s).';

        if ($failedRowsCount = $export->getFailedRowsCount()) {
            $body .= ' '.Number::format($failedRowsCount).' '.str('registro')->plural($failedRowsCount).' falhou ao exportar.';
        }

        return $body;
    }
}
```

### 1.2 ExportAction no toolbar

```php
use App\Filament\Exports\MyModelExporter;
use Filament\Actions\ExportAction;

->toolbarActions([
    ExportAction::make()
        ->label('Exportar Excel')
        ->icon('heroicon-o-table-cells')
        ->color('gray')
        ->exporter(MyModelExporter::class),
    BulkActionGroup::make([
        // bulk actions
    ]),
])
```

### Behavior

- Exporta **todos registros filtrados** visíveis
- Arquivo via **notificação** no painel após job
- Usa **fila** (background)
- Formatos: **Excel (.xlsx)** ou **CSV (.csv)**
- Migration `exports` já rodada (`php artisan migrate:status | grep export`)

### Relationships

Dot notation funciona:
```php
ExportColumn::make('accounts.person.name')->label('Cliente'),
ExportColumn::make('accounts.payment_method')
    ->label('Forma de Pagamento')
    ->formatStateUsing(fn ($state) => $state?->getLabel() ?? '-'),
```

### Conditional Logic

`formatStateUsing` com record completo:
```php
ExportColumn::make('status')
    ->label('Status')
    ->formatStateUsing(function ($state, MyModel $record): string {
        if ($state?->value === 1) return 'Pago';
        if ($record->due_date?->isPast()) return 'Vencido';
        return 'Pendente';
    }),
```

---

## PARTE 2 — PDF de Lista com BulkAction

### 2.1 BulkAction

`app/Filament/Resources/{Group}/{Resource}/Actions/Export{Resource}PdfBulkAction.php`.

```php
<?php
declare(strict_types=1);

namespace App\Filament\Resources\Financial\AccountsReceivables\Actions;

use Filament\Actions\BulkAction;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Support\Facades\Session;

final class ExportMyModelPdfBulkAction extends BulkAction
{
    protected function setUp(): void
    {
        parent::setUp();

        $this
            ->label('Exportar PDF')
            ->icon('heroicon-o-document-arrow-down')
            ->color('primary')
            ->action(function (Collection $records): void {
                $ids = $records->pluck('id')->toArray();
                Session::put('pdf_export_ids', $ids);

                $this->redirect(route('my-model.list.pdf', [
                    'ids' => implode(',', $ids),
                ]), navigate: false);
            })
            ->deselectRecordsAfterCompletion();
    }

    public static function make(?string $name = 'export_pdf'): static
    {
        return parent::make($name);
    }
}
```

### 2.2 Controller

`app/Http/Controllers/MyModelsListPdfController.php`.

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\MyModel;
use Barryvdh\DomPDF\Facade\Pdf;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;

final class MyModelsListPdfController extends Controller
{
    public function generate(Request $request): StreamedResponse
    {
        $idsParam = $request->get('ids');
        $ids = $idsParam ? explode(',', $idsParam) : [];

        $query = MyModel::query()->with(['relation1', 'relation2', 'tenant']);
        if (! empty($ids)) {
            $query->whereIn('id', $ids);
        }
        $records = $query->orderBy('created_at')->get();

        $tenant = $records->first()?->tenant ?? auth()->user()?->tenants()->first();
        $totalAmount = $records->sum('amount');

        $fileName = 'my-model-'.now()->format('Y-m-d-His').'.pdf';

        return Pdf::loadView('pdf.my-model-list', [
            'records' => $records,
            'totalAmount' => $totalAmount,
            'tenant' => $tenant,
        ])->stream($fileName);
    }
}
```

### 2.3 Rota

`routes/web.php`:
```php
use App\Http\Controllers\MyModelsListPdfController;

Route::middleware(['auth'])->prefix('my-model')->name('my-model.')->group(function (): void {
    Route::get('/list/pdf', [MyModelsListPdfController::class, 'generate'])->name('list.pdf');
});
```

### 2.4 Blade

`resources/views/pdf/my-model-list.blade.php` — seguir padrão `accounts-receivable-list.blade.php`:

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <style>
        .invoice-box {
            width: 100%; border: 1px solid #000; font-size: 10px; line-height: 14px;
            font-family: 'Helvetica', Helvetica, Arial, sans-serif; color: #000;
            margin-bottom: 15px; border-collapse: collapse;
        }
        .invoice-box td { padding: 4px; vertical-align: top; }
        .header-title {
            font-size: 14px; font-weight: bold; background-color: #e0e0e0;
            border-bottom: 1px solid #000; padding: 5px; text-transform: uppercase;
        }
        .info-label { font-weight: bold; color: #333; }
        .info-value { color: #000; }
        .table-datagrid { width: 100%; border-collapse: collapse; font-size: 9px; }
        .table-datagrid th {
            background-color: #e0e0e0; border: 1px solid #000; padding: 4px;
            text-align: left; font-weight: bold; text-transform: uppercase;
        }
        .table-datagrid td { border: 1px solid #000; padding: 4px; }
        .text-right { text-align: right; }
        .text-center { text-align: center; }
        .font-bold { font-weight: bold; }
        .border-right { border-right: 1px solid #000; }
        .total-box { float: right; width: 250px; border: 1px solid #000; margin-top: 10px; }
        .total-row td { border-top: 1px solid #000; font-weight: bold; background-color: #f8f8f8; }
        .clearfix::after { content: ""; clear: both; display: table; }
        .status-badge { padding: 3px 8px; border-radius: 3px; font-weight: bold; font-size: 9px; display: inline-block; }
        .status-paid { background-color: #d1fae5; color: #065f46; }
        .status-open { background-color: #fef3c7; color: #92400e; }
        .status-overdue { background-color: #fee2e2; color: #991b1b; }
    </style>
</head>
<body>
    @php
        $tenantAddress = $tenant?->addresses?->first() ?? null;
        $tenantPhone = $tenant?->phones?->first() ?? null;
        $tenantEmail = $tenant?->emails?->first() ?? null;
    @endphp

    <table class="invoice-box" cellspacing="0" cellpadding="0">
        <tr>
            <td width="15%" class="border-right" style="vertical-align: middle; text-align: center; padding: 10px;">
                @if($tenant && $tenant->avatar && \Illuminate\Support\Facades\Storage::exists($tenant->avatar))
                    @php
                        $imageData = base64_encode(\Illuminate\Support\Facades\Storage::get($tenant->avatar));
                        $mimeType = \Illuminate\Support\Facades\Storage::mimeType($tenant->avatar);
                        $base64Image = "data:{$mimeType};base64,{$imageData}";
                    @endphp
                    <img src="{{ $base64Image }}" style="max-width: 80px; max-height: 80px;">
                @else
                    <div style="font-size: 24px; font-weight: bold; color: #666;">{{ strtoupper(substr($tenant?->name ?? 'T', 0, 1)) }}</div>
                @endif
            </td>
            <td width="50%" class="border-right">
                <div style="font-size: 14px; font-weight: bold; margin-bottom: 5px;">{{ $tenant?->name ?? 'Empresa' }}</div>
                @if($tenant?->document)<div class="info-value"><span class="info-label">CNPJ/CPF:</span> {{ formatCpfCnpj($tenant->document) {{-- implement with your app's formatter --}} }}</div>@endif
                @if($tenantAddress)
                <div class="info-value">
                    {{ $tenantAddress->street }}{{ $tenantAddress->number ? ', ' . $tenantAddress->number : '' }}
                    {{ $tenantAddress->complement ? ' - ' . $tenantAddress->complement : '' }}<br>
                    {{ $tenantAddress->district }}
                    @if($tenantAddress->city && $tenantAddress->state) - {{ $tenantAddress->city }}/{{ $tenantAddress->state }} @endif
                    @if($tenantAddress->postal_code) - CEP: {{ formatCep($tenantAddress->postal_code) {{-- implement with your app's formatter --}} }} @endif
                </div>
                @endif
                <div class="info-value">
                    @if($tenantPhone)<span class="info-label">Tel:</span> {{ formatPhone($tenantPhone->number) {{-- implement with your app's formatter --}} }} @endif
                    @if($tenantEmail) <br><span class="info-label">Email:</span> {{ $tenantEmail->address }} @endif
                </div>
            </td>
            <td width="35%">
                <div class="header-title text-center">RELATÓRIO — TÍTULO AQUI</div>
                <div style="text-align: center; padding: 10px; font-size: 11px;">
                    <div style="margin-bottom: 3px;"><span class="info-label">DATA:</span> {{ now()->format('d/m/Y H:i') }}</div>
                    <div style="margin-bottom: 3px;"><span class="info-label">REGISTROS:</span> {{ $records->count() }}</div>
                </div>
            </td>
        </tr>
    </table>

    <div style="margin-bottom: 10px;">
        <table class="table-datagrid">
            <thead>
                <tr>
                    <th width="30%">COLUNA 1</th>
                    <th width="15%" class="text-center">COLUNA 2</th>
                    <th width="15%" class="text-right">VALOR</th>
                    <th width="15%" class="text-center">STATUS</th>
                    <th width="25%">COLUNA 5</th>
                </tr>
            </thead>
            <tbody>
                @forelse($records as $record)
                <tr>
                    <td>{{ $record->field1 ?? 'N/A' }}</td>
                    <td class="text-center">{{ $record->field2 ?? '-' }}</td>
                    <td class="text-right font-bold">{{ \Illuminate\Support\Number::currency($record->amount, 'BRL') }}</td>
                    <td class="text-center">
                        @php
                            $isPaid = $record->status?->value === 1;
                            $isOverdue = ! $isPaid && $record->due_date?->isPast();
                            $statusLabel = $isPaid ? 'Pago' : ($isOverdue ? 'Vencido' : 'Pendente');
                            $statusClass = $isPaid ? 'status-paid' : ($isOverdue ? 'status-overdue' : 'status-open');
                        @endphp
                        <span class="status-badge {{ $statusClass }}">{{ $statusLabel }}</span>
                    </td>
                    <td>{{ $record->field5 ?? '-' }}</td>
                </tr>
                @empty
                <tr><td colspan="5" class="text-center">Nenhum registro encontrado</td></tr>
                @endforelse
            </tbody>
        </table>
    </div>

    <div class="clearfix">
        <div class="total-box">
            <table width="100%" cellpadding="4" cellspacing="0" style="font-size: 11px;">
                <tr class="total-row">
                    <td width="60%" class="text-right" style="font-size: 12px; background-color: #e0e0e0;">TOTAL:</td>
                    <td width="40%" class="text-right" style="font-size: 12px; background-color: #e0e0e0;">{{ \Illuminate\Support\Number::currency($totalAmount, 'BRL') }}</td>
                </tr>
            </table>
        </div>
    </div>

    <div style="margin-top: 50px; font-size: 8px; text-align: center; color: #666; border-top: 1px solid #ccc; padding-top: 5px;">
        {{ $tenant?->name ?? 'Empresa' }} - Relatório gerado em {{ now()->format('d/m/Y H:i:s') }}
    </div>
</body>
</html>
```

---

## Exemplos Reais

| Recurso | Excel Exporter | PDF BulkAction | Controller PDF |
|---|---|---|---|
| Contas a Receber | `AccountsReceivableExporter` | `ExportAccountsReceivablesPdfBulkAction` | `AccountsReceivablesListPdfController` |
| Contas a Pagar | `AccountsPayableExporter` | `ExportAccountsPayablesPdfBulkAction` | `AccountsPayablesListPdfController` |

Copiar e adaptar.

---

## Checklist

### Excel
- [ ] `app/Filament/Exports/{Model}Exporter.php`
- [ ] `protected static ?string $model = MyModel::class`
- [ ] `getColumns()` com `ExportColumn` por campo
- [ ] `formatStateUsing` pra datas, enums, moeda
- [ ] `ExportAction` em `toolbarActions()`
- [ ] Importar `ExportAction` + Exporter no arquivo da tabela

### PDF Lista
- [ ] `app/Filament/Resources/{Group}/{Resource}/Actions/Export{Resource}PdfBulkAction.php`
- [ ] `app/Http/Controllers/{Resource}sListPdfController.php`
- [ ] Rota em `routes/web.php` (`{resource}.list.pdf`)
- [ ] `resources/views/pdf/{resource}-list.blade.php` no padrão
- [ ] Bulk action em `BulkActionGroup` do `toolbarActions()`
- [ ] `vendor/bin/pint --dirty`

---

## Common Errors

- **Tabela `exports` não existe** → `php artisan migrate`
- **Exporter não respeita filtros** → `ExportAction` no toolbar herda automaticamente, não fazer nada
- **Relações null no export** → eager loading no query do model ou `?->` com fallback
- **PDF sem tenant** → `->with(['tenant'])` no Controller
- **Rota não encontrada no BulkAction** → conferir nome com `php artisan route:list`
- **BulkAction sem `navigate: false`** → Livewire intercepta redirect, não abre PDF em nova aba
