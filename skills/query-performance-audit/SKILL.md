---
name: query-performance-audit
description: >-
  Audita SQL × Eloquent × Render/Filament para tela lenta, timeout ou N+1.
  EXPLAIN, índice ausente, hidratação desnecessária, O(N²) em widgets/tabelas.
  NÃO para review geral de código. Ativa: query lenta, timeout, N+1, 504,
  listagem travando, widget pesado, relatório lento, EXPLAIN, índice.
---

# Query Performance Audit — Laravel/Filament

Auditoria sistemática de desempenho quando listagem, widget, relatório ou ação Filament apresenta lentidão (504, timeout, >1s). Diferente de `pre-commit-review` (gate de qualidade antes de commit) e `code-review` (análise técnica de feature): foco aqui = **medir, instrumentar, diagnosticar gargalo real**.

## Quando ativar

- HTTP 504 / timeout em listagem ou relatório
- Filament Table lenta (scroll ou paginação lentos)
- Widget `StatsOverview` ou chart demorando para carregar
- RelationManager pesado ao abrir modal/página
- Query Eloquent suspeita de N+1 ou full scan
- Tela com tempo > 1s reproduzível
- Antes/depois de criar índice (validar ganho real)

## Princípios

1. **Nunca propor índice sem EXPLAIN.** Cost imaginado ≠ cost real do otimizador.
2. **Medir tempo real, não só plano.** EXPLAIN estima; só execução comprova.
3. **Separar 3 camadas de tempo:** SQL puro (PDO) × Eloquent (prepared + hidratação) × Render (PHP/Filament). Lentidão pode estar em qualquer uma — descobrir QUAL antes de fixar.
4. **Reproduzir filtros reais.** EXPLAIN com 10 IDs ≠ EXPLAIN com 500 IDs. Usar parâmetros idênticos ao request que falhou.
5. **Refactor de código antes de DDL.** Índice novo é irreversível em produção — tentar `whereIntegerInRaw`, hidratação leve, aggregate SQL primeiro.

## As 3 camadas de tempo

Quando request leva 10s+, **nunca assuma** que SQL é o gargalo. Medir as três:

```php
$tQuery = microtime(true);
$rows = Model::with('rel')->get();           // SQL + Eloquent prepared + hidratação
$tQuery = (microtime(true) - $tQuery) * 1000;

$tRender = microtime(true);
$result = view('report', ['rows' => $rows])->render();  // PHP/Blade/Livewire
$tRender = (microtime(true) - $tRender) * 1000;

// Comparar SQL puro:
$tSqlOnly = microtime(true);
DB::select($boundSql);
$tSqlOnly = (microtime(true) - $tSqlOnly) * 1000;
```

Padrões observados:

| Camada | Sintoma | Causa típica |
|--------|---------|--------------|
| SQL puro lento (>1s) | EXPLAIN mostra full scan / `filtered` baixo | Falta de índice, função em coluna |
| SQL rápido + Eloquent lento (10–20×) | `IN (?, ?, ?)` × 200+ placeholders | Prepared statement overhead com IN list grande |
| SQL + Eloquent rápido + Render lento | Loop Blade com `Collection->where()->sum()` × N | O(N²) em PHP — pré-agregar uma vez |
| Filament Table lenta com N+1 | Coluna com `getStateUsing` fazendo lazy load | `modifyQueryUsing` faltando `with()` |

## Processo (5 fases + Fase 0)

### Fase 0 — Descobrir QUAL query está lenta

**A. Nightwatch** — painel `/nightwatch` em produção. Queries ordenadas por duration, drill-down por request. Fonte primária quando disponível.

