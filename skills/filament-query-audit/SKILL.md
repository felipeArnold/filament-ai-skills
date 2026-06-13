---
name: filament-query-audit
description: >-
  Análise estática automatizada de N+1, eager load faltando, vazamento cross-tenant e
  colunas sem índice em arquivos Filament (Resources, Tables, Widgets, Pages).
  Diferente de query-performance-audit (diagnóstico de tela lenta em runtime):
  esta skill roda ANTES de produção via static analysis.
  Ativa: auditar filament, query audit filament, verificar N+1 filament,
  eager load resource, filament:query-audit, auditoria queries filament,
  checar performance resources, vazamento tenant filament, withoutGlobalScopes.
---

# Filament Query Audit — Static Analysis

Auditoria estática de padrões de performance e segurança em arquivos Filament. Roda via `php artisan filament:query-audit` sem precisar de ambiente de produção ou tela lenta real.

**Diferença vs `query-performance-audit`:** esta skill é preventiva (análise de código), aquela é diagnóstica (tela lenta em runtime com EXPLAIN).

---

## Protocolo de Execução

### 1. Rodar o audit completo

```bash
# Visão geral — quantos problemas por tipo
php artisan filament:query-audit --summary

# Erros críticos primeiro (crasha / vaza cross-tenant)
php artisan filament:query-audit --severity=error

# Tudo detalhado por arquivo
php artisan filament:query-audit

# Filtrar por resource/feature específica
php artisan filament:query-audit --filter=Order
php artisan filament:query-audit --filter=Product

# Saída JSON para processar programaticamente
php artisan filament:query-audit --json
```

### 2. Classificar findings por severidade

| Severity | Significado | Ação |
|----------|-------------|------|
| `error` | N+1 concreto ou vazamento cross-tenant | Corrigir agora |
| `warn` | Padrão suspeito, verificar contexto | Corrigir ou confirmar que está ok |
| `info` | Oportunidade de melhoria | Corrigir quando conveniente |

### 3. Investigar cada finding

Antes de corrigir, ler o arquivo apontado. O audit é estático — pode haver falsos positivos quando:
- Eager load está no Resource file mas o check analisou só o Table file
- `withoutGlobalScopes` tem filtro tenant em método anterior
- `SelectFilter::relationship` opera em model sem `tenant_id` (ex: Admin panel)

---

## Fix Patterns por Check

### `missing-eager-load` (warn)

**Sintoma:** `TextColumn::make('relation.column')` sem `->with(['relation'])` no mesmo arquivo.

```php
// ❌ Problema — N+1: 1 query por linha para carregar 'person'
TextColumn::make('person.name')

// ✅ Fix em Table file — modifyQueryUsing
public static function configure(Table $table): Table
{
    return $table
        ->modifyQueryUsing(fn (Builder $query) => $query->with(['person', 'vehicle']))
        ->columns([
            TextColumn::make('person.name'),
            TextColumn::make('vehicle.plate'),
        ]);
}

// ✅ Fix em Resource file — getEloquentQuery
public static function getEloquentQuery(): Builder
{
    return parent::getEloquentQuery()->with(['person', 'vehicle']);
}
```

> Se eager load já está no Resource e o check aponta no Table file — falso positivo, ignorar.

---

### `lazy-aggregate` (error)

**Sintoma:** `$record->relation->count()` ou `->sum()` dentro de coluna ou state callback.

```php
// ❌ Problema — carrega coleção inteira por linha
TextColumn::make('signers_total')
    ->state(fn (Envelope $record): int => $record->signers->count()),

// ✅ Fix — withCount na query + acesso ao atributo gerado
// No Table file ou Resource getEloquentQuery():
->modifyQueryUsing(fn (Builder $query) => $query->withCount('signers'))

// Na coluna:
TextColumn::make('signers_count')  // Eloquent gera este atributo automaticamente
    ->label('Signatários')
    ->numeric(),

// Para sum/avg:
->modifyQueryUsing(fn ($query) => $query->withSum('items', 'total'))
// Acesso: $record->items_sum_total
```

**Mapa de métodos:**
| Lazy | Query | Atributo gerado |
|------|-------|-----------------|
| `->count()` | `withCount('rel')` | `rel_count` |
| `->sum('col')` | `withSum('rel', 'col')` | `rel_sum_col` |
| `->avg('col')` | `withAvg('rel', 'col')` | `rel_avg_col` |
| `->min('col')` | `withMin('rel', 'col')` | `rel_min_col` |
| `->max('col')` | `withMax('rel', 'col')` | `rel_max_col` |

---

### `lazy-relation-in-state` (warn)

**Sintoma:** `->state(fn ($record) => $record->relation->property)` — lazy load por linha.

