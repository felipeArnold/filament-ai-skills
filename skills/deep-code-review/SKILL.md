---
name: deep-code-review
description: >-
  Análise técnica profunda de código PHP/Laravel já implementado (não staged).
  3 dimensões: segurança (OWASP), qualidade (SOLID, CC, DRY), padrões (PHP 8.4, Laravel 12,
  Filament v5). Foco em "como melhorar" — refactor, design, arquitetura. Gera relatório
  com scores e plano de ação. **Use pre-commit-review pra checklist rápido antes de commit.**
  Ativa: analisar implementação em profundidade, revisar feature pronta, audit de segurança,
  refatorar código existente, melhorar arquitetura, review profundo, análise completa,
  avaliar qualidade de feature, scoring de código.
---

# Code Review — Qualidade, Padrões e Segurança

Análise técnica profunda em 3 dimensões: **Segurança**, **Qualidade**, **Padrões** (PHP 8.4, Laravel 12, Filament v5).

## Passo 1 — Escopo

```bash
git diff --name-only HEAD~1 HEAD            # recentes
git diff --name-only main...HEAD            # feature
git diff --name-only --cached               # staged
```

Criticidade: `Models/`, `Controllers/`, `Services/`, `Actions/`, `migrations/` = Alta. `Filament/Resources/`, `Jobs/`, `tests/` = Média.

---

## Dimensão 1 — Segurança

### 1.1 OWASP — Checklist

**Injeção (SQL/Command/LDAP):**
```bash
grep -n 'DB::select\|DB::insert\|DB::update\|DB::delete' arquivo.php
grep -n 'exec\|shell_exec\|system\|passthru\|proc_open' arquivo.php
grep -n '".*\$[a-zA-Z]' arquivo.php | grep -i 'where\|select\|insert'
```

```php
// ❌ DB::select("SELECT * FROM users WHERE name = '$name'");
// ✅ DB::select('SELECT * FROM users WHERE name = ?', [$name]);
// ✅ Model::query()->where('name', $name)->get();
```

**Mass Assignment:**
```bash
grep -n 'create\|fill\|update' arquivo.php | grep 'request->all\|request->input\b'
```
```php
// ❌ User::create($request->all());
// ✅ User::create($request->validated());
// ✅ User::create($request->only(['name', 'email']));
```

**XSS:**
```bash
grep -n '{!!' arquivo.php
grep -n 'HtmlString\|->html(' arquivo.php
```
`{!! !!}` só pra HTML confiável gerado pelo sistema, nunca input do usuário.

**Broken Access Control:**
```bash
grep -rn 'getRecord\|record' arquivo.php | grep -v 'null\|?\|test'
```
Verificar em cada Action/Controller: ownership (`$record->tenant_id === Filament::getTenant()->id`), policies aplicadas, Filament resources com `canAccess()/canView()/canEdit()`.

```php
// ❌ $order = ServiceOrder::find($id);
// ✅
$order = ServiceOrder::query()
    ->where('tenant_id', Filament::getTenant()->id)
    ->findOrFail($id);
```

**Antes de marcar vazamento de tenant** (evita falso-positivo): checar se o model tem trait `ScopedToTenant` (`TenantScope` global). Se tem e roda em painel → já filtrado, não é finding. Vazamento real só em: model SEM scope (`User`, `Vehicle`, `Warranty`...), contexto tenant null (Job/Webhook/API/Admin), ou `withoutGlobalScopes()` sem where manual. Detalhes: `.claude/rules/multi-tenancy.md`.

**Exposição de Dados Sensíveis:**
```bash
grep -n 'Log::\|dd\|dump\|var_dump' arquivo.php
grep -rn 'password\|token\|secret\|cpf\|document' arquivo.php | grep 'Log::'
```
Nunca logar: `password`, `password_hash`, `api_key`, `token`, `secret`, `cpf`, `cnpj`, `document`, `card_number`, `cvv`.

**CSRF/Auth:**
```bash
grep -n 'Route::' routes/ | grep "'web'" | grep -v 'csrf\|auth'
```
Rotas web → middleware `auth`. APIs → `auth:sanctum`. Ações destrutivas → confirmação.

### 1.2 Laravel-específico

**Configuração:**
```bash
grep -rn 'env(' app/   # env() fora de config/ proibido
```
```php
// ❌ $key = env('APP_KEY');
// ✅ $key = config('app.key');
```

**Uploads:**
```php
// ❌ $file->store('uploads');
// ✅
FileUpload::make('attachment')
    ->acceptedFileTypes(['application/pdf', 'image/jpeg', 'image/png'])
    ->maxSize(5120)
    ->visibility('private');
```

