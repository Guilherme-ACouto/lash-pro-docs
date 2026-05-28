# Handoff

**Date:** 2026-05-23
**Fase:** 2 — Dashboard implementado ✅ | Sidebar recolhível ✅

## Completed ✓

- **Dashboard — backend completo:**
  - `domain/model/dashboard/` — 4 records: `AppointmentCounts`, `AppointmentDayStat`, `CashFlowDayStat`, `TodayAppointmentStat`
  - `domain/port/in/GetDashboardUseCase.java` — interface com enum `Period` e record `DashboardResult`
  - `domain/port/out/DashboardRepository.java` — 10 métodos de agregação
  - `infrastructure/persistence/repository/DashboardRepositoryImpl.java` — queries JPQL via `EntityManager`
  - `application/usecase/GetDashboardUseCaseImpl.java` — lógica de período + crescimento de clientes
  - `adapter/web/dto/DashboardResponse.java` — record com nested DTOs
  - `adapter/web/controller/DashboardController.java` — `GET /api/dashboard?period=WEEK`

- **Dashboard — frontend completo:**
  - `core/models/dashboard.model.ts`
  - `features/dashboard/services/dashboard.service.ts`
  - `features/dashboard/store/` — actions, reducer, effects, selectors
  - `app.config.ts` — slice `dashboard` registrado
  - `components/kpi-card/` — 6 KPIs com trend badge
  - `components/appointments-chart/` — barras ApexCharts (4 séries)
  - `components/cash-flow-chart/` — área ApexCharts (receitas/despesas)
  - `components/today-appointments/` — lista com chips de status + link
  - `components/mini-calendar/` — calendário puro Angular/CSS com navegação de mês
  - `dashboard.component` — orquestrador com seletor de período e skeleton loading
  - `ng-apexcharts@1.11.0` instalado (fixado para Angular 18)

- **Sidebar recolhível:**
  - Botão circular flutuante na borda direita (`right: -14px`)
  - Expandida 240px → recolhida 64px, transição CSS 0.25s
  - Tooltips com `matTooltip` no modo recolhido
  - `.sidebar-inner` com `overflow: hidden` clipa labels; `<aside>` com `overflow: visible` expõe o botão

- **lash-docs atualizados:** `STATE.md` (rev 5) + `HANDOFF.md`

## In Progress

- Nenhum

## Pending (Fase 2)

- Módulo Financeiro
- Módulo de Estoque

## Blockers

- B01: Zero cobertura de testes automatizados
- B02: `JWT_SECRET` hardcoded no `application.yml` — resolver antes de produção

## Context

- Dashboard ainda não testado pela usuária — testar amanhã
- `ng-apexcharts`: **NÃO atualizar** para 2.x (requer Angular ≥ 20)
- Queries de agregação: sempre via `EntityManager`, não Spring Data interfaces
- Paleta: rosé `#C8A2A2`, dourado `#D4AF37`, nude `#FDF6F0`
- Todos os componentes: 3 arquivos separados (.ts + .html + .css), standalone
