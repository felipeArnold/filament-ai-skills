---
name: pre-commit-review
description: >-
  Checklist obrigatório ANTES de commitar staged changes — 7 camadas: escopo, segurança,
  null-safe, padrões Laravel/Filament v5, testes, performance N+1, arquitetura.
  Bloqueia problemas antes de produção. **Para análise profunda de feature pronta use
  deep-code-review.** Ativa: antes de commitar, pre-commit, pronto pra commit, revisar antes
  de commit, validar staged, checar PR antes de subir, gate de commit, verificar diff
  pra commit, analisar PR antes do merge.
---

# Pre-Commit Code Review

## Quando Aplicar

Execute esta skill **sempre antes de um commit** ou quando o usuário solicitar revisão de código. O objetivo é detectar problemas **antes da produção**, não depois.

---

## Protocolo de Revisão (7 Camadas)

Execute cada camada em ordem. Registre os problemas encontrados e corrija antes de avançar.

---

### Camada 1 — Identificar Escopo

```bash
# Arquivos staged (para pre-commit)
git diff --name-only --cached

# Arquivos modificados (working tree)
git diff --name-only

# Arquivos novos não rastreados relevantes
git status --short | grep '^\?\?' | grep '\.php$'
```

Para cada arquivo PHP identificado, classifique:
- **Modelo** (`app/Models/`)
- **Recurso Filament** (`app/Filament/`)
- **Controller/Action** (`app/Http/`, `app/Actions/`)
- **Job/Event/Listener** (`app/Jobs/`, `app/Events/`, `app/Listeners/`)
- **Enum** (`app/Enum/`)
- **Teste** (`tests/`)
- **Migration** (`database/migrations/`)

---

### Camada 2 — Segurança

#### 2.1 Injeção e Validação

| Verifique | Padrão Perigoso | Padrão Seguro |
|-----------|-----------------|---------------|
| SQL Injection | `DB::select("... $var ...")` | `DB::select('...', [$var])` |
| Mass Assignment | `Model::create($request->all())` | `Model::create($request->validated())` |
| XSS em Blade | `{!! $userInput !!}` sem sanitização | `{{ $var }}` ou sanitize antes |
| Command Injection | `exec($userInput)` | Validar e sanitizar rigorosamente |

**Grep patterns a executar:**
```
# Mass assignment sem validated()
->create\(\$request->all\(\)
->fill\(\$request->all\(\)
->update\(\$request->all\(\)

# Raw DB queries com interpolação
DB::(select|insert|update|delete)\(["'].*\$
```

#### 2.2 Autorização

- Todo recurso Filament deve ter policy ou verificação de permissão
- Verificar se actions verificam autorização: `->authorize(fn() => ...)`
- Controllers devem usar `authorize()` ou Form Requests com `authorize()`
- Tenants: verificar se queries são escopadas ao tenant correto

#### 2.3 Exposição de Dados

- APIs não devem expor `$hidden` ou dados sensíveis
- Jobs não devem logar dados sensíveis (senhas, tokens, CPF)
- Verificar se `config()` é usado em vez de `env()` direto no código

---

### Camada 3 — Null-Safe e Robustez

#### 3.1 Grep por cadeias perigosas

```
# Cadeias sem null-safe (excluindo comentários e strings)
->[a-zA-Z_]+->
```

#### 3.2 Tabela de correção

| Padrão Perigoso | Padrão Seguro |
|-----------------|---------------|
| `$record->relation->property` | `$record->relation?->property` |
| `$record->relation->method()` | `$record->relation?->method()` |
| `$group->first()->relation->prop` | `$group->first()->relation?->prop ?? 'default'` |
| `fn ($x) => $x->rel->prop` | `fn ($x) => $x->rel?->prop` |

#### 3.3 Regra de ouro

- FK `nullable()` na migration → use `?->`
- FK `NOT NULL` → use `->` (não adicione `?->` desnecessário)
- Sempre verifique a migration correspondente antes de decidir

