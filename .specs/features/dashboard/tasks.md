# Tasks — Dashboard (Fase 2)

**Data:** 2026-05-23
**Status:** Pronto para implementação

Legenda: `[P]` = pode rodar em paralelo com outras tasks do mesmo grupo

---

## Grupo 1 — Backend: Domain + Ports (paralelo)

### TASK-01 [P] — Records de domínio do dashboard
**O quê:** Criar os records internos usados pelo use case e repositório.
**Onde:** `domain/model/dashboard/` (novo pacote)
**Arquivos:**
- `AppointmentCounts.java` — `record AppointmentCounts(long total, long completed, long confirmed, long scheduled, long cancelled)`
- `AppointmentDayStat.java` — `record AppointmentDayStat(LocalDate date, long completed, long confirmed, long scheduled, long cancelled)`
- `CashFlowDayStat.java` — `record CashFlowDayStat(LocalDate date, BigDecimal income, BigDecimal expense)`
- `TodayAppointmentStat.java` — `record TodayAppointmentStat(String id, String clientName, String serviceName, String scheduledTime, String status)`

**Depende de:** nada
**Pronto quando:** todos os records compilam sem erro

---

### TASK-02 [P] — `GetDashboardUseCase` (port/in)
**O quê:** Interface do use case com enum de período.
**Onde:** `domain/port/in/GetDashboardUseCase.java`
```java
public interface GetDashboardUseCase {
    enum Period { TODAY, WEEK, MONTH }
    DashboardResult execute(Period period);
}
```
`DashboardResult` é um record definido nesta interface com todos os campos do design (ver design.md §2.2).

**Depende de:** TASK-01
**Pronto quando:** interface compila

---

### TASK-03 [P] — `DashboardRepository` (port/out)
**O quê:** Interface do repositório com todos os métodos de agregação.
**Onde:** `domain/port/out/DashboardRepository.java`
Métodos conforme design.md §2.2.

**Depende de:** TASK-01
**Pronto quando:** interface compila

---

## Grupo 2 — Backend: Infraestrutura

### TASK-04 — `DashboardRepositoryImpl` com queries JPQL
**O quê:** Implementação usando `EntityManager` para queries de agregação.
**Onde:** `infrastructure/persistence/repository/DashboardRepositoryImpl.java`
**Detalhes:**
- `@Repository @RequiredArgsConstructor`
- Injetar `EntityManager em`
- Cada método executa JPQL via `em.createQuery(...)`
- Queries conforme design.md §2.3
- `countNewClientsInPeriod` usa `createdAt` da `ClientEntity`
- `cashFlowLast7Days` usa `paymentDate` da `FinancialEntryEntity`
- `todayAppointments` faz `LEFT JOIN FETCH` em client e service, ordena por `scheduledTime`
- `daysWithAppointmentsInMonth` retorna `List<LocalDate>` com `SELECT DISTINCT a.scheduledDate`

**Depende de:** TASK-03
**Pronto quando:** classe compila; queries retornam resultados coerentes ao testar manualmente

---

### TASK-05 — `GetDashboardUseCaseImpl`
**O quê:** Implementação do use case — traduz `Period` em `LocalDate start/end` e monta o `DashboardResult`.
**Onde:** `application/usecase/GetDashboardUseCaseImpl.java`
**Detalhes:**
- `@org.springframework.stereotype.Service @RequiredArgsConstructor`
- `Period.TODAY` → `start = end = LocalDate.now()`
- `Period.WEEK` → `start = segunda-feira da semana atual`, `end = hoje`
- `Period.MONTH` → `start = primeiro dia do mês`, `end = hoje`
- `clientsGrowth`: `(newThisMonth - newLastMonth) / max(newLastMonth, 1) * 100.0` (arredondado 1 casa)
- Montar `DashboardResult` com todos os campos

**Depende de:** TASK-02, TASK-04
**Pronto quando:** classe compila; Spring injeta sem erros

---

## Grupo 3 — Backend: Controller + DTO

### TASK-06 — `DashboardResponse` e DTOs aninhados
**O quê:** Records de resposta HTTP (separados dos records de domínio).
**Onde:** `adapter/web/dto/dashboard/`
- `DashboardResponse.java` — record com todos os campos do design.md §2.6
- `AppointmentDayStatDTO.java`
- `CashFlowDayStatDTO.java`
- `TodayAppointmentDTO.java`

**Depende de:** TASK-02
**Pronto quando:** records compilam

---

### TASK-07 — `DashboardController`
**O quê:** Endpoint REST `GET /api/dashboard?period=WEEK`.
**Onde:** `adapter/web/controller/DashboardController.java`
**Detalhes:**
- `@RestController @RequestMapping("/api/dashboard") @RequiredArgsConstructor`
- Método `get(@RequestParam(defaultValue = "WEEK") String period)`
- Converter string → `GetDashboardUseCase.Period` com try/catch → default WEEK se inválido
- Mapear `DashboardResult` → `DashboardResponse` inline no controller
- Retornar `ResponseEntity.ok(response)`

