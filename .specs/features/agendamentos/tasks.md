# Agendamentos — Tasks

## Status Geral

```
T01 [x] Domain + Ports
T02 [x] Use Cases
T03 [x] Infrastructure
T04 [x] Controller + DTOs
T05 [x] CLI-13 / SVC-13
T06 [x] Frontend Model
T07 [x] AppointmentHttp
T08 [x] NgRx Store
T09 [x] AppointmentDayComponent (timeline)
T10 [x] AppointmentFormComponent
T11 [x] AppointmentDetailComponent
T12 [x] Integração Final
```

**Implementado em:** 2026-05-22 — mvn compile ✅ — npm run build ✅ — aguardando teste pela usuária
**Rev 2 (2026-05-22):** Correções pós-implementação — Hibernate 500, label click, redesign para grade semanal
**Rev 3 (2026-05-22):** NO_SHOW status; sidebar com mini-cal + legenda + profissional; toggle dia/semana/mês; view mês; AppointmentPopupComponent; DayAppointmentsModalComponent
**Rev 4 (2026-05-22):** Pós-teste — bugs corrigidos + modal de agendamentos + lógica de exclusão de cliente refinada:
- Bug: `AppointmentController.toResponse()` — NPE em `r.clientId().toString()` quando cliente desvinculado; corrigido com null-guard
- Bug: forkJoin em `loadWeek()` swallowa 500 silenciosamente — raiz do problema "agenda vazia"
- Bug: `@Transactional` importado de `jakarta.transaction` em vez de `org.springframework.transaction.annotation`
- Bug: queries com `JOIN FETCH` em campo nullable causavam problema no Hibernate 6; padronizado para `LEFT JOIN FETCH`
- Bug: reducer `deactivateSuccess` usava `.filter()` (excluía) em vez de `.map()` (atualizava `active`)
- Bug: backend `ListClientsUseCaseImpl` / `ListServicesUseCaseImpl` com `active = true` hardcoded — deativados sumiam no reload
- Feature: `AppointmentSummary` — domain record com `id, scheduledDate, scheduledTime, serviceName`
- Feature: `HasFutureAppointmentsException` carrega `List<AppointmentSummary>`; `GlobalExceptionHandler` serializa como `appointments` no 409
- Feature: V4 Flyway migration — `appointments.client_id` nullable
- Feature: `AppointmentMapper` com null-guard em `client`
- Feature: `AppointmentRepository` port/out — novos métodos `findFutureActiveByClientId`, `findFutureActiveByServiceId`, `unlinkClientFromPastAppointments`, `deleteFutureAppointmentsByClientId`
- Feature: `AppointmentsWarningDialogComponent` em `shared/components/` — modal com lista de agendamentos clicáveis
- Feature: filtros Todos / Ativos / Inativos nas telas de Clientes e Serviços
- Feature: botões Desativar + Reativar sempre visíveis nas listagens (desabilitados por `active`)
- Feature: `DeleteClientUseCaseImpl` — bloqueia se futuros SCHEDULED/CONFIRMED; apaga futuros CANCELLED; desvincula passados

---

## Grafo de Dependências

```
T01 → T02 ─────────────→ T04
T01 → T03 ─────────────→ T04
T01 → T03 ─────────────→ T05

T06 → T07 → T08 → T09 ─┐
                   T10 ─┤→ T12
                   T11 ─┘
```

**Paralelismo:**
- `[P]` T02 + T03 após T01
- `[P]` T04 + T05 após T02+T03
- `[P]` T06 independente (começa junto com T01)
- `[P]` T09 + T10 + T11 após T08

---

## Tasks

### T01 — Domain + Ports

**O que:** Criar modelos de domínio, exceções e todas as interfaces de porta para Agendamentos e FinancialEntry mínimo.

**Onde:**
- `domain/model/Appointment.java`
- `domain/model/AppointmentStatus.java` (enum: SCHEDULED, CONFIRMED, COMPLETED, CANCELLED, NO_SHOW)
- `domain/model/FinancialEntry.java`
- `domain/exception/AppointmentNotFoundException.java`
- `domain/exception/AppointmentConflictException.java`
- `domain/exception/HasFutureAppointmentsException.java`
- `domain/port/in/CreateAppointmentUseCase.java`
- `domain/port/in/UpdateAppointmentUseCase.java`
- `domain/port/in/GetAppointmentUseCase.java`
- `domain/port/in/ListAppointmentsByDateUseCase.java`
- `domain/port/in/ChangeAppointmentStatusUseCase.java`
- `domain/port/out/AppointmentRepository.java`
- `domain/port/out/FinancialEntryRepository.java`