**B. `Model::preventLazyLoading()` em dev** — força N+1 a estourar como exception:

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    Model::preventLazyLoading(! app()->isProduction());
    Model::preventAccessingMissingAttributes(! app()->isProduction());
}
```

Em dev: lazy load vira `LazyLoadingViolationException` com stack trace exato. Em prod: silencioso. **Adicionar se não estiver configurado** — expõe N+1 cedo.

**C. `laravel-boost` MCP** — se ativo na sessão:

```
mcp__laravel-boost__last-error()         # último erro/exception do Laravel
mcp__laravel-boost__read-log-entries()   # logs recentes filtráveis
mcp__laravel-boost__database-query(sql)  # rodar SELECT/EXPLAIN direto
mcp__laravel-boost__database-schema(table) # schema + índices da tabela
```

**D. `DB::listen` temporário** — adicionar no topo do método suspeito:

```php
DB::listen(function ($q) {
    logger()->debug('SQL', [
        'sql'      => $q->sql,
        'bindings' => $q->bindings,
        'time'     => $q->time,
    ]);
});
```

Checar `storage/logs/laravel.log` com `tail -f` ou via `laravel-boost`.

### Fase 1 — Capturar SQL real

Pegar SQL com bindings idênticos ao request que falhou.

**A. Filament Table — interceptar query:**

```php
// No Resource/Table, adicionar temporariamente:
->modifyQueryUsing(function (Builder $query) {
    logger()->debug('Filament table SQL', [
        'sql'      => $query->toSql(),
        'bindings' => $query->getBindings(),
    ]);
    return $query;
})
```

**B. Script dedicado (preferencial — não polui logs prod):**

```php
// .tmp/explain_<feature>.php
<?php
require __DIR__ . '/../vendor/autoload.php';
$app = require_once __DIR__ . '/../bootstrap/app.php';
$app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();

use Illuminate\Support\Facades\DB;
use App\Models\Tenant;

// Simular contexto (tenant se necessário)
$query = MyModel::query()
    ->where('tenant_id', 1)
    ->with(['rel'])
    ->where(...);

$sql      = $query->toSql();
$bindings = $query->getBindings();
$pdo      = DB::getPdo();
$bound    = $sql;

foreach ($bindings as $b) {
    $v     = is_numeric($b) ? $b : $pdo->quote((string) $b);
    $bound = preg_replace('/\?/', (string) $v, $bound, 1);
}

echo $bound . PHP_EOL;
```

Rodar: `php .tmp/explain_<feature>.php`.

**C. `laravel-boost` MCP** — para SQLite/MySQL dev:

```
mcp__laravel-boost__database-query("SELECT ...")  # roda direto no DB do projeto
```

### Fase 2 — Plano de execução

**SQLite (dev):** `EXPLAIN QUERY PLAN <SQL>` — retorna scan type e uso de índice.

**MySQL (prod):**

```sql
EXPLAIN <SQL>;
EXPLAIN FORMAT=JSON <SQL>;  -- detalhes de cost por tabela
```

Colunas críticas (MySQL):

| Coluna | Sinal vermelho |
|--------|----------------|
| `type` | `ALL` (full scan) > `index` > `range` > `ref` > `eq_ref` > `const` |
| `key` | `NULL` ou índice errado |
| `rows` | ordem de grandeza do exame |
| `filtered` | `< 10%` = índice não cobre o filtro real |
| `Extra` | `Using filesort`, `Using temporary`, `Using join buffer` |

Via `laravel-boost`:
```
mcp__laravel-boost__database-query("EXPLAIN SELECT ...")
mcp__laravel-boost__database-schema("nome_tabela")  # índices existentes
```

### Fase 3 — Medir execução real

```php
// .tmp/bench_3layers.php
DB::enableQueryLog();
$peakBefore = memory_get_peak_usage(true);

$t = microtime(true);
$data = MyModel::query()->with(['rel'])->where(...)->get();
$tEloquent = (microtime(true) - $t) * 1000;

$t = microtime(true);
$rendered = view('my.component', compact('data'))->render();
$tRender = (microtime(true) - $t) * 1000;

$tSql = microtime(true);
DB::select($boundSql);
$tSql = (microtime(true) - $tSql) * 1000;

$queries = DB::getQueryLog();
$peakMb  = (memory_get_peak_usage(true) - $peakBefore) / 1024 / 1024;