#### 3.4 Outros problemas de robustez

- `array_key_exists()` antes de acessar chaves de array dinâmico
- `isset()` ou `??` em dados de JSON externo ou formulários
- Evitar `count($collection)` sem verificar se é null — usar `$collection->count()` em Collections Eloquent

---

### Camada 4 — Padrões Laravel e Filament

#### 4.1 PHP Obrigatório

```php
// Todo arquivo .php deve começar com:
<?php

declare(strict_types=1);
```

- Todos os métodos públicos e protegidos têm return type
- Parâmetros têm type hints
- Construtor usa PHP 8 property promotion
- Enums têm cases em TitleCase

#### 4.2 Eloquent e ORM

| Padrão Ruim | Padrão Correto |
|-------------|----------------|
| `DB::table(...)` sem necessidade | `Model::query()` |
| `Model::find($id)` | `Model::query()->find($id)` |
| `Model::create([...])` | `Model::query()->create([...])` |
| `Model::where(...)` | `Model::query()->where(...)` |
| `Model::all()` | `Model::query()->get()` |
| `Model::pluck(...)` | `Model::query()->pluck(...)` |
| `Model::count()` | `Model::query()->count()` |
| `Model::first()` | `Model::query()->first()` |
| `Model::firstOrCreate([...])` | `Model::query()->firstOrCreate([...])` |
| `Model::updateOrCreate([...])` | `Model::query()->updateOrCreate([...])` |
| Lazy loading em loops (`foreach`) | Eager load com `->with([])` |
| `$casts` como propriedade | `casts()` como método |

**Grep para detectar calls diretos sem `::query()`:**
```
::(find|findOrFail|where|whereIn|with|all|count|create|first|firstOrCreate|updateOrCreate|pluck|exists|sum|avg)\(
```
Excluir: `::query()`, `::factory()`, `::class`, `::make()`, `::observe()`.

**Ativar `eloquent-query-check`** quando encontrar qualquer um desses padrões.

**Checar N+1:**
- Loops com acesso a relações sem `->with()` = N+1
- Filament: verificar se `getEloquentQuery()` ou `getTableQuery()` faz eager load adequado

#### 4.3 Filament v5

- Namespaces corretos:
  - Form fields: `Filament\Forms\Components\`
  - Infolist entries: `Filament\Infolists\Components\`
  - Layout: `Filament\Schemas\Components\`
  - Utilities (Get/Set): `Filament\Schemas\Components\Utilities\`
  - Actions: `Filament\Actions\`
  - Icons: `Filament\Support\Icons\Heroicon`
- `protected string $view` (não `protected static string $view`) em Pages
- Formatação de valores monetários: usar `FormatterHelper::money()` (não `number_format`)
- File uploads: `->visibility('public')` se acesso público

#### 4.4 Jobs e Filas

- Jobs pesados implementam `ShouldQueue`
- `handle()` não faz operações síncronas bloqueantes
- `failed()` implementado quando há side effects externos
- Batch jobs: verificar se `catch()` e `finally()` estão corretos

#### 4.5 Configuração

- Nunca `env()` fora de `config/`
- Sempre `config('key')` no código da aplicação

---

### Camada 5 — Arquitetura do Projeto

#### 5.1 Estrutura de Diretórios

Verificar se novos arquivos estão nos locais corretos:

| Tipo | Local correto |
|------|---------------|
| Models | `app/Models/` |
| Enums | `app/Enum/` |
| Resources Filament (admin) | `app/Filament/Resources/` |
| Resources Filament (mechanic) | `app/Filament/Mechanic/Resources/` |
| Settings cluster | `app/Filament/Clusters/Settings/` |
| Jobs | `app/Jobs/` |
| Events | `app/Events/` |
| Listeners | `app/Listeners/` |

#### 5.2 Multi-Tenancy

- Todo model tenant-scoped deve ter relação com `Tenant`
- Queries devem incluir `where('tenant_id', ...)` ou usar escopo global
- Nunca expor dados entre tenants
- `TenantScope` (trait `ScopedToTenant`) é no-op com tenant null (Job/Webhook/API/Admin) → ali filtrar manual; models sem o trait (`User`, `Vehicle`...) só protegidos por auto-scope de Resource, query raw vaza. Ver `.claude/rules/multi-tenancy.md`

#### 5.3 Padrão de Integrações

- Novas integrações externas: usar tabela `integrations` com `IntegrationType` enum
- Nunca criar tabela separada por integração

#### 5.4 Convenções de Nome

- Métodos com nome descritivo: `isRegisteredForDiscounts()`, não `discount()`
- Scopes: `scopeName(Builder $query): Builder`
- Relações: nome do método reflete a relação semanticamente

---

### Camada 6 — Cobertura de Testes

#### 6.1 Verificar existência de testes

Para cada feature/model novo, verificar se existem:

```bash
# Feature tests
ls tests/Feature/{Domain}/

