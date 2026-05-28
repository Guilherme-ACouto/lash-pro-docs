# Design — Dashboard (Fase 2)

**Data:** 2026-05-23
**Status:** Aprovado para implementação

---

## 1. Visão Geral da Arquitetura

```
GET /api/dashboard?period=TODAY|WEEK|MONTH
        │
        ▼
DashboardController
        │
        ▼
GetDashboardUseCaseImpl
        │
        ├── ClientJpaRepository      (count ativos, crescimento)
        ├── AppointmentJpaRepository (contagens, série por período)
        └── FinancialEntryJpaRepository (receita, despesa, fluxo de caixa)
        │
        ▼
DashboardResponse (DTO único)
```

Frontend:
```
DashboardComponent (orquestrador)
   ├── NgRx Store (dashboard slice)
   ├── kpi-card  ×6            (cards superiores)
   ├── appointments-chart      (barras segmentadas)
   ├── cash-flow-chart         (área dupla)
   ├── today-appointments      (lista do dia)
   └── mini-calendar           (calendário puro Angular)
```

---

## 2. Backend

### 2.1 Pacotes novos

```
domain/port/in/
  GetDashboardUseCase.java

domain/port/out/
  DashboardRepository.java

application/usecase/
  GetDashboardUseCaseImpl.java

infrastructure/persistence/repository/
  DashboardRepositoryImpl.java       ← queries nativas / JPQL

adapter/web/controller/
  DashboardController.java

adapter/web/dto/
  DashboardResponse.java             ← record com todos os campos
```

### 2.2 `DashboardRepository` (port/out)

```java
public interface DashboardRepository {
    long countActiveClients();
    long countNewClientsInPeriod(LocalDate start, LocalDate end);
    AppointmentCounts countAppointmentsByPeriod(LocalDate start, LocalDate end);
    List<AppointmentDayStat> appointmentsSeriesByPeriod(LocalDate start, LocalDate end);
    BigDecimal sumRevenue(LocalDate start, LocalDate end);
    BigDecimal sumReceivable();
    BigDecimal sumPayable();
    List<CashFlowDayStat> cashFlowLast7Days();
    List<TodayAppointmentStat> todayAppointments(LocalDate today);
    List<LocalDate> daysWithAppointmentsInMonth(LocalDate monthStart, LocalDate monthEnd);
}
```

Records internos (todos no mesmo arquivo ou em `domain/model/dashboard/`):
- `AppointmentCounts(long total, long completed, long confirmed, long scheduled, long cancelled)`
- `AppointmentDayStat(LocalDate date, long completed, long confirmed, long scheduled, long cancelled)`
- `CashFlowDayStat(LocalDate date, BigDecimal income, BigDecimal expense)`
- `TodayAppointmentStat(String id, String clientName, String serviceName, String scheduledTime, String status)`

### 2.3 `DashboardRepositoryImpl` — queries principais

Todas as queries usam `@Query` com JPQL ou `nativeQuery = true` direto no `DashboardRepositoryImpl` via `EntityManager` (sem Jpa interface dedicada — queries ad-hoc de agregação).

```java
// Clientes ativos
SELECT COUNT(c) FROM ClientEntity c WHERE c.active = true

// Crescimento (mês anterior)
SELECT COUNT(c) FROM ClientEntity c
WHERE c.active = true AND c.createdAt >= :start AND c.createdAt < :end

// Contagens de agendamentos por período
SELECT a.status, COUNT(a) FROM AppointmentEntity a
WHERE a.scheduledDate >= :start AND a.scheduledDate <= :end
GROUP BY a.status

// Série diária de agendamentos
SELECT a.scheduledDate, a.status, COUNT(a) FROM AppointmentEntity a
WHERE a.scheduledDate >= :start AND a.scheduledDate <= :end
GROUP BY a.scheduledDate, a.status
ORDER BY a.scheduledDate

// Receita no período
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'INCOME' AND f.status = 'PAID'
AND f.paymentDate >= :start AND f.paymentDate <= :end

// Total a receber (sem filtro de período)
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'INCOME' AND f.status IN ('PENDING','OVERDUE')

// Total a pagar
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'EXPENSE' AND f.status IN ('PENDING','OVERDUE')

// Fluxo de caixa últimos 7 dias
SELECT f.paymentDate, f.type, COALESCE(SUM(f.amount), 0)
FROM FinancialEntryEntity f
WHERE f.status = 'PAID' AND f.paymentDate >= :sevenDaysAgo
GROUP BY f.paymentDate, f.type
ORDER BY f.paymentDate
```