**Depende de:** —

**Done when:**
- `Appointment.java`: `@Getter @Builder @NoArgsConstructor @AllArgsConstructor`; campos `UUID id`, `UUID clientId`, `UUID serviceId`, `LocalDate scheduledDate`, `LocalTime scheduledTime`, `int durationMinutes`, `AppointmentStatus status`, `String notes`, `UUID financialEntryId` (nullable), `LocalDateTime createdAt`, `LocalDateTime updatedAt`
- `AppointmentStatus`: enum com valores `SCHEDULED`, `CONFIRMED`, `COMPLETED`, `CANCELLED`, `NO_SHOW`
- `FinancialEntry.java`: campos `UUID id`, `String type`, `String description`, `BigDecimal amount`, `LocalDate dueDate`, `String status`, `UUID appointmentId`, `String category`, `LocalDateTime createdAt`
- `AppointmentNotFoundException extends DomainException` com construtor `(UUID id)`
- `AppointmentConflictException extends DomainException` com mensagem `"Horário indisponível — já existe um agendamento neste período"`
- `HasFutureAppointmentsException extends DomainException` com campo `int count`; mensagem `"Esta {entidade} tem {n} agendamento(s) futuro(s)"`
- `CreateAppointmentUseCase` com `CreateAppointmentCommand(UUID clientId, UUID serviceId, LocalDate scheduledDate, LocalTime scheduledTime, int durationMinutes, String notes)` e `AppointmentResult(UUID id, UUID clientId, String clientName, UUID serviceId, String serviceName, BigDecimal servicePrice, String scheduledDate, String scheduledTime, int durationMinutes, String status, String notes, String financialEntryId, String createdAt)`
- `UpdateAppointmentUseCase` com `UpdateAppointmentCommand` igual ao create; retorna `AppointmentResult`
- `GetAppointmentUseCase.execute(UUID id)` retorna `AppointmentResult`
- `ListAppointmentsByDateUseCase.execute(LocalDate date)` retorna `List<AppointmentResult>`
- `ChangeAppointmentStatusUseCase` com métodos: `confirm(UUID id)`, `complete(UUID id)`, `cancel(UUID id)`, `noShow(UUID id)` — todos retornam `void`
- `ListAppointmentsByDateUseCase` com `execute(LocalDate date)` e `executeRange(LocalDate startDate, LocalDate endDate)`
- `AppointmentRepository (port/out)`:
  ```java
  Appointment save(Appointment a);
  Optional<Appointment> findById(UUID id);
  List<Appointment> findActiveByDate(LocalDate date);  // para conflito
  List<AppointmentResult> findByDateWithDetails(LocalDate date);  // enriquecido
  List<AppointmentResult> findByDateRangeWithDetails(LocalDate startDate, LocalDate endDate); // view mês
  long countFutureActiveByClientId(UUID clientId, LocalDate today);
  long countFutureActiveByServiceId(UUID serviceId, LocalDate today);
  ```
- `FinancialEntryRepository (port/out)`: apenas `FinancialEntry save(FinancialEntry entry)`

**Gate:** `mvn compile` sem erros

---

### T02 — Use Cases [P após T01]

**O que:** Implementar os 5 use cases e o mapper.

**Onde:**
- `application/usecase/CreateAppointmentUseCaseImpl.java`
- `application/usecase/UpdateAppointmentUseCaseImpl.java`
- `application/usecase/GetAppointmentUseCaseImpl.java`
- `application/usecase/ListAppointmentsByDateUseCaseImpl.java`
- `application/usecase/ChangeAppointmentStatusUseCaseImpl.java`
- `application/usecase/AppointmentUseCaseMapper.java`

**Depende de:** T01