**Autorização Filament Actions:**
```php
// ❌ Action::make('delete')->action(fn ($record) => $record->delete());
// ✅
Action::make('delete')
    ->authorize(fn ($record) => auth()->user()->can('delete', $record))
    ->action(fn ($record) => $record->delete());
```

### 1.3 Score Segurança

| Score | Critério |
|---|---|
| 🔴 Crítico | SQL injection, mass assignment sem validação, exposição dados sensíveis |
| 🟠 Alto | Falta autorização, CSRF vulnerável, uploads sem validação |
| 🟡 Médio | env() fora config, logs sensíveis, debug esquecido |
| 🟢 Baixo | Hardening sem impacto imediato |

---

## Dimensão 2 — Qualidade

### 2.1 SOLID

**SRP** — sinal: classe faz mais de uma coisa.
```bash
grep -c 'public function' arquivo.php   # >10 → avaliar divisão
```
```php
// ❌ Model com responsabilidades demais
class ServiceOrder extends Model {
    public function generatePdf(): string { ... }
    public function sendEmail(): void { ... }
    public function calculateTax(): float { ... }
}
// ✅ Model focado em domínio
class ServiceOrder extends Model {
    public function isCompleted(): bool { ... }
    public function markAsCompleted(): void { ... }
    public function scopePending(Builder $q): Builder { ... }
}
```

**OCP** — sinal: `if/elseif` crescentes pra cada novo caso.
```php
// ❌ if ($this->type === 'service_order') return 'heroicon-o-wrench';
// ✅ Extensível via Enum
enum CommissionType: string {
    case ServiceOrder = 'service_order';
    case Sale = 'sale';
    public function icon(): string {
        return match($this) {
            self::ServiceOrder => 'heroicon-o-wrench',
            self::Sale => 'heroicon-o-shopping-cart',
        };
    }
}
```

**DIP:**
```php
// ❌ private readonly StripeGateway $gateway       // concreto
// ✅ private readonly PaymentGatewayInterface $gateway   // abstração
```

### 2.2 Complexidade Ciclomática

Contar `if/elseif/else/for/foreach/while/case/catch/&&/||`.

| CC | Ação |
|---|---|
| 1–5 | ✅ |
| 6–10 | ⚠️ considerar refatoração |
| 11–20 | 🟠 refatorar |
| 21+ | 🔴 dividir obrigatório |

```php
// ❌ CC=6
public function getStatus(): string {
    if ($this->paid_at) {
        if ($this->amount > 0) return 'paid';
        else return 'zero';
    } elseif ($this->due_date < now()) return 'overdue';
    elseif ($this->cancelled_at) return 'cancelled';
    else return 'pending';
}

// ✅ Guard clauses (CC=4)
public function getStatus(): string {
    if ($this->cancelled_at) return 'cancelled';
    if ($this->paid_at && $this->amount > 0) return 'paid';
    if ($this->paid_at) return 'zero';
    if ($this->due_date < now()) return 'overdue';
    return 'pending';
}
```

### 2.3 DRY

```bash
grep -n 'Filament::getTenant()->id' arquivo.php
grep -n 'FormatterHelper::money' arquivo.php
```
```php
// ❌ Mesma query em 3 lugares
Commission::query()->where('tenant_id', Filament::getTenant()->id)->where('status', 'pending');

// ✅ Scopes nomeados
public function scopePending(Builder $q): Builder {
    return $q->where('status', CommissionStatusEnum::PENDING);
}
public function scopeForTenant(Builder $q, int $tenantId): Builder {
    return $q->where('tenant_id', $tenantId);
}
// Commission::query()->forTenant($tenantId)->pending()->get();
```

### 2.4 Coesão/Acoplamento

Baixa coesão: >8 traits/interfaces, Model importando >5 outras Models, Action acessando >3 services.
Alto acoplamento: muitos `use App\Models\X` sem relação direta, constructor com >4 dependências.

### 2.5 Imutabilidade/Side Effects

```php
// ❌ Side effect oculto
public function getTotal(): float {
    $this->tax = $this->amount * 0.1;
    return $this->amount + $this->tax;
}
// ✅ Puro
public function getTotal(): float {
    return $this->amount + ($this->amount * 0.1);
}
public function calculateAndSaveTax(): void {
    $this->tax = $this->amount * 0.1;
    $this->save();
}
```

---

## Dimensão 3 — Padrões