**Depende de:** TASK-05, TASK-06
**Pronto quando:** `GET /api/dashboard` retorna JSON com todos os campos; status 200

---

## Grupo 4 — Frontend: Base (paralelo com Grupo 1-3)

### TASK-08 [P] — Instalar ng-apexcharts
**O quê:** Instalar a biblioteca de gráficos.
**Onde:** `lash-frontend/`
```bash
npm install apexcharts ng-apexcharts
```
**Depende de:** nada
**Pronto quando:** `package.json` contém `ng-apexcharts`; `npm install` sem erros

---

### TASK-09 [P] — Model TypeScript do dashboard
**O quê:** Interfaces e tipos.
**Onde:** `core/models/dashboard.model.ts`
Conforme design.md §3.2: `DashboardPeriod`, `AppointmentDayStat`, `CashFlowDayStat`, `TodayAppointment`, `DashboardData`.

**Depende de:** nada
**Pronto quando:** arquivo compila sem erros TypeScript

---

## Grupo 5 — Frontend: Store + Service

### TASK-10 — `DashboardService`
**O quê:** Service HTTP que chama `/api/dashboard`.
**Onde:** `features/dashboard/services/dashboard.service.ts`
```typescript
get(period: DashboardPeriod): Observable<DashboardData> {
  return this.http.get<DashboardData>('/api/dashboard', { params: { period } });
}
```
**Depende de:** TASK-09
**Pronto quando:** compila; método retorna `Observable<DashboardData>`

---

### TASK-11 — NgRx Store do dashboard
**O quê:** Actions, reducer, effects e selectors.
**Onde:** `features/dashboard/store/`

**`dashboard.actions.ts`:**
- `loadDashboard` (sem props)
- `loadDashboardSuccess({ data: DashboardData })`
- `loadDashboardFailure({ error: string })`
- `setPeriod({ period: DashboardPeriod })`

**`dashboard.reducer.ts`:**
- State: `{ data: DashboardData | null, period: DashboardPeriod, isLoading: boolean, error: string | null }`
- Estado inicial: `period = 'WEEK'`
- `setPeriod` → atualiza period e liga `isLoading = true`
- `loadDashboard` → `isLoading = true`
- `loadDashboardSuccess` → `data = payload, isLoading = false`
- `loadDashboardFailure` → `error, isLoading = false`

**`dashboard.effects.ts`:**
- `loadDashboard$` → chama `DashboardService.get(period)` (usa `withLatestFrom(selectDashboardPeriod)`)
- `setPeriod$` → dispara `loadDashboard` após mudar período

**`dashboard.selectors.ts`:**
- `selectDashboardData`, `selectDashboardPeriod`, `selectDashboardLoading`
- `selectTodayAppointments` — `data?.todayAppointments ?? []`
- `selectDaysWithAppointments` — `data?.daysWithAppointments ?? []`

**Depende de:** TASK-09, TASK-10
**Pronto quando:** compila sem erros TypeScript

---

### TASK-12 — Registrar slice `dashboard` no app config
**O quê:** Adicionar `dashboard: dashboardReducer` ao `provideStore` e `DashboardEffects` ao `provideEffects`.
**Onde:** `app.config.ts`
**Depende de:** TASK-11
**Pronto quando:** app compila; store contém slice `dashboard`

---

## Grupo 6 — Frontend: Sub-componentes (paralelo)

### TASK-13 [P] — `KpiCardComponent`
**O quê:** Card reutilizável de KPI.
**Onde:** `features/dashboard/components/kpi-card/`
**Inputs:** `icon: string`, `label: string`, `value: string | number`, `prefix?: string`, `trend?: number`, `accentColor?: string`
**Visual:**
- Ícone grande no topo (cor `accentColor` ou rosé padrão)
- Valor principal em `28px bold`
- Label em `12px uppercase rosé`
- Trend badge: `↑ X%` (verde) ou `↓ X%` (vermelho), oculto se `trend` for null
- Card com borda superior colorida por `accentColor`

**Depende de:** TASK-09
**Pronto quando:** componente renderiza corretamente com dados mockados

---

### TASK-14 [P] — `AppointmentsChartComponent`
**O quê:** Gráfico de barras segmentadas por status.
**Onde:** `features/dashboard/components/appointments-chart/`
**Detalhes:**
- Import `NgApexchartsModule`
- `@Input() series: AppointmentDayStat[]`
- Converte para formato ApexCharts: 4 séries (Realizados, Confirmados, Agendados, Cancelados)
- Cores: `['#2E7D32', '#00838F', '#1976D2', '#E53935']`
- Tipo: `bar`, `stacked: false`
- Eixo X: datas formatadas `dd/MM`
- Tooltip em pt-BR
- Altura: 280px

**Depende de:** TASK-08, TASK-09
**Pronto quando:** componente renderiza gráfico com dados mockados

---