**Done when:**
- `CreateAppointmentUseCaseImpl`:
  1. Carrega cliente via `ClientRepository.findById()` — lança `ClientNotFoundException` se não encontrado ou inativo (`!client.isActive()`)
  2. Carrega serviço via `ServiceRepository.findById()` — lança `ServiceNotFoundException` se não encontrado ou inativo
  3. Valida horário expediente: `scheduledTime >= 06:00` e `scheduledTime.plusMinutes(durationMinutes) <= 20:00` — lança `DomainException("Horário fora do expediente (06:00–20:00)")` se inválido
  4. Carrega agendamentos ativos do dia via `AppointmentRepository.findActiveByDate(date)`
  5. Verifica sobreposição: `a.getScheduledTime().isBefore(newEnd) && a.getScheduledTime().plusMinutes(a.getDurationMinutes()).isAfter(newStart)` — lança `AppointmentConflictException` se conflito
  6. Salva novo agendamento com status `SCHEDULED`
- `UpdateAppointmentUseCaseImpl`: busca por ID, valida status (não permite editar COMPLETED ou CANCELLED), revalida expediente e conflito (excluindo o próprio agendamento), salva
- `GetAppointmentUseCaseImpl`: busca por ID via `AppointmentRepository`, carrega `clientName` via `ClientRepository`, carrega `serviceName + servicePrice` via `ServiceRepository`, monta `AppointmentResult`
- `ListAppointmentsByDateUseCaseImpl`: delega para `AppointmentRepository.findByDateWithDetails(date)` e converte para `List<AppointmentResult>`
- `ChangeAppointmentStatusUseCaseImpl`:
  - `confirm(UUID id)`: SCHEDULED → CONFIRMED; lança exceção se status diferente
  - `complete(UUID id)`: `@Transactional`; CONFIRMED → COMPLETED; cria `FinancialEntry` via `FinancialEntryRepository.save()` com `type="INCOME"`, `description="{serviceName} — {clientName}"`, `amount=servicePrice`, `dueDate=scheduledDate`, `status="PENDING"`, `appointmentId=appointment.id`, `category="ATENDIMENTO"`; atualiza `appointment.financialEntryId` com o ID da entrada criada
  - `cancel(UUID id)`: SCHEDULED|CONFIRMED → CANCELLED; lança exceção se já COMPLETED, CANCELLED ou NO_SHOW
  - `noShow(UUID id)`: SCHEDULED|CONFIRMED → NO_SHOW
- `AppointmentUseCaseMapper.toResult(Appointment, String clientName, String serviceName, BigDecimal servicePrice)`

**Gate:** `mvn compile` sem erros

---

### T03 — Infrastructure [P após T01]

**O que:** Criar entidades JPA, mappers, repositórios Spring Data e implementações para Appointment e FinancialEntry.

**Onde:**
- `infrastructure/persistence/entity/AppointmentEntity.java`
- `infrastructure/persistence/entity/FinancialEntryEntity.java`
- `infrastructure/persistence/mapper/AppointmentMapper.java`
- `infrastructure/persistence/mapper/FinancialEntryMapper.java`
- `infrastructure/persistence/repository/AppointmentJpaRepository.java`
- `infrastructure/persistence/repository/AppointmentRepositoryImpl.java`
- `infrastructure/persistence/repository/FinancialEntryJpaRepository.java`
- `infrastructure/persistence/repository/FinancialEntryRepositoryImpl.java`

**Depende de:** T01

**Done when:**
- `AppointmentEntity`: `@Entity @Table(name = "appointments")`; `@ManyToOne(fetch = FetchType.LAZY)` para `ClientEntity` (campo `client`) e `ServiceEntity` (campo `service`); `status` como `String`; `scheduledDate` como `LocalDate`; `scheduledTime` como `LocalTime`; `financialEntryId` como `UUID` nullable
- `FinancialEntryEntity`: `@Entity @Table(name = "financial_entries")`; campos mapeados incluindo `appointmentId` como `UUID`
- `AppointmentJpaRepository` com queries:
  ```java
  @Query("SELECT a FROM AppointmentEntity a JOIN FETCH a.client JOIN FETCH a.service WHERE a.scheduledDate = :date ORDER BY a.scheduledTime")
  List<AppointmentEntity> findByDateWithDetails(@Param("date") LocalDate date);

  @Query("SELECT a FROM AppointmentEntity a WHERE a.scheduledDate = :date AND a.status != 'CANCELLED'")
  List<AppointmentEntity> findActiveByDate(@Param("date") LocalDate date);

  @Query("SELECT COUNT(a) FROM AppointmentEntity a WHERE a.client.id = :clientId AND a.scheduledDate > :today AND a.status IN ('SCHEDULED','CONFIRMED')")
  long countFutureActiveByClientId(@Param("clientId") UUID clientId, @Param("today") LocalDate today);

  @Query("SELECT COUNT(a) FROM AppointmentEntity a WHERE a.service.id = :serviceId AND a.scheduledDate > :today AND a.status IN ('SCHEDULED','CONFIRMED')")
  long countFutureActiveByServiceId(@Param("serviceId") UUID serviceId, @Param("today") LocalDate today);
  ```