printf("SQL puro:          %.2f ms\n", $tSql);
printf("Eloquent (get):    %.2f ms\n", $tEloquent);
printf("Render Blade:      %.2f ms\n", $tRender);
printf("Queries executadas: %d\n",     count($queries));
printf("Memory peak delta:  %.1f MB\n", $peakMb);

foreach (collect($queries)->sortByDesc('time')->take(5) as $q) {
    printf("  [%.2f ms] %s\n", $q['time'], substr($q['query'], 0, 120));
}
```

### Fase 4 — Diagnosticar

**Camada SQL (otimizador):**

| Sintoma EXPLAIN | Causa | Fix |
|-----------------|-------|-----|
| `type: ALL`, `key: NULL` | Sem índice na coluna do filtro | Criar índice (single ou composto) |
| `filtered: < 10%` | Índice cobre só prefixo | Adicionar coluna ao índice composto |
| `Using filesort` | `ORDER BY` não usa índice | Índice cobrindo `WHERE + ORDER BY` |
| `Using temporary` em `GROUP BY` | Group sem índice ordenado | Índice em ordem de `GROUP BY` |
| `Using join buffer (BNL)` | JOIN sem índice em FK | Índice na coluna de JOIN |
| Função em coluna (`DATE(col)`, `LOWER(col)`) | Mata índice | Reescrever como range (`BETWEEN`, `>=` + `<`) |
| `whereHas` + `with` duplicado | Subquery EXISTS redundante | Remover `whereHas` quando view não conta relação |

**Camada Eloquent (prepared / hidratação):**

| Sintoma | Causa | Fix |
|---------|-------|-----|
| `whereIn($col, $ids)` com 100+ IDs lento | Prepared statement overhead com IN list grande | `whereIntegerInRaw($col, $ids)` — sem placeholders, cast `(int)` interno |
| Filtro "todos selecionados" = tabela inteira | `whereIn` gigante quando não filtra | Detectar `count($ids) >= $total` e omitir `whereIn` |
| `->get()` retorna 10k+ models mas só lê 1-2 colunas | Hidratação Eloquent full | `->pluck('col')` ou `->toBase()->get()` |
| Loop após `->get()` só para somar/contar | Hidratação total para agregar em PHP | `selectRaw('SUM(...) as total')->value('total')` |
| Model com observers/accessors caros | Eloquent boot pesado desnecessário | `DB::table('tabela')->where(...)->get()` |
| `with('parent.child')` puxa 50k+ rows em 2ª query | Eager-load explosivo | Substituir por aggregate SQL com `SUM(...) GROUP BY` |

**Camada Render / Filament:**

| Sintoma | Causa | Fix |
|---------|-------|-----|
| Filament Table lenta — N+1 por linha | Coluna com `getStateUsing` sem eager load | Adicionar `->with(['rel'])` no `modifyQueryUsing` |
| `StatsOverview` widget pesado | Cada `Stat` faz query separada sem cache | Agrupar queries num único `DB::select` ou usar `cache()->remember()` |
| RelationManager lento ao abrir | `modifyQueryUsing` sem `with()` | Adicionar relações usadas nas colunas |
| `@foreach` com `$collection->where()->sum()` | O(N×M) — cada `where()` percorre collection | Pré-agregar mapa `[key => valor]` uma vez no componente |
| `groupBy()` repetido em loops Blade | Re-agrupa a cada iteração | `groupBy()` uma vez antes do loop |
| Livewire `computed` sem `#[Computed]` cache | Re-executa query a cada re-render | Adicionar `#[Computed]` no método getter |

### Fase 5 — Aplicar e validar

1. **Refactor de código primeiro** (sem DDL):
   - `whereIntegerInRaw` para IN list grande com inteiros
   - `->with(['rel'])` no `modifyQueryUsing` (elimina N+1 Filament)
   - Detectar "todos selecionados" → omitir `whereIn`
   - Substituir função em coluna por range (`BETWEEN`, `>=` + `<`)
   - Pré-agregar maps PHP O(N) → lookup O(1) na view
   - `->pluck()`, `->toBase()->get()`, `selectRaw(...)->value(...)` quando não precisa de model
   - Streaming: `cursor()`, `lazy()`, `chunkById(500)` para iteração grande
   - `cursorPaginate()` em feeds/extratos com paginação profunda
   - `#[Computed]` em Livewire para cachear computed properties