### TASK-15 [P] — `CashFlowChartComponent`
**O quê:** Gráfico de área dupla (receita vs despesa).
**Onde:** `features/dashboard/components/cash-flow-chart/`
**Detalhes:**
- Import `NgApexchartsModule`
- `@Input() series: CashFlowDayStat[]`
- Duas séries: Receitas (`#C8A2A2`) e Despesas (`#D4AF37`)
- Tipo: `area`, fill `gradient`, opacidade 0.3
- Eixo X: últimos 7 dias formatados `dd/MM`
- Tooltip mostra receita, despesa e saldo (`receita - despesa`)
- Altura: 220px

**Depende de:** TASK-08, TASK-09
**Pronto quando:** componente renderiza com dados mockados

---

### TASK-16 [P] — `TodayAppointmentsComponent`
**O quê:** Lista de agendamentos do dia.
**Onde:** `features/dashboard/components/today-appointments/`
**Detalhes:**
- `@Input() appointments: TodayAppointment[]`
- Cada item: horário (bold rosé) + nome do cliente + chip de status
- Chip de status segue mesmas cores de `appointment-detail` (SCHEDULED=azul, CONFIRMED=teal, etc.)
- Clique navega via `Router` para `/appointments/:id`
- Estado vazio: ícone `event_available` + "Nenhum agendamento para hoje"
- Scroll interno se lista longa (max-height 280px)

**Depende de:** TASK-09
**Pronto quando:** componente renderiza lista e estado vazio

---

### TASK-17 [P] — `MiniCalendarComponent`
**O quê:** Calendário mensal puro Angular/CSS.
**Onde:** `features/dashboard/components/mini-calendar/`
**Inputs:** `daysWithAppointments: string[]` (ISO dates), `currentMonth?: Date`
**Output:** `dayClick: EventEmitter<string>`
**Detalhes:**
- Grid CSS 7 colunas (Dom–Sáb)
- Cabeçalho: nome do mês + ano + botões `<` `>`
- Dia atual: fundo rosé `#C8A2A2`, texto branco
- Dia com agendamento: ponto rosé `●` abaixo do número
- Clique emite ISO date string
- Sem dependência de biblioteca externa

**Depende de:** nada (só CSS/Angular puro)
**Pronto quando:** calendário renderiza mês atual com navegação funcional

---

## Grupo 7 — Frontend: Composição final

### TASK-18 — `DashboardComponent` — layout e integração
**O quê:** Compor todos os sub-componentes com dados da store.
**Onde:** `features/dashboard/dashboard.component.ts/html/css`
**Detalhes:**

**`ngOnInit`:** `this.store.dispatch(DashboardActions.loadDashboard())`

**Template — estrutura:**
```
saudação (bom dia/tarde/noite + "Olá!")
toggle período [Hoje] [Semana] [Mês]

grid 3×2 de KpiCards:
  Clientes | Agendamentos | Faturamento
  Cancelamentos | A Receber | A Pagar

AppointmentsChartComponent (largura total)

linha inferior 2 colunas:
  CashFlowChartComponent | MiniCalendar + TodayAppointments
```

**KPI cards valores:**
- Clientes: `data.activeClients`, trend `data.clientsGrowth`, ícone `people`, cor `#C8A2A2`
- Agendamentos: `data.totalAppointments`, ícone `calendar_today`, cor `#1976D2`
- Faturamento: `currencyBr` pipe em `data.revenue`, ícone `attach_money`, cor `#2E7D32`
- Cancelamentos: `data.cancellations`, ícone `cancel`, cor `#E53935`
- A Receber: `currencyBr` em `data.receivable`, ícone `arrow_circle_down`, cor `#00838F`
- A Pagar: `currencyBr` em `data.payable`, ícone `arrow_circle_up`, cor `#D4AF37`

**Skeleton:** `@if (isLoading$ | async)` mostra placeholder animado em cada seção

**Toggle período:** ao mudar dispara `DashboardActions.setPeriod({ period })`

**Mini calendário:** ao clicar em dia navega para `/appointments?date=YYYY-MM-DD`

**CSS:** layout `display: grid`, responsivo (mobile = coluna única)

**Depende de:** TASK-12, TASK-13, TASK-14, TASK-15, TASK-16, TASK-17
**Pronto quando:** dashboard abre, KPIs carregam da API, gráficos renderizam, toggle de período funciona, mini calendário navega para agenda

---

## Ordem de Execução Recomendada

```
[Paralelo]  TASK-01 + TASK-08 + TASK-09 + TASK-17
     ↓
[Paralelo]  TASK-02 + TASK-03 + TASK-10
     ↓
[Paralelo]  TASK-04 + TASK-11
     ↓
[Paralelo]  TASK-05 + TASK-12 + TASK-13 + TASK-14 + TASK-15 + TASK-16
     ↓
            TASK-06
     ↓
            TASK-07
     ↓
            TASK-18
```

**Total: 18 tasks | Estimativa: 1 sessão de implementação**