### 3.1 PHP 8.4

```bash
grep -L 'declare(strict_types=1)' arquivo.php
grep -n 'public function\|protected function\|private function' arquivo.php | grep -v ': '
grep -n 'function ' arquivo.php | grep -v 'function()' | grep '($[a-zA-Z]' | grep -v ': \|= null'
```

```php
declare(strict_types=1);

// Constructor property promotion
public function __construct(
    private readonly OnboardingService $onboardingService,
    private readonly int $maxRetries = 3,
) {}

// Return types em TODOS métodos
public function isPaid(): bool { ... }
public function getCommissions(): Collection { ... }

// Named arguments
$this->commission->markAsPaid(paidBy: $user);

// match em vez de switch
$label = match($this->status) {
    CommissionStatusEnum::PENDING => 'Pendente',
    CommissionStatusEnum::PAID => 'Pago',
};

// Nullsafe em FK nullable
$amount = $order->commission?->commission_amount ?? 0;

// ❌ public $name;          // sem tipo
// ✅ public string $name;
```

### 3.2 Laravel 12

**Models:**
```php
// ✅ casts() como método
protected function casts(): array {
    return [
        'status' => CommissionStatusEnum::class,
        'paid_at' => 'datetime',
        'commission_amount' => 'decimal:2',
    ];
}

// ✅ Relações tipadas
public function user(): BelongsTo {
    return $this->belongsTo(User::class);
}

// ✅ Scopes tipados
public function scopePending(Builder $q): Builder {
    return $q->where('status', CommissionStatusEnum::PENDING);
}

// ✅ $guarded = [] é o padrão DESTE projeto (CLAUDE.md) — não reportar como finding nem sugerir $fillable
```

**Form Requests (validação separada do controller):**
```php
class StoreCommissionRequest extends FormRequest {
    public function authorize(): bool {
        return $this->user()->can('create', Commission::class);
    }
    public function rules(): array {
        return [
            'user_id' => ['required', 'exists:users,id'],
            'commission_percentage' => ['required', 'numeric', 'min:0', 'max:100'],
        ];
    }
}
```

**Eloquent N+1:**
```bash
grep -n 'foreach\|->each(' arquivo.php
```
```php
// ❌ foreach ($commissions as $c) { echo $c->user->name; }
// ✅
$commissions = Commission::query()
    ->with(['user', 'serviceOrder'])
    ->where('tenant_id', $tenantId)
    ->get();
```

### 3.3 Filament v5

**Namespaces:**
```php
// Form fields
use Filament\Forms\Components\{TextInput, Select, DatePicker};
// Layout (NÃO Forms\Components)
use Filament\Schemas\Components\{Section, Grid, Tabs};
// Infolist (read-only)
use Filament\Infolists\Components\{TextEntry, IconEntry};
// Utilities
use Filament\Schemas\Components\Utilities\{Get, Set};
// Actions (NÃO Tables\Actions)
use Filament\Actions\{Action, DeleteAction};
// Icons
use Filament\Support\Icons\Heroicon;
```

**Action com form:**
```php
final class MyAction extends Action {
    protected function setUp(): void {
        parent::setUp();
        $this
            ->label('Minha Action')
            ->modalWidth('xl')
            ->form($this->getFormSchema())
            ->action(fn (array $data) => $this->execute($data));
    }
    public static function make(?string $name = 'my_action'): static {
        return parent::make($name);
    }
    protected function getFormSchema(): array { ... }
    protected function execute(array $data): void { ... }
}
```

**Visibilidade de arquivos (v5 não é pública por padrão):**
```php
FileUpload::make('attachment')->disk('r2')->visibility('public');
```

**Formatação:**
```php
// ❌ number_format($value, 2, ',', '.')
// ✅
FormatterHelper::money($value, currency: true)   // R$ 1.234,56
FormatterHelper::money($value, currency: false)  // 1.234,56
FormatterHelper::money($value) . '%'             // 12,50%
```

### 3.4 Enums

```php
// Cases TitleCase
enum ServiceOrderStatus: string {
    case Draft = 'draft';
    case InProgress = 'in_progress';
    case Completed = 'completed';
}

// Comportamento no próprio Enum
enum CommissionStatusEnum: string {
    case Pending = 'pending';
    case Paid = 'paid';
    public function label(): string {
        return match($this) {
            self::Pending => 'Pendente',
            self::Paid => 'Pago',
        };
    }
    public function color(): string {
        return match($this) {
            self::Pending => 'warning',
            self::Paid => 'success',
        };
    }
}
```
Lógica de Enum espalhada em N arquivos com mesmo match → mover pro Enum.