### 2.4 `GetDashboardUseCaseImpl`

Chama `DashboardRepository` e monta `DashboardResponse`. Calcula:
- `period` (TODAY/WEEK/MONTH) → `LocalDate start, end`
- `clientsGrowth` = `(currentMonthNew - prevMonthNew) / prevMonthNew * 100`

### 2.5 `DashboardController`

```java
@GetMapping("/api/dashboard")
public ResponseEntity<DashboardResponse> getDashboard(
    @RequestParam(defaultValue = "WEEK") String period) {
    return ResponseEntity.ok(useCase.execute(period));
}
```

### 2.6 `DashboardResponse` (record)

```java
public record DashboardResponse(
    long activeClients,
    double clientsGrowth,
    long totalAppointments,
    long completedAppointments,
    long confirmedAppointments,
    long scheduledAppointments,
    long cancellations,
    BigDecimal revenue,
    BigDecimal receivable,
    BigDecimal payable,
    List<AppointmentDayStatDTO> appointmentsSeries,
    List<CashFlowDayStatDTO> cashFlowSeries,
    List<TodayAppointmentDTO> todayAppointments,
    List<String> daysWithAppointments   // ISO dates "YYYY-MM-DD"
) {}
```

---

## 3. Frontend

### 3.1 Estrutura de arquivos

```
features/dashboard/
├── dashboard.component.ts/html/css   ← já existe (placeholder)
├── dashboard.routes.ts               ← já existe
├── components/
│   ├── kpi-card/                     ← card reutilizável
│   ├── appointments-chart/           ← barras ApexCharts
│   ├── cash-flow-chart/              ← área ApexCharts
│   ├── today-appointments/           ← lista simples
│   └── mini-calendar/                ← calendário puro CSS/Angular
├── services/
│   └── dashboard.service.ts
└── store/
    ├── dashboard.actions.ts
    ├── dashboard.reducer.ts
    ├── dashboard.effects.ts
    └── dashboard.selectors.ts
```

### 3.2 Modelo TypeScript

```typescript
// core/models/dashboard.model.ts
export type DashboardPeriod = 'TODAY' | 'WEEK' | 'MONTH';

export interface AppointmentDayStat {
  date: string;
  completed: number;
  confirmed: number;
  scheduled: number;
  cancelled: number;
}

export interface CashFlowDayStat {
  date: string;
  income: number;
  expense: number;
}

export interface TodayAppointment {
  id: string;
  clientName: string;
  serviceName: string;
  scheduledTime: string;
  status: string;
}

export interface DashboardData {
  activeClients: number;
  clientsGrowth: number;
  totalAppointments: number;
  completedAppointments: number;
  confirmedAppointments: number;
  scheduledAppointments: number;
  cancellations: number;
  revenue: number;
  receivable: number;
  payable: number;
  appointmentsSeries: AppointmentDayStat[];
  cashFlowSeries: CashFlowDayStat[];
  todayAppointments: TodayAppointment[];
  daysWithAppointments: string[];
}
```

### 3.3 NgRx Store

**State:**
```typescript
interface DashboardState {
  data: DashboardData | null;
  period: DashboardPeriod;
  isLoading: boolean;
  error: string | null;
}
```

**Actions:** `loadDashboard`, `loadDashboardSuccess`, `loadDashboardFailure`, `setPeriod`

