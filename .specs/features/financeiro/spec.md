# Spec — Módulo Financeiro

**Data:** 2026-05-23
**Complexidade:** Large
**Status:** Especificado

---

## Visão Geral

Tela centralizada de gestão financeira do negócio. Apresenta resumo do mês em destaque, cards de recebimentos e despesas com progresso percentual, gráfico histórico mensal, e uma tabela paginada de transações organizadas em abas por categoria. Permite criar, editar e excluir lançamentos manuais; lançamentos gerados por agendamentos podem ser marcados como pagos/pendentes mas não excluídos.

---

## Modelo de Dados — Extensões Necessárias

### Campos a adicionar em `financial_entries` (V5 migration)

| Campo | Tipo SQL | Nullable | Descrição |
|---|---|---|---|
| `payment_method` | `VARCHAR(50)` | sim | Forma de pagamento: PIX, Dinheiro, Cartão Débito, Cartão Crédito, Transferência, etc. |
| `expense_type` | `VARCHAR(20)` | sim | Subcategoria da despesa: `FIXED`, `VARIABLE`, `PEOPLE`, `TAX`, `TRANSFER` |
| `received_from` | `VARCHAR(255)` | sim | Nome de quem realizou o pagamento (para receitas manuais; entradas de agendamento usam o nome do cliente via JOIN) |

> **Nota:** O `type` atual (`INCOME`/`EXPENSE`) permanece inalterado. A aba "Transferências" filtra por `expense_type = 'TRANSFER'` com `type = 'EXPENSE'`. Não há um tipo TRANSFER separado para evitar quebrar o CHECK constraint existente e a lógica do dashboard.

### Categorias padronizadas

As categorias (`category`) são strings livres no modelo, mas o frontend apresenta sugestões:

**INCOME:** Serviço, Venda de produto, Gorjeta, Outros recebimentos  
**EXPENSE FIXED:** Aluguel, Internet, Telefone, Software/Assinatura, Outros fixos  
**EXPENSE VARIABLE:** Material de consumo, Equipamentos, Marketing, Outros variáveis  
**EXPENSE PEOPLE:** Salário, Comissão, Pró-labore, Outros pessoal  
**EXPENSE TAX:** Imposto, Simples Nacional, DAS, Outros impostos  
**EXPENSE TRANSFER:** Transferência bancária, Sangria de caixa, Outros

---

## Requisitos Funcionais

### FIN-01 — Cabeçalho de resumo do mês

| ID | Requisito |
|---|---|
| FIN-01.1 | Exibir o **resultado previsto do mês corrente** em destaque: `INCOME(PAID+PENDING) − EXPENSE(PAID+PENDING)` no mês atual |
| FIN-01.2 | Resultado positivo exibido em verde; negativo em vermelho |
| FIN-01.3 | Subtítulo indica o período: "Previsão para [mês/ano]" |

### FIN-02 — Card de Recebimentos

| ID | Requisito |
|---|---|
| FIN-02.1 | Exibir valor já **recebido** no mês (`INCOME` + `PAID`) |
| FIN-02.2 | Exibir valor **previsto** no mês (`INCOME` + `PAID` + `PENDING`) |
| FIN-02.3 | Barra de progresso percentual: `recebido / previsto × 100` (capped em 100%) |
| FIN-02.4 | Percentual exibido ao lado da barra (ex: "72%") |

### FIN-03 — Card de Despesas

| ID | Requisito |
|---|---|
| FIN-03.1 | Exibir valor já **pago** no mês (`EXPENSE` + `PAID`) |
| FIN-03.2 | Exibir valor **previsto** no mês (`EXPENSE` + `PAID` + `PENDING`) |
| FIN-03.3 | Barra de progresso percentual: `pago / previsto × 100` (capped em 100%) |
| FIN-03.4 | Percentual exibido ao lado da barra |

### FIN-04 — Card de Saldo

| ID | Requisito |
|---|---|
| FIN-04.1 | Exibir **saldo atual** = soma de todos `INCOME PAID` − `EXPENSE PAID` (sem filtro de período) |
| FIN-04.2 | Exibir **previsão do mês** = resultado previsto do mês corrente (igual a FIN-01.1) |
| FIN-04.3 | Valor do saldo positivo em verde, negativo em vermelho |

### FIN-05 — Gráfico histórico mensal

| ID | Requisito |
|---|---|
| FIN-05.1 | Gráfico de barras agrupadas comparando Recebimentos (rosé `#C8A2A2`) vs Despesas (cinza `#9E9E9E`) por mês |
| FIN-05.2 | Exibir os últimos **6 meses** incluindo o mês corrente |
| FIN-05.3 | Valores consolidados: apenas entradas `PAID` no histórico |
| FIN-05.4 | Tooltip ao passar o mouse exibe Recebimentos, Despesas e Saldo do mês |
| FIN-05.5 | Eixo Y em R$ com formatação brasileira |

