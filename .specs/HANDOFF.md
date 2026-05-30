# Handoff

**Date:** 2026-05-29
**Fase:** 2 — Dashboard ✅ | Módulo Financeiro ✅ (aguardando testes finais)

## Completed ✓

- **Dashboard — backend completo:**
  - `domain/model/dashboard/` — 4 records: `AppointmentCounts`, `AppointmentDayStat`, `CashFlowDayStat`, `TodayAppointmentStat`
  - `domain/port/in/GetDashboardUseCase.java` — interface com enum `Period` e record `DashboardResult`
  - `domain/port/out/DashboardRepository.java` — 10 métodos de agregação
  - `infrastructure/persistence/repository/DashboardRepositoryImpl.java` — queries JPQL via `EntityManager`
  - `application/usecase/GetDashboardUseCaseImpl.java` — lógica de período + crescimento de clientes
  - `adapter/web/dto/DashboardResponse.java` — record com nested DTOs
  - `adapter/web/controller/DashboardController.java` — `GET /api/dashboard?period=WEEK`

- **Dashboard — frontend completo (Rev 1 — 2026-05-29):**
  - `core/models/dashboard.model.ts`
  - `features/dashboard/services/dashboard.service.ts`
  - `features/dashboard/store/` — actions, reducer, effects, selectors
  - `app.config.ts` — slice `dashboard` registrado
  - `components/kpi-card/` — 6 KPIs com trend badge
  - `components/appointments-chart/` — barras ApexCharts (4 séries)
  - `components/cash-flow-chart/` — área ApexCharts (receitas/despesas)
  - `components/today-appointments/` — lista com chips de status + link
  - ~~`components/mini-calendar/`~~ — **removido** (não agregava valor; `selectDaysWithAppointments` e `onCalendarDayClick` também removidos)
  - `dashboard.component` — orquestrador com seletor de período e skeleton loading
  - `ng-apexcharts@1.11.0` instalado (fixado para Angular 18)

- **Sidebar recolhível:**
  - Botão circular flutuante na borda direita (`right: -14px`)
  - Expandida 240px → recolhida 64px, transição CSS 0.25s
  - Tooltips com `matTooltip` no modo recolhido
  - `.sidebar-inner` com `overflow: hidden` clipa labels; `<aside>` com `overflow: visible` expõe o botão

- **Módulo Financeiro — backend completo:**
  - `domain/model/`: `FinancialEntry`, `FinancialEntryType`, `FinancialEntryStatus`, `FinancialEntryExpenseType`, `MonthlyFinancialStat`
  - `domain/port/in/`: `ListFinancialEntriesUseCase` (+ `findDistinctCategories`), `CreateFinancialEntryUseCase`, `UpdateFinancialEntryUseCase`, `DeleteFinancialEntryUseCase`, `ToggleFinancialEntryPaidUseCase`, `GetFinancialSummaryUseCase`
  - `domain/port/out/`: `FinancialEntryRepository` (+ `findDistinctCategories`), `FinancialSummaryRepository`
  - `infrastructure/persistence/`: `FinancialEntryEntity` (ManyToOne read-only para appointment), `FinancialEntryMapper`, `FinancialEntryJpaRepository`, `FinancialEntryRepositoryImpl`, `FinancialSummaryRepositoryImpl` (EntityManager)
  - `adapter/web/controller/FinancialController.java` — 7 endpoints em `/api/financial`
  - V5 Flyway migration: `payment_method`, `expense_type`, `received_from` em `financial_entries`
  - Regras: INCOME+PAID → toggle bloqueado; entry vinculada a agendamento → deleção bloqueada

- **Módulo Financeiro — frontend completo:**
  - `features/financial/models/financial.model.ts` — todos os tipos + `NEXT_MONTH` no `FinancialPeriod`
  - `features/financial/services/financial.service.ts` — HTTP + `getCategories()`
  - `features/financial/store/` — actions/reducer/effects/selectors com `allCategories` no estado
  - `components/financial-summary/` — header resultado previsto + 3 cards (com subtítulos) + gráfico ApexCharts
  - `components/financial-transactions/` — 6 abas + tabela + paginador; `isToggleable(entry)` bloqueia INCOME+PAID
  - `components/financial-form/` — ReactiveForm em MatDialog; `isIncomePaid` bloqueia edição de receita recebida; `locked-banner` vermelho
  - `financial.component` — filtros período/categoria + date pickers para CUSTOM + `ConfirmDeleteDialogComponent` inline
  - Categorias: carregadas via `GET /api/financial/categories` no init, independente da aba/período

- **Agendamentos — integração financeira:**
  - `complete()` abre `PaymentMethodDialogComponent` antes de concluir
  - Entrada financeira criada como PAID com `paymentDate = scheduledDate`

## In Progress

- Nenhum

## Pending (Fase 2)

- Módulo de Estoque

## Blockers

- B01: Zero cobertura de testes automatizados
- B02: `JWT_SECRET` hardcoded no `application.yml` — resolver antes de produção

## Context

- `ng-apexcharts`: **NÃO atualizar** para 2.x (requer Angular ≥ 20)
- Queries de agregação: sempre via `EntityManager`, não Spring Data interfaces
- Paleta: rosé `#C8A2A2`, dourado `#D4AF37`, nude `#FDF6F0`
- Todos os componentes: 3 arquivos separados (.ts + .html + .css), standalone
- `LEFT JOIN FETCH` obrigatório em todas as queries com associações nullable (Hibernate 6)
- Módulo Financeiro aguardando testes finais pela usuária