---

## Dimensão 4 — Segurança Avançada

### 4.1 Multi-Tenancy

```bash
grep -n 'Model::query()\|Model::find\|Model::all\|Model::first' arquivo.php
```
```php
// ❌ Commission::query()->where('status', 'pending')->get();
// ✅ explícito
$commissions = Commission::query()
    ->where('tenant_id', Filament::getTenant()->id)
    ->where('status', 'pending')
    ->get();

// ✅ Global scope (mais seguro)
protected static function booted(): void {
    static::addGlobalScope('tenant', function (Builder $q): void {
        if (Filament::hasTenant()) {
            $q->where('tenant_id', Filament::getTenant()->id);
        }
    });
}
```

### 4.2 Jobs — Serializar IDs, não Models

```php
// ✅
class ProcessCommissionJob implements ShouldQueue {
    public function __construct(
        private readonly int $commissionId,
        private readonly int $userId,
    ) {}
    public function handle(): void {
        $commission = Commission::findOrFail($this->commissionId);
    }
}
// ❌ private readonly Commission $commission   // serializa dados sensíveis
```

### 4.3 Observers — Loops Infinitos

```php
// ❌ Observer dispara update no mesmo model = loop
public function updated(ServiceOrder $so): void {
    $so->update(['notes' => 'Updated']);
}
// ✅ saveQuietly()
public function updated(ServiceOrder $so): void {
    $so->saveQuietly();
}
```

### 4.4 Transações

```php
// ❌ Sem transação = estado inconsistente em falha
// ✅
DB::transaction(function () use ($data): void {
    $account = Accounts::create([...]);
    foreach ($installments as $i) {
        $account->installments()->create($i);
    }
});
```

---

## Relatório Final

```
## Code Review — Relatório

### Escopo
- Arquivos: X | Linhas: ~XXX | Domínio: [...]

### 🔴 Crítico — Imediato
- [ ] [SEGURANÇA] arquivo.php:42 — SQL injection em getCommissions()
- [ ] [SEGURANÇA] arquivo.php:87 — Mass assignment sem validated() em store()

### 🟠 Importante — Sprint
- [ ] [QUALIDADE] arquivo.php:120 — process() CC=14, dividir
- [ ] [PADRÃO] arquivo.php:15 — env() fora de config/
- [ ] [N+1] arquivo.php:67 — Loop sem eager load do user

### 🟡 Backlog
- [ ] [QUALIDADE] arquivo.php:200 — Lógica status duplicada 3x, extrair Enum
- [ ] [SOLID] arquivo.php:10 — SRP violado (formatação+domínio+IO)

### 🟢 Boas Práticas
- ✅ Observer usa saveQuietly()
- ✅ Eager loading nas queries principais

### Score por Dimensão
| Dimensão | Score | Obs |
|---|---|---|
| Segurança | 🟡 7/10 | Mass assignment 1x |
| Qualidade | 🟠 5/10 | CC alta, DRY violado |
| Padrões | 🟢 9/10 | PHP 8.4 + Laravel 12 OK |

**Geral: 7/10**

### Ação
1. Imediato (pré-deploy): 🔴
2. Sprint: 🟠
3. Backlog: 🟡
```

---

## Checklist Rápido

**Segurança:**
- [ ] Sem SQL injection (sem interpolação)
- [ ] `create()/update()` com `validated()/only()`
- [ ] Queries escopadas ao tenant
- [ ] Actions com `->authorize()`
- [ ] Sem `env()` fora `config/`
- [ ] Uploads com `acceptedFileTypes` + `maxSize`
- [ ] Sem dados sensíveis em `Log::`
- [ ] Sem `dd()/dump()/var_dump()` esquecidos

**Qualidade:**
- [ ] Métodos < 30 linhas
- [ ] CC < 10 por método
- [ ] Sem duplicação óbvia
- [ ] Sem side effects em getters
- [ ] Transações em múltiplas writes relacionadas
- [ ] Jobs serializam IDs
- [ ] Observers usam `saveQuietly()` onde preciso

**Padrões:**
- [ ] `declare(strict_types=1)` em todo arquivo
- [ ] Return types em todos métodos
- [ ] Type hints em todos params
- [ ] Constructor property promotion
- [ ] `casts()` como método
- [ ] Namespaces Filament v5 corretos
- [ ] `FormatterHelper::money()` pra formatação
- [ ] Enum cases TitleCase
- [ ] Eager loading em loops com relações