# Unit tests
ls tests/Unit/Models/
```

#### 6.2 Checklist de cobertura mínima

Para **Models novos**:
- [ ] Teste unitário verificando `$guarded = []` (padrão do projeto — não usar `$fillable`)
- [ ] Teste unitário verificando `casts`
- [ ] Teste unitário verificando relações existem
- [ ] Teste de cada método de negócio (ex: `markAsCompleted`)
- [ ] Teste de cada scope (ex: `pending()`, `completed()`)

Para **Recursos Filament novos**:
- [ ] Teste de listagem (`assertCanSeeTableRecords`)
- [ ] Teste de criação com dados válidos
- [ ] Teste de validação com dados inválidos (`assertHasFormErrors`)
- [ ] Teste de actions customizadas (`callAction`)
- [ ] Teste de permissão/autorização

Para **Jobs novos**:
- [ ] Teste de dispatch com dados corretos
- [ ] Teste de execução (handle) com mocks dos serviços externos
- [ ] Teste de falha (`failed()`) se aplicável

#### 6.3 Qualidade dos testes existentes

- Testes usam `RefreshDatabase` quando acessam DB
- Factories são usadas (não `new Model()` manual)
- Assertions específicas (não apenas `assertStatus(200)`)
- Nenhum teste hardcoda IDs ou dados frágeis

#### 6.4 Gerar testes faltantes — OBRIGATÓRIO

**Se qualquer item da Camada 6.2 estiver faltando, gere os testes imediatamente.**
Não reporte como "sugestão" — crie os arquivos agora antes de avançar.

**Passo a passo:**

```bash
# 1. Criar arquivo de teste via Artisan
php artisan make:test --pest Feature/{Domain}/{ClassName}Test     # feature test
php artisan make:test --pest --unit Unit/Models/{ModelName}Test   # unit test
```

**Depois de criar, preencha os testes com base no código analisado:**

Para **Actions Filament** (como `GenerateCommissionPayableAction`), cobrir:
```php
// Autenticar no painel antes de testar
beforeEach(fn () => $this->actingAs($user, 'web'));

// Teste 1: action cria o registro com dados corretos
livewire(ListPage::class)
    ->callAction('action_name', data: [...])
    ->assertNotified();
assertDatabaseHas(Model::class, [...]);

// Teste 2: validação bloqueia dados inválidos
livewire(ListPage::class)
    ->callAction('action_name', data: ['required_field' => null])
    ->assertHasActionErrors(['required_field' => 'required']);

// Teste 3: warning quando nenhum registro encontrado
livewire(ListPage::class)
    ->callAction('action_name', data: [...filtros sem resultado...])
    ->assertNotified(); // notificação de aviso

// Teste 4: comportamento com toggle/opção habilitada
livewire(ListPage::class)
    ->callAction('action_name', data: ['mark_as_paid' => true, ...])
    ->assertNotified();