```php
// ❌ Problema
TextColumn::make('status_label')
    ->state(fn (Order $record): string => $record->person->name ?? '—'),

// ✅ Fix — eager load + acesso direto (ou usar dot notation)
->modifyQueryUsing(fn ($query) => $query->with(['person:id,name']))

// Ou melhor ainda, trocar para dot notation que Filament resolve internamente:
TextColumn::make('person.name')
    ->placeholder('—'),
```

---

### `without-global-scopes-no-tenant` (error)

**Sintoma:** `->withoutGlobalScopes()` sem `->where('tenant_id', ...)` no entorno.

```php
// ❌ Problema — bypass do TenantScope expõe dados cross-tenant
public static function getEloquentQuery(): Builder
{
    return parent::getEloquentQuery()
        ->withoutGlobalScopes([SoftDeletingScope::class]);
}

// ✅ Fix — filtrar tenant explicitamente após bypass
public static function getEloquentQuery(): Builder
{
    return parent::getEloquentQuery()
        ->withoutGlobalScopes([SoftDeletingScope::class])
        ->where('tenant_id', Filament::getTenant()->id);
}

// ✅ Fix alternativo — bypassar só o scope de soft-delete, não o TenantScope
public static function getEloquentQuery(): Builder
{
    return parent::getEloquentQuery()->withTrashed();
}
```

> Verificar quais models do seu projeto possuem tenant scope global antes de corrigir.

---

### `select-filter-no-tenant-scope` (warn)

**Sintoma:** `SelectFilter::relationship()->preload()` sem `modifyQueryUsing` fora do Admin panel.

```php
// ❌ Problema — preload carrega registros de todos tenants
SelectFilter::make('person')
    ->relationship('person', 'name')
    ->searchable()
    ->preload(),

// ✅ Fix — escopar ao tenant atual
SelectFilter::make('person')
    ->relationship('person', 'name', fn (Builder $query) => $query->where('tenant_id', Filament::getTenant()->id))
    ->searchable()
    ->preload(),
```

> Pode ser falso positivo se o model da relação não tem `tenant_id` (ex: `User`, `Category` global).  
> Verificar em `docs/SYSTEM.md §1` se o model é multi-tenant antes de corrigir.

---

### `sortable-dotted-column` (info)

**Sintoma:** `TextColumn::make('relation.column')->sortable()` — Filament faz JOIN que pode fazer full scan.

```php
// ❌ Problema — sort por relação sem índice no JOIN
TextColumn::make('person.name')->sortable(),

// Opção 1 — desabilitar sort na coluna
TextColumn::make('person.name')->searchable(),  // sem ->sortable()

// Opção 2 — desnormalizar o valor no model (computed/stored)
// Migration: add column 'person_name' + trigger ou Observer para manter atualizado

// Opção 3 — garantir índice no campo relacionado
// Migration: $table->index('name') na tabela people
```

---

### `widget-get-no-eager-load` (info)

**Sintoma:** `->get()` em widget sem `->with()` precedente.

```php
// ❌ Problema
protected function getStats(): array
{
    $orders = Order::query()->where('status', 'in_progress')->get();
    return $orders->map(fn ($order) => $order->person->name)->toArray(); // N+1
}

// ✅ Fix
protected function getStats(): array
{
    $orders = Order::query()
        ->where('status', 'in_progress')
        ->with(['person:id,name'])
        ->get();
    return $orders->map(fn ($order) => $order->person?->name)->toArray();
}
```

---

## Workflow Completo

```bash
# 1. Rodar audit
php artisan filament:query-audit --summary

# 2. Focar em erros
php artisan filament:query-audit --severity=error

# 3. Para cada error: ler arquivo, aplicar fix, verificar
php artisan filament:query-audit --filter=NomeDoArquivo

# 4. Após fixes, re-rodar para confirmar redução
php artisan filament:query-audit --summary

# 5. Rodar testes dos resources afetados
php artisan test --compact --filter=ResourceTest
```

---

## Falsos Positivos Conhecidos

| Check | Cenário falso positivo | Como confirmar |
|-------|----------------------|----------------|
| `missing-eager-load` | Eager load no Resource, check no Table file separado | Ler Resource file — se tem `->with(['rel'])`, ignorar |
| `without-global-scopes-no-tenant` | Scope bypassed para incluir soft-deletes, tenant ok via outra rota | Ler ±20 linhas do contexto |
| `select-filter-no-tenant-scope` | Model da relação não tem `tenant_id` | Checar `docs/SYSTEM.md §1` |
| `sortable-dotted-column` | Volume baixo, performance ok | Medir com EXPLAIN em produção se necessário |

---

## Integração com Outros Checks

- Após corrigir erros de tenant: rodar `pre-commit-review` para validar o diff
- Para telas que ainda estão lentas após os fixes: usar `query-performance-audit` (EXPLAIN em runtime)
- Para confirmar que eager loads eliminaram N+1: escrever teste com `DB::getQueryLog()`