- `AppointmentRepositoryImpl implements AppointmentRepository` — implementa todos os métodos do port/out; `findByDateWithDetails` usa os dados de `a.getClient()` e `a.getService()` (já carregados via JOIN FETCH) para montar `AppointmentResult`
- `FinancialEntryRepositoryImpl implements FinancialEntryRepository` — apenas `save()`

**Gate:** `mvn compile` sem erros

---

### T04 — Controller + DTOs

**O que:** Criar DTOs, controller REST e atualizar GlobalExceptionHandler.

**Onde:**
- `adapter/web/dto/CreateAppointmentRequest.java`
- `adapter/web/dto/UpdateAppointmentRequest.java`
- `adapter/web/dto/AppointmentResponse.java`
- `adapter/web/controller/AppointmentController.java`
- `adapter/web/controller/GlobalExceptionHandler.java` (atualizar)

**Depende de:** T02, T03

**Done when:**
- `CreateAppointmentRequest`: `@NotNull UUID clientId`, `@NotNull UUID serviceId`, `@NotNull LocalDate scheduledDate`, `@NotNull LocalTime scheduledTime`, `@NotNull @Min(1) Integer durationMinutes`, `String notes`
- `UpdateAppointmentRequest`: idêntico ao create
- `AppointmentResponse`: record com todos os campos de `AppointmentResult` (clientName, serviceName, servicePrice incluídos)
- `AppointmentController @RestController @RequestMapping("/api/appointments")` com 9 endpoints:
  - `GET /?date=YYYY-MM-DD` → lista do dia → 200
  - `GET /?date=YYYY-MM-DD&endDate=YYYY-MM-DD` → lista por range (view mês) → 200
  - `POST /` → criar → 201
  - `GET /:id` → detalhe → 200
  - `PUT /:id` → editar → 200
  - `PATCH /:id/confirm` → confirmar → 204
  - `PATCH /:id/complete` → marcar realizado → 204
  - `PATCH /:id/cancel` → cancelar → 204
  - `PATCH /:id/no-show` → marcar não compareceu → 204
- `GlobalExceptionHandler` com novos handlers:
  - `AppointmentNotFoundException` → 404
  - `AppointmentConflictException` → 409
  - `HasFutureAppointmentsException` → 409 com body incluindo campo `count`

**Gate:** `mvn test` passa; `mvn spring-boot:run` sobe; `GET /api/appointments?date=today` retorna lista vazia `[]`

---

### T05 — CLI-13 / SVC-13 [P após T03]

**O que:** Atualizar os use cases de desativação de Clientes e Serviços para verificar agendamentos futuros, com suporte ao parâmetro `force`.

**Onde:**
- `domain/port/in/DeactivateClientUseCase.java` (atualizar assinatura)
- `domain/port/in/DeactivateServiceUseCase.java` (atualizar assinatura)
- `application/usecase/DeactivateClientUseCaseImpl.java` (atualizar)
- `application/usecase/DeactivateServiceUseCaseImpl.java` (atualizar)
- `adapter/web/controller/ClientController.java` (atualizar endpoint deactivate)
- `adapter/web/controller/ServiceController.java` (atualizar endpoint deactivate)

**Depende de:** T03 (precisa de `AppointmentRepository` disponível para injeção)