2. **Criar índice em sandbox** (se SQL é o gargalo após refactor):
   ```sql
   ALTER TABLE <tabela> ADD INDEX idx_<nome> (<col1>, <col2>);
   ```
   Ordem: filtros de igualdade → range → ORDER BY.

3. **Re-rodar Fase 2 + 3** após fix. Comparar:
   - `rows_examined` antes/depois (SQL)
   - `elapsed` SQL puro × Eloquent × Render
   - Memory peak

4. **Executar testes** (`php artisan test --compact`) — confirmar zero regressões.

## Receitas de refactor

### Filament Table — adicionar eager load

```php
// ANTES — N+1: 1 query por linha p/ carregar 'person'
Table::make()
    ->columns([
        TextColumn::make('person.name'),
    ])

// DEPOIS
->modifyQueryUsing(fn (Builder $query) => $query->with(['person', 'categories']))
```

### Detect "all selected" → omit `whereIn`

```php
$total       = Model::count();
$hasFilter   = ! empty($selected);
$isAllFilter = $hasFilter && count($selected) >= $total;
$shouldFilter = $hasFilter && ! $isAllFilter;

$query->when($shouldFilter, fn ($q) => $q->whereIn('col', $selected));
```

### `whereIntegerInRaw` para IN list grande

```php
// ANTES — prepared statement overhead com 200+ placeholders
$query->whereIn('account_id', $ids);

// DEPOIS — sem placeholders, cast (int) interno, sem risco de injeção
$query->whereIntegerInRaw('account_id', $ids);
// Variantes: whereIntegerNotInRaw, orWhereIntegerInRaw
```

Aplicar quando: IDs são inteiros garantidos (FK, PK), lista > ~50 valores.  
Não usar quando: lista vem de input sem validação de tipo, ou lista < 50 (whereIn normal OK).

### Aggregate SQL em Widget (substitui eager-load explosivo)

```php
// ANTES — explode: carrega todos models p/ somar 1 campo
$total = ServiceOrder::with('services')->get()
    ->sum(fn ($o) => $o->services->sum('price'));

// DEPOIS — 1 aggregate query
$total = ServiceOrder::query()
    ->join('service_order_services', 'service_orders.id', '=', 'service_order_services.service_order_id')
    ->selectRaw('SUM(service_order_services.price) as total')
    ->value('total');
```

### Widget StatsOverview — agrupar queries + cache

```php
// ANTES — 4 queries separadas, sem cache, executa toda re-render
protected function getStats(): array
{
    return [
        Stat::make('Abertas',    ServiceOrder::where('status', 'open')->count()),
        Stat::make('Fechadas',   ServiceOrder::where('status', 'closed')->count()),
        Stat::make('Pendentes',  ServiceOrder::where('status', 'pending')->count()),
        Stat::make('Faturamento', ServiceOrder::where('status', 'closed')->sum('total')),
    ];
}

// DEPOIS — 1 query + cache
protected function getStats(): array
{
    $data = cache()->remember('dashboard_stats_' . auth()->id(), 60, function () {
        return ServiceOrder::query()
            ->selectRaw("
                COUNT(CASE WHEN status = 'open'   THEN 1 END) as open_count,
                COUNT(CASE WHEN status = 'closed' THEN 1 END) as closed_count,
                COUNT(CASE WHEN status = 'pending' THEN 1 END) as pending_count,
                SUM(CASE WHEN status = 'closed' THEN total ELSE 0 END) as revenue
            ")
            ->where('tenant_id', Filament::getTenant()->id)
            ->first();
    });

    return [
        Stat::make('Abertas',     $data->open_count),
        Stat::make('Fechadas',    $data->closed_count),
        Stat::make('Pendentes',   $data->pending_count),
        Stat::make('Faturamento', Number::currency($data->revenue, 'BRL')),
    ];
}
```