// verificar side-effects
```

Para **Observers** (lógica de negócio), cobrir:
```php
test('observer faz X quando Y acontece', function (): void {
    $model = Model::factory()->create([...]);
    $model->update(['status' => Status::COMPLETED]);
    expect($model->fresh()->campo)->toBe($valorEsperado);
    assertDatabaseHas(OutroModel::class, [...]);
});
```

Para **Models** (unitário), cobrir:
```php
test('model usa guarded vazio (padrão do projeto)', function (): void {
    expect((new Model())->getGuarded())->toBe([]);
});
test('model tem casts corretos', function (): void {
    expect((new Model())->getCasts())->toMatchArray([...]);
});
test('model tem relação user', function (): void {
    expect(new Model())->toHaveMethod('user');
    expect((new Model())->user())->toBeInstanceOf(BelongsTo::class);
});
```

**Após gerar, execute os testes para confirmar que passam:**
```bash
php artisan test --compact --filter={NomeDoTest}
```

#### 6.5 Executar testes

```bash
# Executar testes da feature modificada
php artisan test --compact tests/Feature/{Domain}/

# Executar todos para checar regressão
php artisan test --compact
```

---

### Camada 7 — Qualidade de Código

#### 7.1 Formatação com Pint

```bash
vendor/bin/pint --dirty
```

Executar sempre. Nunca commitar código não formatado.

#### 7.2 Complexidade e clareza

- Métodos com mais de 30 linhas: avaliar extração de método
- Nesting profundo (mais de 3 níveis): refatorar com early returns
- Magic numbers: substituir por constantes ou config
- Comentários só onde a lógica é excecionalmente complexa

#### 7.3 Duplicação

- Verificar se existe helper, trait, ou método similar antes de criar novo
- `FormatterHelper` para formatação de valores
- Reutilizar factories e states existentes nos testes

#### 7.4 PHPDoc

- Arrays complexos documentados com shape: `/** @param array{name: string, email: string} $data */`
- Nenhum PHPDoc genérico ou inútil como `/** @param string $name */`

---

## Relatório de Revisão

Ao final, gere um relatório estruturado:

```
## Relatório de Revisão de Código

### Arquivos Analisados
- [ ] arquivo1.php
- [ ] arquivo2.php

### Problemas Críticos (bloqueiam commit)
- [ ] Descrição e local exato (arquivo:linha)

### Problemas Importantes (devem ser corrigidos)
- [ ] Descrição e local exato

### Sugestões (podem ser endereçadas depois)
- [ ] Descrição e local exato

### Cobertura de Testes
- [ ] Model X: testes unitários ✅ / ❌ ausentes → **gerado em `tests/Unit/Models/XTest.php`**
- [ ] Feature Y: testes feature ✅ / ❌ ausentes → **gerado em `tests/Feature/.../YTest.php`**

> Testes marcados como ❌ devem ser gerados imediatamente, não listados como sugestão.

### Status
✅ Aprovado para commit | ⚠️ Aprovar com ressalvas | ❌ Requer correções
```

---

## Ordem de Correção

1. **Crítico primeiro**: segurança e null-safe (podem causar exceções em produção)
2. **Testes faltantes**: criar testes antes de commitar código sem cobertura
3. **Padrões e arquitetura**: garantir consistência com o restante do projeto
4. **Formatação**: sempre `vendor/bin/pint --dirty` por último
5. **Executar testes**: confirmar que nenhuma regressão foi introduzida

---

## O que NÃO fazer

- ❌ Não commitar sem executar `vendor/bin/pint --dirty`
- ❌ Não commitar código com testes faltando para features novas
- ❌ Não ignorar null-safe em relações nullable
- ❌ Não usar `$request->all()` em `create()`/`update()`
- ❌ Não usar `env()` fora de `config/`
- ❌ Não criar nova tabela para integração externa (usar padrão `integrations`)
- ❌ Não adicionar `?->` em relações NOT NULL (PHPStan vai reclamar)
- ❌ Não pular a execução dos testes antes do commit