**Done when:**
- `DeactivateClientUseCase.deactivate(UUID id, boolean force)` — assinatura atualizada
- `DeactivateClientUseCaseImpl.deactivate(UUID id, boolean force)`:
  - Injeta `AppointmentRepository` via construtor (adicionar ao `@RequiredArgsConstructor`)
  - Se `!force`: chama `appointmentRepository.countFutureActiveByClientId(id, LocalDate.now())`; se `count > 0` lança `HasFutureAppointmentsException("cliente", count)`
  - Se `force` (ou count == 0): desativa normalmente
- `DeactivateServiceUseCaseImpl` idem com `countFutureActiveByServiceId`
- `ClientController.deactivate(@PathVariable UUID id, @RequestParam(defaultValue = "false") boolean force)` → passa `force` ao use case
- `ServiceController.deactivate` idem

**Gate:** `mvn compile` sem erros; testar manualmente: `PATCH /api/clients/:id/deactivate` com cliente sem agendamentos → 204; com agendamentos futuros → 409 com `count`

---

### T06 — Frontend Model [P]

**O que:** Criar o model TypeScript para Agendamentos.

**Onde:** `src/app/core/models/appointment.model.ts`

**Depende de:** —

**Done when:**
- Tipos: `AppointmentStatus = 'SCHEDULED' | 'CONFIRMED' | 'COMPLETED' | 'CANCELLED'`
- Interface `Appointment` com todos os campos incluindo `clientName`, `serviceName`, `servicePrice`, `financialEntryId?`
- Interface `CreateAppointmentRequest`
- Interface `AppointmentState`: `appointments: Appointment[]`, `currentDate: string`, `selectedAppointment: Appointment | null`, `isLoading: boolean`, `isSaving: boolean`, `error: string | null`
- Interface `TimeSlot`: `time: string`, `appointment: Appointment | null`

**Gate:** `npm run build` sem erros de tipo

---

### T07 — AppointmentHttp

**O que:** Criar o serviço Angular de comunicação HTTP.

**Onde:** `src/app/features/appointments/services/appointment.service.ts`

**Depende de:** T06

**Done when:**
- `@Injectable({ providedIn: 'root' })` com `inject(HttpClient)`
- `baseUrl = '/api/appointments'`
- Métodos:
  - `listByDate(date: string): Observable<Appointment[]>`
  - `getById(id: string): Observable<Appointment>`
  - `create(req: CreateAppointmentRequest): Observable<Appointment>`
  - `update(id: string, req: CreateAppointmentRequest): Observable<Appointment>`
  - `confirm(id: string): Observable<void>`
  - `complete(id: string): Observable<void>`
  - `cancel(id: string): Observable<void>`
- `listByDate` passa `date` como query param; se `date` não fornecido usa hoje (`new Date().toISOString().split('T')[0]`)

**Gate:** `npm run build` sem erros de tipo

---

### T08 — NgRx Store

**O que:** Criar actions, reducer, selectors e effects.

**Onde:**
- `src/app/features/appointments/store/appointment.actions.ts`
- `src/app/features/appointments/store/appointment.reducer.ts`
- `src/app/features/appointments/store/appointment.selectors.ts`
- `src/app/features/appointments/store/appointment.effects.ts`

**Depende de:** T06, T07

**Done when:**
- **Actions** sob `AppointmentActions`:
  - `loadAppointments` / `loadAppointmentsSuccess` / `loadAppointmentsFailure`
  - `setDate` (muda currentDate e dispara loadAppointments)
  - `selectAppointment` / `selectAppointmentSuccess` / `selectAppointmentFailure` / `clearSelectedAppointment`
  - `createAppointment` / `createAppointmentSuccess` / `createAppointmentFailure`
  - `updateAppointment` / `updateAppointmentSuccess` / `updateAppointmentFailure`
  - `confirmAppointment` / `confirmAppointmentSuccess` / `confirmAppointmentFailure`
  - `completeAppointment` / `completeAppointmentSuccess` / `completeAppointmentFailure`
  - `cancelAppointment` / `cancelAppointmentSuccess` / `cancelAppointmentFailure`
  - `noShowAppointment` / `noShowAppointmentSuccess` / `noShowAppointmentFailure`