**Effect:** `setPeriod` e `loadDashboard` → chama `DashboardService.get(period)` → `loadDashboardSuccess`

**Seletores:** `selectDashboardData`, `selectDashboardPeriod`, `selectDashboardLoading`, `selectKpis`, `selectAppointmentsSeries`, `selectCashFlowSeries`, `selectTodayAppointments`, `selectDaysWithAppointments`

### 3.4 Componentes

#### `KpiCardComponent`
- Inputs: `icon`, `label`, `value`, `suffix` (R$ ou vazio), `trend` (número opcional), `color`
- Exibe: ícone rosé, valor principal em destaque, label, variação com seta ↑/↓ colorida
- Sem gráfico circular (donut) — o modelo de referência usa isso mas não temos dados de % significativos; substituir por ícone grande + trend badge. Mais limpo e fiel aos dados reais.

#### `AppointmentsChartComponent`
- ApexCharts `bar` com `stacked: false`
- Séries: Realizados, Confirmados, Agendados, Cancelados
- Cores: `#2E7D32`, `#00838F`, `#1976D2`, `#E53935`
- Eixo X: datas formatadas em pt-BR

#### `CashFlowChartComponent`
- ApexCharts `area`
- Duas séries: Receitas (`#C8A2A2`) e Despesas (`#D4AF37`)
- Fill com gradiente e opacidade 0.3
- Eixo X: últimos 7 dias

#### `TodayAppointmentsComponent`
- Lista Angular pura (sem chart)
- Chip de status com cores do sistema
- Clique navega para `/appointments/:id`

#### `MiniCalendarComponent`
- Grid CSS 7 colunas (Dom–Sáb)
- Input: `daysWithAppointments: string[]`
- Dia atual: fundo rosé
- Dias com agendamento: ponto rosé abaixo do número
- Clique emite `dayClick: EventEmitter<string>`; parent navega para agenda

### 3.5 Layout do `DashboardComponent`

```
┌─────────────────────────────────────────────────────────┐
│  Bom dia, [nome]!          [Hoje] [Semana] [Mês]        │
├──────────┬──────────┬──────────┬──────────┬──────────┬──┤
│ Clientes │ Agend.   │Faturado  │Cancelam. │A Receber │A │
│   KPI    │   KPI    │   KPI    │   KPI    │   KPI    │P │
├──────────┴──────────┴──────────┴──────────┴──────────┴──┤
│                                                          │
│  [Gráfico de Barras — Agendamentos por período]          │
│                                                          │
├────────────────────────────────┬────────────────────────┤
│                                │                        │
│  [Gráfico Área — Fluxo Caixa] │  [Mini Calendário]     │
│                                │  [Agendamentos Hoje]   │
└────────────────────────────────┴────────────────────────┘
```

Em mobile: tudo empilhado em coluna única, cards em grid 2×3.

---

## 4. Biblioteca de Gráficos

**ngx-apexcharts** — instalação:
```bash
npm install apexcharts ng-apexcharts
```

Import no componente standalone:
```typescript
import { NgApexchartsModule } from 'ng-apexcharts';
```

Sem configuração global necessária.

---

## 5. Decisões de Design

| Decisão | Razão |
|---|---|
| 1 endpoint único `/api/dashboard` | Evita N round-trips; dashboard carrega em 1 request |
| `EntityManager` para queries de agregação | Queries complexas com GROUP BY não se encaixam em métodos Spring Data; JPQL ad-hoc via EM é mais limpo que proliferar interfaces Jpa |
| Sem donut chart nos KPI cards | Não há denominador natural para porcentagem; donut seria arbitrário. Trend badge (↑ 8%) é mais informativo |
| Mini calendário em CSS puro | Evitar biblioteca extra para algo tão simples; total controle visual |
| `DashboardPeriod` global no store | Filtro de período afeta múltiplos cards e o gráfico de barras simultaneamente |