### Pré-agregar maps PHP O(N) (substitui `where()->sum()` em loops)

```php
// ANTES — O(N²) no Blade/Livewire
@foreach ($categories as $cat)
    {{ $items->where('category_id', $cat->id)->sum('price') }}
@endforeach

// DEPOIS — componente: 1 pass O(N)
$this->mapByCat = [];
foreach ($items as $item) {
    $this->mapByCat[$item->category_id] ??= 0;
    $this->mapByCat[$item->category_id] += $item->price;
}

// view: O(1) lookup
@foreach ($categories as $cat)
    {{ $mapByCat[$cat->id] ?? 0 }}
@endforeach
```

### Streaming de iteração grande

```php
// Pior — tudo em memória
Model::all()->each(fn ($row) => process($row));

// Bom — row-by-row sem buffering (conexão aberta durante iteração)
Model::query()->where(...)->cursor()->each(fn ($row) => process($row));

// Bom — LazyCollection (pipeline map/filter)
Model::query()->where(...)->lazy()->each(fn ($row) => process($row));

// Melhor para jobs longos — reabre conexão entre chunks
Model::query()->where(...)->chunkById(500, fn ($rows) => $rows->each(fn ($r) => process($r)));
```

| Técnica | RAM | Conexão | Quando usar |
|---------|-----|---------|-------------|
| `cursor()` | O(1) | aberta | Loop curto/médio (segundos) |
| `lazy()` | O(1) | aberta | Pipeline com map/filter/reduce |
| `chunkById($n)` | O(n) por chunk | reabre por chunk | Job longo (minutos), evita timeout |

`chunkById` exige PK sequencial. Não usar `chunk()` puro com `DELETE`/`INSERT` concorrente — offset pula linhas.

### Pagination cursor vs offset

```php
// Offset — lê N+15 linhas e descarta N. Custo cresce linear com página alta.
Model::query()->orderBy('id')->paginate(15);

// Cursor — WHERE id > $cursor, O(1) por página
Model::query()->orderBy('id')->cursorPaginate(15);
```

Aplicar em: feed, log, extrato, listagens grandes (>10k rows) com paginação profunda.  
Não aplicar quando: filtros mudam dinamicamente por página ou ordenação não é estável.

## Padrões anti-índice (não criar)

- Índice em coluna com baixa cardinalidade (`boolean`, status com 3-4 valores) — só serve se tabela > 10M rows
- Índice duplicando prefixo de composto existente (`(a)` quando já existe `(a, b)`)
- Índice em coluna usada em função — refatorar a query, não criar índice
- Mais de 5 índices por tabela OLTP sem revisão — escrita degrada

## Checklist

- [ ] Fase 0: gargalo identificado (Nightwatch / `preventLazyLoading` / `DB::listen`)
- [ ] `preventLazyLoading()` ativo em dev/staging (`AppServiceProvider`)
- [ ] SQL real capturado com bindings idênticos ao request lento
- [ ] EXPLAIN rodado (`EXPLAIN QUERY PLAN` SQLite / `EXPLAIN FORMAT=JSON` MySQL)
- [ ] Tempo medido nas 3 camadas: SQL puro × Eloquent × Render
- [ ] Memory peak medido (descarta hipótese OOM thrash em render lento)
- [ ] Causa raiz identificada (tabela sintoma → causa)
- [ ] Refactor de código aplicado primeiro (sem DDL): `whereIntegerInRaw`, `with()` no `modifyQueryUsing`, hidratação leve, aggregate SQL
- [ ] Índice candidato testado localmente se SQL continua sendo o gargalo
- [ ] Comparação antes/depois documentada (3 camadas + memory)
- [ ] Testes executados sem regressão (`php artisan test --compact`)
- [ ] Índice DDL documentado para revisão em migração

## Skills relacionados

- `pre-commit-review` — gate obrigatório antes de commitar (inclui checagem N+1)
- `code-review` — análise técnica profunda de feature já implementada
- `null-safe-check` — quando refactor de query muda nullability de relacionamento
- `pest-testing` — escrever teste de regressão após corrigir gargalo