- **Reducer** `appointmentReducer` com slice key `appointments`; estado inicial com `currentDate: today`, `appointments: []`
- **Selectors**: `selectAppointmentsState`, `selectAllAppointments`, `selectCurrentDate`, `selectSelectedAppointment`, `selectAppointmentsLoading`, `selectAppointmentsSaving`, `selectAppointmentsError`
- **Effects**:
  - `loadAppointments$` — chama `appointmentService.listByDate(date)`
  - `setDate$` — ao receber `setDate`, despacha `loadAppointments` com nova data
  - `selectAppointment$` — chama `appointmentService.getById(id)`
  - `createAppointment$` — ao sucesso navega para `/appointments`
  - `updateAppointment$` — ao sucesso navega para `/appointments/:id`
  - `confirmAppointment$`, `completeAppointment$`, `cancelAppointment$`, `noShowAppointment$` — ao sucesso recarrega o agendamento selecionado (`selectAppointment`)

**Atualização de Client + Service Effects (CLI-13/SVC-13):**
- `ClientEffects.deactivateClient$`: ao receber erro 409 com `err.error?.count`, chama `confirm()` com mensagem incluindo o count; se confirmado, re-despacha `ClientActions.deactivateClient({ id, force: true })`
- Atualizar `ClientActions.deactivateClient` para aceitar `force?: boolean`
- `ClientService.deactivate(id, force = false)` passa `?force=true` como query param
- Mesmo padrão para `ServiceEffects` e `ServiceActions`

**Gate:** `npm run build` sem erros de tipo

---

### T09 — AppointmentDayComponent [P após T08] ✅ Redesenhado (rev 3)

**O que:** Componente de agenda estilo Google Calendar com views Dia / Semana / Mês, sidebar e dialogs.

**Onde:**
- `src/app/features/appointments/components/appointment-day/` (3 arquivos)
- `src/app/features/appointments/components/appointment-popup/` (3 arquivos — NOVO)
- `src/app/features/appointments/components/day-appointments-modal/` (3 arquivos — NOVO)

**Depende de:** `AppointmentService` (direto), `selectCurrentDate` (NgRx), `MatDialog`

**Implementado como:**

*Sidebar:*
- Mini calendário com navegação de mês independente (`miniMonth`, `buildMiniCal()`)
- Legenda colorida para 5 status (incluindo NO_SHOW)
- Filtro profissional via `<select>` nativo + `[(ngModel)]`

*Toggle de view:* `mat-button-toggle-group` com `setView(view: CalendarView)`

*View semana/dia:*
- Grade CSS `52px + repeat(7, minmax(72px, 1fr))` (semana) ou `52px 1fr` (dia)
- `forkJoin` carrega 7 dias em paralelo
- Blocos `position: absolute`, `top`/`height` calculados por horário/duração
- Click em coluna vazia → `/appointments/novo?date=&time=`

*View mês:*
- Grade 42 células (6×7), carregamento via `svc.listByDateRange(start, end)`
- Máximo 3 chips por célula; "+ N mais" abre `DayAppointmentsModalComponent`
- Click no chip abre `AppointmentPopupComponent`
- Click no número do dia → `switchToDayView(date)`

*AppointmentPopupComponent:*
- Dialog com borda esquerda colorida por status
- Exibe cliente, data, horário, serviço, duração, observações, badge de status
- Botões: editar (navega para form) e "Ver detalhes completos"

*DayAppointmentsModalComponent:*
- Lista todos agendamentos do dia; clique em item → fecha e abre AppointmentPopupComponent

**Gate:** `npm run build` sem erros de tipo

---

### T10 — AppointmentFormComponent [P após T08]

**O que:** Criar formulário de criação e edição de agendamento.

**Onde:** `src/app/features/appointments/components/appointment-form/appointment-form.component.ts`

**Depende de:** T08

**Done when:**
- Detecta modo create/edit via `:id` na rota; lê query params `date` e `time` para pré-preencher
- **Campos:**
  - Cliente: `mat-autocomplete` com busca por nome (debounce 300ms via `ClientService.list(search)`); exibe lista de clientes ativos; valor do form = `clientId`
  - Serviço: `mat-select` com serviços ativos (carregados via `ServiceService`); ao selecionar → preenche `durationMinutes` com `service.durationMinutes`
  - Data: `mat-datepicker`
  - Horário: `<input type="time">` pré-preenchido com query param `time` se disponível
  - Duração (minutos): number input editável
  - Observações: textarea
