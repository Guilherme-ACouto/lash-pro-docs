# Spec — Dashboard (Fase 2)

**Data:** 2026-05-23
**Complexidade:** Large
**Status:** Especificado

---

## Visão Geral

Tela inicial do sistema após login. Apresenta uma visão consolidada do negócio em tempo real: KPIs financeiros, volume de agendamentos, histórico de faturamento e mini calendário para navegação rápida.

---

## Fontes de Dados Disponíveis

| Dado | Tabela / Derivação |
|---|---|
| Clientes cadastrados | `clients` (total e ativos) |
| Agendamentos | `appointments` (por status, por período) |
| Cancelamentos | `appointments WHERE status = 'CANCELLED'` |
| Faturamento realizado | `financial_entries WHERE type='INCOME' AND status='PAID'` |
| Contas a receber | `financial_entries WHERE type='INCOME' AND status IN ('PENDING','OVERDUE')` |
| Contas a pagar | `financial_entries WHERE type='EXPENSE' AND status IN ('PENDING','OVERDUE')` |
| Fluxo de caixa | INCOME PAID − EXPENSE PAID por dia/semana/mês |

> **Nota:** "Clientes em débito" não tem suporte direto no modelo atual (sem conceito de crédito/débito por cliente). Será exibido como agendamentos com entrada financeira em OVERDUE agrupados por cliente — implementado no Módulo Financeiro (Fase 2). O card aparece no dashboard mas exibe "—" até o módulo financeiro ser entregue.

---

## Requisitos Funcionais

### KPIs — Cards superiores

| ID | Requisito |
|---|---|
| DSH-01 | Card "Clientes" exibe total de clientes ativos cadastrados com variação percentual em relação ao mês anterior |
| DSH-02 | Card "Agendamentos" exibe contagem de agendamentos com filtro Hoje / Semana / Mês (toggle); exclui CANCELLED e NO_SHOW da contagem principal |
| DSH-03 | Card "Faturamento" exibe soma de `financial_entries` INCOME + PAID no período selecionado (mesmo filtro do DSH-02) |
| DSH-04 | Card "Cancelamentos" exibe contagem de agendamentos com status CANCELLED ou NO_SHOW no período selecionado |
| DSH-05 | Card "A Receber" exibe soma de `financial_entries` INCOME + PENDING/OVERDUE (independe do filtro de período — sempre total em aberto) |
| DSH-06 | Card "A Pagar" exibe soma de `financial_entries` EXPENSE + PENDING/OVERDUE (sempre total em aberto) |

### Gráfico de Agendamentos por Período

| ID | Requisito |
|---|---|
| DSH-07 | Gráfico de barras mostrando volume de agendamentos agrupados por dia (se filtro = Semana) ou por semana (se filtro = Mês) |
| DSH-08 | Barras segmentadas por status: Realizados (verde), Confirmados (teal), Agendados (azul), Cancelados/No-show (vermelho) |
| DSH-09 | Tooltip ao passar o mouse exibe detalhes do período |

### Gráfico de Faturamento

| ID | Requisito |
|---|---|
| DSH-10 | Gráfico de área mostrando faturamento diário (INCOME PAID) nos últimos 7 dias |
| DSH-11 | Segunda linha/área sobreposta com despesas (EXPENSE PAID) para visualização de fluxo de caixa |
| DSH-12 | Tooltip exibe receita, despesa e saldo líquido do dia |

### Mini Calendário

| ID | Requisito |
|---|---|
| DSH-13 | Mini calendário mensal no canto inferior direito |
| DSH-14 | Dias com agendamentos exibem indicador visual (ponto rosé) |
| DSH-15 | Clicar em um dia navega para a agenda (`/appointments?date=YYYY-MM-DD`) |
| DSH-16 | Dia atual destacado com fundo rosé |

### Agendamentos do Dia

| ID | Requisito |
|---|---|
| DSH-17 | Lista dos próximos agendamentos do dia corrente (ordenados por horário) |
| DSH-18 | Cada item exibe: horário, nome do cliente, serviço e chip de status |
| DSH-19 | Clicar em um item navega para `/appointments/:id` |
| DSH-20 | Se não houver agendamentos, exibe estado vazio amigável |

### Comportamento Geral

| ID | Requisito |
|---|---|
| DSH-21 | Filtro de período (Hoje / Semana / Mês) é global para DSH-02, DSH-03, DSH-04 e DSH-07 |
| DSH-22 | Dashboard carrega todos os dados em paralelo (sem bloquear a renderização por um único request) |
| DSH-23 | Estado de loading por seção independente (skeleton em cada card/gráfico) |
| DSH-24 | Dashboard é a rota `/dashboard` e a rota padrão `/` redireciona para ela |

---

## Requisitos Não Funcionais

- Biblioteca de gráficos: **ngx-apexcharts** (ApexCharts) — suporta donut, barras e área; produz visual moderno alinhado com o padrão do sistema
- Paleta de cores dos gráficos segue o sistema: rosé `#C8A2A2`, dourado `#D4AF37`, verde `#2E7D32`, teal `#00838F`, vermelho `#E53935`
- Layout responsivo: em mobile, cards empilham em coluna única; gráficos reduzem altura
- Nenhum dado em cache — sempre busca ao navegar para o dashboard

---

## Fora de Escopo (Fase 2 posterior)

- Clientes em débito (depende do Módulo Financeiro completo)
- Metas / targets (ex: "meta de R$ X no mês")
- Exportação de relatórios
- Comparação entre profissionais (sistema é single-user nesta fase)

---

## Endpoint Necessário

Único endpoint agregado:

```
GET /api/dashboard?period=TODAY|WEEK|MONTH
```

Retorna um único objeto `DashboardResponse` com todos os KPIs e séries para os gráficos, evitando múltiplos round-trips.

```json
{
  "activeClients": 42,
  "clientsGrowth": 8.3,
  "appointments": { "total": 12, "completed": 5, "confirmed": 3, "scheduled": 4 },
  "cancellations": 2,
  "revenue": 850.00,
  "receivable": 320.00,
  "payable": 150.00,
  "appointmentsSeries": [
    { "date": "2026-05-19", "completed": 3, "confirmed": 1, "scheduled": 2, "cancelled": 0 }
  ],
  "cashFlowSeries": [
    { "date": "2026-05-17", "income": 200.00, "expense": 50.00 }
  ],
  "todayAppointments": [
    { "id": "...", "clientName": "Maria", "serviceName": "Extensão", "scheduledTime": "09:00", "status": "CONFIRMED" }
  ],
  "daysWithAppointments": ["2026-05-19", "2026-05-21", "2026-05-23"]
}
```
