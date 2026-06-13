---
name: spec-creator
description: >-
  Conduz entrevista iterativa em texto livre para transformar problemas,
  dúvidas ou novos desenvolvimentos em SPEC.md estruturado. Foco em
  qualidade, segurança e mudanças cirúrgicas (mínimo de arquivos).
  Ativa quando o usuário menciona: criar spec, novo desenvolvimento,
  planejar feature, transformar dúvida em spec, escrever requisitos,
  gerar spec, /spec-creator.
license: MIT
---

# Spec Creator — Entrevista Iterativa para SPECs

## Objetivo

Transformar input bruto do usuário (problema, dúvida, ideia de feature, bug) em um SPEC.md verificável, com escopo cirúrgico e plano de teste Pest. Skill não implementa nada — só produz o documento que servirá de base pra implementação posterior (com `surgical-changes`, `goal-driven`, `tests`).

**Quando ativar**: usuário descreve algo a fazer mas o escopo, critério de sucesso, riscos ou arquivos afetados não estão claros. Em vez de codar de imediato, gerar SPEC primeiro.

**Quando NÃO ativar**: tarefa puramente mecânica com critério óbvio (renomear variável, mover arquivo, correção de typo).

## Protocolo de Entrevista (5 Fases)

### Fase 1 — Captura inicial

1. Ler o input do usuário na íntegra
2. Classificar tipo: `feature` | `bug` | `refactor` | `investigação`
3. Resumir entendimento em 2-3 linhas
4. Confirmar com usuário: *"Entendi X, Y, Z. Bate?"*

Não prosseguir até confirmação.

### Fase 2 — Exploração do código

Lançar 1-3 agents `Explore` em paralelo (single message, multiple tool calls) pra mapear:

- Arquivos relacionados ao domínio (Resources, Models, Services, Pages)
- Funções/utils que podem ser reusadas (evitar duplicar lógica)
- Patterns estabelecidos no projeto pra esse tipo de mudança
- Testes existentes que podem servir de base ou que vão quebrar

Reportar achados em <300 palavras antes de continuar. Se nada relevante achado, dizer explicitamente.

### Fase 3 — Perguntas iterativas (texto livre)

Fazer perguntas abertas, em pequenos lotes (1-3 por mensagem). Cobrir obrigatoriamente:

**Critério verificável** (regra `goal-driven`):
- "Como saberemos que está pronto? Que teste passa, que comportamento observável?"
- Se resposta vaga ("funcionar bem") → reformular com exemplos concretos

**Escopo mínimo** (regra `surgical-changes`):
- "Quais arquivos você acha que precisam mudar?"
- Cruzar com exploração da Fase 2 — desafiar se lista parecer maior que o necessário

**Segurança**:
- Auth/autorização: quem pode usar essa feature?
- Multi-tenant: dado pertence a um tenant? scope/policy aplicável?
- Validação de input: campos sensíveis, regras de validação
- Dados sensíveis: algo que não pode logar ou expor em PDF/email?

**Edge cases & não-objetivos**:
- "O que pode dar errado? Estados inválidos? Dados ausentes?"
- "O que explicitamente fica fora?" (importante pra evitar scope creep depois)

**Performance** (se toca listagem/query):
- N+1 esperado? (regra `n-plus-one`)
- Volume estimado de dados?

Iterar até ter respostas concretas em todas as 4 seções obrigatórias.

### Fase 4 — Síntese e validação

1. Montar rascunho do SPEC **inline no chat** (não escrever arquivo ainda)
2. Apresentar usando o template abaixo
3. Pedir ajustes pontuais: *"Algo a corrigir, adicionar ou remover?"*
4. Iterar até usuário aprovar explicitamente

### Fase 5 — Gravação

1. Confirmar slug do arquivo: `SCREAMING_SNAKE_CASE_SPEC.md`
2. Caminho: `docs/<SLUG>_SPEC.md`
3. Se arquivo já existir → perguntar antes de sobrescrever
4. Escrever arquivo
5. Reportar caminho final ao usuário com 1 linha de resumo

## Template SPEC.md