- Validators: `required` em cliente, serviço, data, horário, duração; `min(1)` em duração
- Submit: despacha `createAppointment` ou `updateAppointment`
- Erros inline por campo
- Erro de conflito (409 vindo do store) exibido como alert no topo do form
- Botão Cancelar volta para agenda do dia

**Gate:** `npm run build` sem erros de tipo

---

### T11 — AppointmentDetailComponent [P após T08]

**O que:** Criar o componente de detalhe do agendamento com botões de status.

**Onde:** `src/app/features/appointments/components/appointment-detail/appointment-detail.component.ts`

**Depende de:** T08, `DurationPipe`

**Done when:**
- Despacha `selectAppointment` no `ngOnInit` com `:id` da rota
- Exibe todos os campos: cliente (link para `/clients/:id`), serviço, data, horário início-fim, duração (`DurationPipe`), status (chip com cor), observações, data de criação
- **Botões por status:**
  - `SCHEDULED`: "Confirmar" (despacha `confirmAppointment`) + "Cancelar"
  - `CONFIRMED`: "Marcar como realizado" (despacha `completeAppointment`) + "Não compareceu" (`noShowAppointment`) + "Cancelar"
  - `COMPLETED`: sem botões de ação; exibe link "Ver no financeiro" se `financialEntryId` presente
  - `NO_SHOW`: sem botões
  - `CANCELLED`: sem botões
- "Confirmar" e "Marcar como realizado" pedem confirmação via `confirm()` do browser
- "Cancelar" pede confirmação via `confirm()`
- Botão "Editar" visível se status `SCHEDULED` ou `CONFIRMED` → navega para `/appointments/:id/editar`
- 404 com link para voltar se erro no estado

**Gate:** `npm run build` sem erros de tipo

---

### T12 — Integração Final

**O que:** Atualizar rotas, registrar no store e verificar fluxo completo.

**Onde:**
- `src/app/features/appointments/appointments.routes.ts`
- `src/app/app.config.ts`

**Depende de:** T09, T10, T11

**Done when:**
- `appointments.routes.ts`:
  ```typescript
  { path: '', component: AppointmentDayComponent },
  { path: 'novo', component: AppointmentFormComponent },
  { path: ':id', component: AppointmentDetailComponent },
  { path: ':id/editar', component: AppointmentFormComponent },
  ```
- `app.config.ts`:
  ```typescript
  provideStore({ auth: authReducer, clients: clientReducer, services: serviceReducer, appointments: appointmentReducer }),
  provideEffects([AuthEffects, ClientEffects, ServiceEffects, AppointmentEffects]),
  ```
- Fluxo completo funciona com backend rodando:
  1. Agenda do dia carrega com agendamentos do dia
  2. Click em slot vazio → form pré-preenchido
  3. Criar agendamento → aparece na timeline
  4. Confirmar → cor muda para dourado
  5. Marcar realizado → cor muda para verde
  6. `GET /api/appointments?date=hoje` confirma entrada criada

**Gate:** `npm start` sobe sem erros; fluxo completo funciona end-to-end

---

## Resumo

| Task | Escopo | Depende | Paralelo |
|---|---|---|---|
| T01 | Domain + ports (13 arquivos) | — | — |
| T02 | Use cases (6 arquivos) | T01 | [P] com T03 |
| T03 | Infrastructure (8 arquivos) | T01 | [P] com T02 |
| T04 | Controller + DTOs (5 arquivos) | T02, T03 | [P] com T05 |
| T05 | CLI-13/SVC-13 (6 arquivos atualizados) | T03 | [P] com T04 |
| T06 | Frontend model (1 arquivo) | — | [P] com T01 |
| T07 | HTTP service (1 arquivo) | T06 | — |
| T08 | NgRx store + update clients/services (4+updates) | T06, T07 | — |
| T09 | AppointmentDayComponent | T08 | [P] com T10, T11 |
| T10 | AppointmentFormComponent | T08 | [P] com T09, T11 |
| T11 | AppointmentDetailComponent | T08 | [P] com T09, T10 |
| T12 | Routes + config + integração | T09, T10, T11 | — |

**Total:** 12 tasks, ~37 arquivos novos, ~8 arquivos atualizados