### FIN-06 — Filtros

| ID | Requisito |
|---|---|
| FIN-06.1 | Filtro de **período** com opções: Este mês, Esta semana, Hoje, Últimos 30 dias, Últimos 90 dias, Este ano, Personalizado |
| FIN-06.2 | Período "Personalizado" exibe dois campos de data (de / até) |
| FIN-06.3 | Filtro de **categoria**: dropdown com todas as categorias existentes nas transações do período + opção "Todas" |
| FIN-06.4 | Filtros se aplicam à tabela de transações (FIN-07); os cards e gráfico sempre refletem o mês corrente (independente do filtro) |
| FIN-06.5 | Seleção padrão ao entrar na tela: período = "Este mês", categoria = "Todas" |

### FIN-07 — Abas de transações

| ID | Requisito |
|---|---|
| FIN-07.1 | Abas disponíveis: **Recebimentos**, **Despesas fixas**, **Despesas variáveis**, **Pessoas**, **Impostos**, **Transferências** |
| FIN-07.2 | Cada aba exibe apenas as transações do tipo/subtipo correspondente: |
| | → Recebimentos: `type = INCOME` |
| | → Despesas fixas: `type = EXPENSE AND expense_type = 'FIXED'` |
| | → Despesas variáveis: `type = EXPENSE AND expense_type = 'VARIABLE'` |
| | → Pessoas: `type = EXPENSE AND expense_type = 'PEOPLE'` |
| | → Impostos: `type = EXPENSE AND expense_type = 'TAX'` |
| | → Transferências: `type = EXPENSE AND expense_type = 'TRANSFER'` |
| FIN-07.3 | Badge com contagem de registros em cada aba |
| FIN-07.4 | Aba ativa persiste enquanto o usuário muda filtros de período/categoria |

### FIN-08 — Tabela de transações

| ID | Requisito |
|---|---|
| FIN-08.1 | Colunas: **Data** (due_date), **Descrição**, **Recebido de / Pago a**, **Categoria**, **Valor**, **Tipo pagamento**, **Pago** |
| FIN-08.2 | Coluna "Recebido de / Pago a": para entradas vinculadas a agendamento exibe o nome do cliente (JOIN com appointments→clients); para entradas manuais exibe `received_from` |
| FIN-08.3 | Coluna "Valor": INCOME exibido em verde com prefixo "+"; EXPENSE exibido em vermelho com prefixo "−" |
| FIN-08.4 | Coluna "Pago": toggle/chip clicável — alterna entre `PENDING` ↔ `PAID` com chamada de API imediata |
| FIN-08.5 | Status `OVERDUE` exibido em vermelho na coluna "Pago" (não clicável para alterar para PENDING) |
| FIN-08.6 | Entradas vinculadas a agendamento exibem ícone de link (não podem ser excluídas) |
| FIN-08.7 | Clicar em uma linha abre o formulário de edição (exceto entradas de agendamento: abre em modo read-only com link para o agendamento) |
| FIN-08.8 | Botão de excluir na linha (ícone lixeira) — só visível para entradas sem `appointment_id`; exige confirmação |
| FIN-08.9 | Ordenação padrão: `due_date DESC` |
| FIN-08.10 | Paginação: 20 por página |

### FIN-09 — Formulário de nova/editar transação

| ID | Requisito |
|---|---|
| FIN-09.1 | Botão "Nova transação" abre um drawer lateral (não página separada) |
| FIN-09.2 | Campos obrigatórios: Tipo (INCOME/EXPENSE), Subcategoria (expense_type, apenas para EXPENSE), Descrição, Valor, Data de vencimento |
| FIN-09.3 | Campos opcionais: Categoria, Recebido de / Pago a (received_from), Forma de pagamento, Observações, Data de pagamento, Status |
| FIN-09.4 | Ao salvar com `payment_date` preenchido, `status` deve ser automaticamente definido como `PAID` |
| FIN-09.5 | Ao salvar sem `payment_date`, `status` deve ser `PENDING` |
| FIN-09.6 | Edição de transação vinculada a agendamento: somente campos `payment_method`, `payment_date`, `status` e `notes` são editáveis |
| FIN-09.7 | Validação em tempo real: valor deve ser > 0; data de pagamento não pode ser futura se status = PAID |

### FIN-10 — Backend: use cases necessários