```markdown
# SPEC: <Nome curto da feature/problema>

**Data**: YYYY-MM-DD
**Tipo**: feature | bug | refactor | investigação
**Status**: rascunho | aprovado | implementado

## 1. Problema & Objetivo Verificável

**Problema**: <1-2 parágrafos: situação atual e por que precisa mudar>

**Objetivo verificável** (critérios concretos, testáveis):
- [ ] <ex: teste `it('rejeita estoque negativo', ...)` passa>
- [ ] <ex: usuário exporta PDF e baixa arquivo válido>
- [ ] <ex: query da listagem < 100ms com 1000 registros>

**Não-objetivos** (escopo explicitamente fora):
- <ex: não cobrir export em formato XML nesta iteração>

## 2. Arquivos Afetados (Mínimo)

Lista exaustiva. Diff final NÃO pode tocar arquivos fora desta lista (regra `surgical-changes`).

| Arquivo | Mudança | Motivo |
|---|---|---|
| `app/Filament/Resources/Foo/FooResource.php` | editar | adicionar action X |
| `tests/Feature/Foo/FooExportTest.php` | criar | cobertura do critério 1 |

**Reuso identificado** (Fase 2):
- `app/Services/Pdf/PdfGenerator.php::generate()` → já faz Y, usar em vez de duplicar
- <outros>

## 3. Riscos de Segurança

| Risco | Aplicável? | Mitigação |
|---|---|---|
| Multi-tenant: vazamento entre tenants | sim/não | scope global / policy / `tenant_id` em query |
| Autorização (quem pode usar) | sim/não | Policy `FooPolicy::action`, gate em Filament action |
| Mass-assignment | sim/não | Models wust mantêm unguard global — validar input em Form Request |
| Input não validado | sim/não | Form Request com regras explícitas |
| Dados sensíveis em log/PDF/email | sim/não | <campo X não pode aparecer em Y> |
| N+1 em listagem (regra `n-plus-one`) | sim/não | `with('relacao')` + validar com `detect:n-plus-one` |
| Race condition / concorrência | sim/não | lock pessimista / fila / idempotência |

Se "não aplicável" → justificar em 1 linha.

## 4. Plano de Teste Pest

Testes a criar/atualizar **antes** do código (regras `goal-driven` + `tests`).

```php
// tests/Feature/<Dominio>/<Feature>Test.php

it('<happy path concreto>', function (): void { ... });
it('<edge case 1>', function (): void { ... });
it('<validação rejeita X com mensagem M>', function (): void { ... });
it('<usuário sem permissão recebe 403>', function (): void { ... });
it('<usuário de outro tenant não vê o recurso>', function (): void { ... });
```

Cobertura mínima (marcar conforme aplicável):
- [ ] Caminho feliz
- [ ] Edge cases identificados na Fase 3
- [ ] Validação (cada regra do Form Request)
- [ ] Autorização (sem permissão → 403)
- [ ] Multi-tenant (outro tenant → 404)
- [ ] Sem N+1 (se toca listagem)

## 5. Plano de Execução (TDD)

1. Migration (se aplicável)
2. Escrever testes Pest → vermelho
3. Implementar mínimo pra passar → verde
4. Rodar `php artisan detect:n-plus-one --resource=<Resource>` se toca listagem
5. Rodar skill `pre-commit-review` antes de commitar
6. Commit cirúrgico (diff só contém arquivos da seção 2)
```

## O que NÃO fazer

- ❌ Pular Fase 1 (confirmação) e ir direto pra exploração
- ❌ Pular Fase 4 (rascunho inline) e gravar SPEC sem validação do usuário
- ❌ SPEC sem critério verificável concreto (viola `goal-driven`)
- ❌ Lista de arquivos vaga ("vários arquivos em `app/`") — sempre nominal
- ❌ Seção 3 vazia com "nenhum risco" sem justificar cada linha da tabela
- ❌ Plano de teste só com nomes genéricos ("testar método X") — usar caso concreto
- ❌ Sobrescrever SPEC existente em `docs/` sem confirmar
- ❌ Começar a implementar a feature depois do SPEC (skill termina na gravação)
- ❌ Inserir SPEC em outro caminho que não `docs/<SLUG>_SPEC.md`

## Saída esperada

Ao final da Fase 5:

```
SPEC gerado: docs/<SLUG>_SPEC.md
Próximo passo sugerido: TDD — começar pelos testes da seção 4.
```

Skill encerra. Usuário decide quando implementar.