| ID | Requisito |
|---|---|
| FIN-10.1 | `GetFinancialSummaryUseCase`: retorna resumo do mês (resultado previsto, saldo atual, cards de recebimento/despesa, série dos 6 meses para o gráfico) |
| FIN-10.2 | `ListFinancialEntriesUseCase`: listagem paginada com filtros de período, categoria e expense_type |
| FIN-10.3 | `CreateFinancialEntryUseCase`: cria entrada manual |
| FIN-10.4 | `UpdateFinancialEntryUseCase`: atualiza entrada (validar restrições para entradas de agendamento) |
| FIN-10.5 | `DeleteFinancialEntryUseCase`: exclui entrada — lança exceção se `appointment_id != null` |
| FIN-10.6 | `ToggleFinancialEntryPaidUseCase`: alterna PENDING↔PAID com lógica de `payment_date` automático |

### FIN-11 — Backend: endpoints REST

| Método | Path | Use Case |
|---|---|---|
| `GET` | `/api/financial/summary` | `GetFinancialSummaryUseCase` |
| `GET` | `/api/financial/entries?from=&to=&category=&expenseType=&page=&size=` | `ListFinancialEntriesUseCase` |
| `POST` | `/api/financial/entries` | `CreateFinancialEntryUseCase` |
| `PUT` | `/api/financial/entries/{id}` | `UpdateFinancialEntryUseCase` |
| `DELETE` | `/api/financial/entries/{id}` | `DeleteFinancialEntryUseCase` |
| `PATCH` | `/api/financial/entries/{id}/toggle-paid` | `ToggleFinancialEntryPaidUseCase` |

---

## Requisitos Não Funcionais

- Layout responsivo: em mobile os cards empilham em coluna; gráfico reduz altura; tabela habilita scroll horizontal
- Gráfico usa `ng-apexcharts@1.11.0` (versão fixada, já instalada)
- Paleta de cores: rosé `#C8A2A2`, dourado `#D4AF37`, verde `#2E7D32`, vermelho `#E53935`, cinza despesas `#9E9E9E`
- Drawer de nova transação usa `MatDrawer` ou `MatDialog` (verificar qual está em uso nos outros módulos)
- Pipes compartilhados `currencyBr` e `datePtbr` obrigatórios
- Estado global via NgRx (slice `financial`)

---

## Fora de Escopo (esta fase)

- Relatórios exportáveis (PDF/CSV)
- Conciliação bancária (importação de extrato)
- Recorrência automática de lançamentos fixos
- Comissões por profissional (Fase 3)
- Multi-categoria por lançamento
- Alertas de vencimento por e-mail/WhatsApp

---

## Impacto em Módulos Existentes

| Módulo | Impacto |
|---|---|
| Agendamentos | `CompleteAppointmentUseCaseImpl` já cria `FinancialEntry` — nenhuma mudança necessária |
| Dashboard | Já consome `financial_entries` — os novos campos (`payment_method`, `expense_type`, `received_from`) são nullable e não quebram as queries existentes |
| V5 Migration | ALTER TABLE adiciona 3 colunas nullable — sem risco de dados existentes |

---

## Estrutura de Arquivos Esperada

### Backend

```
domain/
  model/         FinancialEntry.java (+ 3 novos campos)
                 FinancialEntryExpenseType.java (enum: FIXED, VARIABLE, PEOPLE, TAX, TRANSFER)
  port/in/       GetFinancialSummaryUseCase.java
                 ListFinancialEntriesUseCase.java
                 CreateFinancialEntryUseCase.java
                 UpdateFinancialEntryUseCase.java
                 DeleteFinancialEntryUseCase.java
                 ToggleFinancialEntryPaidUseCase.java
  port/out/      FinancialEntryRepository.java (estender com novos métodos)

application/
  usecase/       GetFinancialSummaryUseCaseImpl.java
                 ListFinancialEntriesUseCaseImpl.java
                 CreateFinancialEntryUseCaseImpl.java
                 UpdateFinancialEntryUseCaseImpl.java
                 DeleteFinancialEntryUseCaseImpl.java
                 ToggleFinancialEntryPaidUseCaseImpl.java

infrastructure/
  persistence/
    entity/      FinancialEntryEntity.java (+ novos campos)
    repository/  FinancialEntryJpaRepository.java (+ novos métodos de query)
                 FinancialEntryRepositoryImpl.java (+ implementações)
    mapper/      FinancialEntryMapper.java (atualizar)
  web/
    controller/  FinancialController.java
    dto/         FinancialSummaryResponse.java
                 FinancialEntryResponse.java
                 CreateFinancialEntryRequest.java
                 UpdateFinancialEntryRequest.java

resources/db/migration/
                 V5__financial_entries_new_fields.sql
```

### Frontend

```
features/financeiro/
  components/
    financial-summary/         (header + cards + gráfico)
    financial-transactions/    (abas + tabela + paginação)
    financial-form/            (drawer de nova/editar transação)
  services/
    financial.service.ts
  store/
    financial.actions.ts
    financial.reducer.ts
    financial.effects.ts
    financial.selectors.ts
  models/
    financial.model.ts
  financeiro.routes.ts
```
