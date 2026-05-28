# Agendamentos — Design

**Atualizado:** 2026-05-22 (rev 3)

## Avaliação de Escopo

**Complex** — Envolve UI de timeline com click-to-create, lógica de conflito de horário, máquina de estados, integração financeira transacional, e resolução de CLI-13/SVC-13 com mudanças em módulos existentes.

---

## Decisões Arquiteturais

| Decisão | Escolha | Razão |
|---|---|---|
| Detecção de conflito | Lógica Java no use case (não JPQL) | Aritmética de tempo no JPQL/PostgreSQL é verbosa; carregar agendamentos do dia (≤28 slots) é trivial |
| Integração financeira | `@Transactional` em `ChangeAppointmentStatusUseCaseImpl.complete()` | Atomicidade: agendamento e entrada financeira mudam juntos ou nenhum muda |
| CLI-13 / SVC-13 | Parâmetro `?force=true` no endpoint de desativação | RESTful, permite frontend confirmar com usuário antes de forçar |
| AppointmentEntity | `@ManyToOne(LAZY)` para `ClientEntity` e `ServiceEntity` com JOIN FETCH nas queries | Evita N+1 sem carregar tudo desnecessariamente |
| AppointmentResult | Desnormalizado — inclui `clientName`, `serviceName`, `servicePrice` | Frontend não precisa de chamadas extras para montar a agenda |
| FinancialEntry (fase 1) | Modelo mínimo sem controller | O módulo Financeiro completo vem na Fase 2; aqui só precisamos de `save()` |
| Timeline UI | Grade semanal estilo Google Calendar — 7 colunas simultâneas, agendamentos como blocos `position: absolute` com `top` e `height` calculados por horário/duração | Usuária solicitou visão completa da semana; blocos absolutos são mais fiéis ao Google Calendar e permitem visualizar duração real do agendamento |
| Carregamento semanal | `forkJoin` com 7 chamadas paralelas a `listByDate()`, resultado como `Record<string, Appointment[]>` | 7 requisições independentes carregam em paralelo, sem necessidade de endpoint específico para semana |
| Formulário | Página separada com query params `?date=&time=` | Consistência com Clientes e Serviços; evita complexidade de MatDialog + NgRx |

---

## Backend

### Novos arquivos — Domain

```
domain/model/
├── Appointment.java           ← NOVO
└── FinancialEntry.java        ← NOVO (mínimo para save)

domain/exception/
├── AppointmentNotFoundException.java    ← NOVO
├── AppointmentConflictException.java    ← NOVO
└── HasFutureAppointmentsException.java  ← NOVO (CLI-13/SVC-13)

domain/port/in/
├── CreateAppointmentUseCase.java        ← NOVO
├── UpdateAppointmentUseCase.java        ← NOVO
├── GetAppointmentUseCase.java           ← NOVO
├── ListAppointmentsByDateUseCase.java   ← NOVO
└── ChangeAppointmentStatusUseCase.java  ← NOVO

domain/port/out/
├── AppointmentRepository.java           ← NOVO
└── FinancialEntryRepository.java        ← NOVO
```

**Appointment.java** — campos: `UUID id`, `UUID clientId`, `UUID serviceId`, `LocalDate scheduledDate`, `LocalTime scheduledTime`, `int durationMinutes`, `AppointmentStatus status` (enum), `String notes`, `UUID financialEntryId` (nullable), `LocalDateTime createdAt/updatedAt`

**AppointmentStatus** (enum): `SCHEDULED`, `CONFIRMED`, `COMPLETED`, `CANCELLED`, `NO_SHOW`

**FinancialEntry.java** (mínimo) — campos: `UUID id`, `String type`, `String description`, `BigDecimal amount`, `LocalDate dueDate`, `String status`, `UUID appointmentId`, `String category`, `LocalDateTime createdAt`

**HasFutureAppointmentsException** — `extends DomainException`; campo `int count`; mensagem: `"Esta {entidade} tem {n} agendamento(s) futuro(s)"`

---

### Novos arquivos — Application

```
application/usecase/
├── CreateAppointmentUseCaseImpl.java       ← NOVO
├── UpdateAppointmentUseCaseImpl.java       ← NOVO
├── GetAppointmentUseCaseImpl.java          ← NOVO
├── ListAppointmentsByDateUseCaseImpl.java  ← NOVO
├── ChangeAppointmentStatusUseCaseImpl.java ← NOVO (@Transactional no complete)
└── AppointmentUseCaseMapper.java           ← NOVO
```

**CreateAppointmentUseCaseImpl** — fluxo:
1. Carregar cliente (lançar `ClientNotFoundException` se não existir ou inativo)
2. Carregar serviço (lançar `ServiceNotFoundException` se não existir ou inativo)
3. Verificar horário de expediente: `scheduledTime >= 06:00` e `scheduledTime + durationMinutes <= 20:00`
4. Carregar todos agendamentos não-cancelados do dia via `AppointmentRepository.findByDate(date)`
5. Verificar conflito em Java: algum agendamento existente sobrepõe `[startTime, startTime+duration)`?
   ```java
   boolean conflicts = appointments.stream().anyMatch(a -> {
       LocalTime aEnd = a.getScheduledTime().plusMinutes(a.getDurationMinutes());
       LocalTime newEnd = command.scheduledTime().plusMinutes(command.durationMinutes());
       return a.getScheduledTime().isBefore(newEnd) && aEnd.isAfter(command.scheduledTime());
   });
   if (conflicts) throw new AppointmentConflictException();
   ```
6. Salvar com status `SCHEDULED`

**ChangeAppointmentStatusUseCaseImpl** — métodos:
- `confirm(UUID id)`: SCHEDULED → CONFIRMED
- `complete(UUID id)`: CONFIRMED → COMPLETED + `@Transactional` cria `FinancialEntry`
  - `FinancialEntry`: `type=INCOME`, `description="{serviceName} — {clientName}"`, `amount=service.price`, `dueDate=appointment.scheduledDate`, `status=PENDING`, `appointmentId=appointment.id`, `category="ATENDIMENTO"`
  - Injeta: `AppointmentRepository`, `ServiceRepository`, `ClientRepository`, `FinancialEntryRepository`
- `cancel(UUID id)`: SCHEDULED|CONFIRMED → CANCELLED (bloqueia se já NO_SHOW)
- `noShow(UUID id)`: SCHEDULED|CONFIRMED → NO_SHOW

**AppointmentResult** (record no `CreateAppointmentUseCase`):
```java
record AppointmentResult(
    UUID id, UUID clientId, String clientName,
    UUID serviceId, String serviceName, BigDecimal servicePrice,
    String scheduledDate, String scheduledTime, int durationMinutes,
    String status, String notes, String financialEntryId, String createdAt
)
```

---

### Novos arquivos — Infrastructure

```
infrastructure/persistence/
├── entity/
│   ├── AppointmentEntity.java    ← NOVO
│   └── FinancialEntryEntity.java ← NOVO
├── mapper/
│   ├── AppointmentMapper.java    ← NOVO
│   └── FinancialEntryMapper.java ← NOVO
└── repository/
    ├── AppointmentJpaRepository.java    ← NOVO
    ├── AppointmentRepositoryImpl.java   ← NOVO
    ├── FinancialEntryJpaRepository.java ← NOVO
    └── FinancialEntryRepositoryImpl.java← NOVO
```

**AppointmentEntity** — `@ManyToOne(fetch = FetchType.LAZY)` para `ClientEntity` e `ServiceEntity`. Status como `String` mapeado para coluna `VARCHAR(50)`.

**AppointmentJpaRepository** — queries:

```java
// Lista do dia com JOIN FETCH (evita N+1)
@Query("SELECT a FROM AppointmentEntity a JOIN FETCH a.client JOIN FETCH a.service WHERE a.scheduledDate = :date ORDER BY a.scheduledTime")
List<AppointmentEntity> findByDateWithDetails(@Param("date") LocalDate date);

// Simples para verificação de conflito
@Query("SELECT a FROM AppointmentEntity a WHERE a.scheduledDate = :date AND a.status != 'CANCELLED'")
List<AppointmentEntity> findActiveByDate(@Param("date") LocalDate date);

// Para CLI-13 / SVC-13
@Query("SELECT COUNT(a) FROM AppointmentEntity a WHERE a.client.id = :clientId AND a.scheduledDate > :today AND a.status IN ('SCHEDULED','CONFIRMED')")
long countFutureActiveByClientId(@Param("clientId") UUID clientId, @Param("today") LocalDate today);

@Query("SELECT COUNT(a) FROM AppointmentEntity a WHERE a.service.id = :serviceId AND a.scheduledDate > :today AND a.status IN ('SCHEDULED','CONFIRMED')")
long countFutureActiveByServiceId(@Param("serviceId") UUID serviceId, @Param("today") LocalDate today);
```

**AppointmentRepository (port/out)**:
```java
Appointment save(Appointment appointment);
Optional<Appointment> findById(UUID id);
List<Appointment> findByDate(LocalDate date);           // para conflito (sem JOIN FETCH)
List<AppointmentResult> findByDateWithDetails(LocalDate date); // para lista enriquecida
List<AppointmentResult> findByDateRangeWithDetails(LocalDate startDate, LocalDate endDate); // range para view mês
long countFutureActiveByClientId(UUID clientId, LocalDate today);
long countFutureActiveByServiceId(UUID serviceId, LocalDate today);
```

**FinancialEntryRepository (port/out)**:
```java
FinancialEntry save(FinancialEntry entry);
```

---

### Novos arquivos — Adapter

```
adapter/web/
├── dto/
│   ├── CreateAppointmentRequest.java   ← NOVO
│   ├── UpdateAppointmentRequest.java   ← NOVO
│   └── AppointmentResponse.java        ← NOVO
└── controller/
    └── AppointmentController.java      ← NOVO
```

**API Contract:**

| Método | Endpoint | Descrição | HTTP |
|---|---|---|---|
| `GET` | `/api/appointments?date=YYYY-MM-DD` | Lista do dia com detalhes | 200 |
| `GET` | `/api/appointments?date=YYYY-MM-DD&endDate=YYYY-MM-DD` | Lista por intervalo (view mês) | 200 |
| `POST` | `/api/appointments` | Criar agendamento | 201 |
| `GET` | `/api/appointments/:id` | Detalhe | 200 |
| `PUT` | `/api/appointments/:id` | Editar (data/hora/duração/obs) | 200 |
| `PATCH` | `/api/appointments/:id/confirm` | Confirmar | 204 |
| `PATCH` | `/api/appointments/:id/complete` | Marcar realizado + gerar financeiro | 204 |
| `PATCH` | `/api/appointments/:id/cancel` | Cancelar | 204 |
| `PATCH` | `/api/appointments/:id/no-show` | Marcar não compareceu | 204 |

**CreateAppointmentRequest**:
```java
record CreateAppointmentRequest(
    @NotNull UUID clientId,
    @NotNull UUID serviceId,
    @NotNull LocalDate scheduledDate,
    @NotNull LocalTime scheduledTime,
    @NotNull @Min(1) Integer durationMinutes,
    String notes
)
```

---

### Arquivos atualizados

```
application/usecase/DeactivateClientUseCaseImpl.java   ← ATUALIZAR (CLI-13)
application/usecase/DeactivateServiceUseCaseImpl.java  ← ATUALIZAR (SVC-13)
adapter/web/controller/GlobalExceptionHandler.java     ← ATUALIZAR (novos handlers)
```

**CLI-13 / SVC-13 — implementação:**

`DeactivateClientUseCaseImpl.deactivate(UUID id)` passa a aceitar `force: boolean`:
```java
public void deactivate(UUID id) {
    // mesma assinatura — adiciona verificação:
    long count = appointmentRepository.countFutureActiveByClientId(id, LocalDate.now());
    if (count > 0) throw new HasFutureAppointmentsException("cliente", (int) count);
    clientRepository.save(withActive(client, false));
}
```

No controller: `PATCH /api/clients/:id/deactivate?force=false` — se `force=true`, não lança a exceção.

**GlobalExceptionHandler** — adicionar:
- `AppointmentNotFoundException` → 404
- `AppointmentConflictException` → 409 `"Horário indisponível — já existe um agendamento neste período"`
- `HasFutureAppointmentsException` → 409 com campo `count` no body

---

## Frontend

### Estrutura de arquivos

```
lash-frontend/src/app/
├── core/models/
│   └── appointment.model.ts              ← NOVO
├── features/appointments/
│   ├── appointments.routes.ts            ← ATUALIZAR (substituir placeholder)
│   ├── services/
│   │   └── appointment.service.ts        ← NOVO
│   ├── store/
│   │   ├── appointment.actions.ts        ← NOVO
│   │   ├── appointment.reducer.ts        ← NOVO
│   │   ├── appointment.effects.ts        ← NOVO
│   │   └── appointment.selectors.ts      ← NOVO
│   └── components/
│       ├── appointment-day/              ← 3 arquivos (.ts + .html + .css)
│       ├── appointment-form/             ← 3 arquivos
│       ├── appointment-detail/           ← 3 arquivos
│       ├── appointment-popup/            ← NOVO — dialog estilo Google Calendar
│       └── day-appointments-modal/       ← NOVO — lista agendamentos do dia
└── app.config.ts                         ← ATUALIZAR
```

---

### appointment.model.ts

```typescript
export type AppointmentStatus = 'SCHEDULED' | 'CONFIRMED' | 'COMPLETED' | 'CANCELLED' | 'NO_SHOW';

export interface Appointment {
  id: string;
  clientId: string;
  clientName: string;
  serviceId: string;
  serviceName: string;
  servicePrice: number;
  scheduledDate: string;   // "YYYY-MM-DD"
  scheduledTime: string;   // "HH:mm:ss"
  durationMinutes: number;
  status: AppointmentStatus;
  notes?: string;
  financialEntryId?: string;
  createdAt: string;
}

export interface CreateAppointmentRequest {
  clientId: string;
  serviceId: string;
  scheduledDate: string;
  scheduledTime: string;
  durationMinutes: number;
  notes?: string;
}

export interface AppointmentState {
  appointments: Appointment[];   // agendamentos da data atual
  currentDate: string;           // "YYYY-MM-DD"
  selectedAppointment: Appointment | null;
  isLoading: boolean;
  isSaving: boolean;
  error: string | null;
}

// Slot de 30 minutos para a timeline
export interface TimeSlot {
  time: string;        // "06:00", "06:30", ...
  appointment: Appointment | null;
}
```

---

### AppointmentPopupComponent (NOVO)

Dialog estilo Google Calendar para exibir detalhes de um agendamento. Aberto quando o usuário clica em um chip na view mês.

```typescript
export interface AppointmentPopupData { appointment: Appointment; }

@Component({ standalone: true, imports: [CommonModule, RouterLink, MatButtonModule, MatIconModule, MatDialogModule, DurationPipe] })
export class AppointmentPopupComponent {
  dialogRef = inject(MatDialogRef<AppointmentPopupComponent>);
  data: AppointmentPopupData = inject(MAT_DIALOG_DATA);
  // endTime(), formatDate(), viewDetails() → navega para detalhe, edit() → navega para edição
}
```

- Borda esquerda colorida por status (`.status-border-SCHEDULED`, etc.)
- Toolbar com ícone de edição e botão fechar
- Exibe: cliente, data, horário início–fim, serviço, duração, status (badge), observações
- Botão "Ver detalhes completos" → fecha dialog + navega para `/appointments/:id`

**panelClass:** `appt-popup-panel` (estilos globais em `styles.scss` removem padding do container)

---

### DayAppointmentsModalComponent (NOVO)

Modal com lista de todos os agendamentos de um dia específico. Aberta ao clicar em "+ N mais" na view mês.

```typescript
export interface DayAppointmentsModalData { date: string; appointments: Appointment[]; }

@Component({ standalone: true, imports: [CommonModule, MatButtonModule, MatIconModule, MatDialogModule] })
export class DayAppointmentsModalComponent {
  dialogRef = inject(MatDialogRef<DayAppointmentsModalComponent>);
  data: DayAppointmentsModalData = inject(MAT_DIALOG_DATA);
  private dialog = inject(MatDialog);
  openAppointment(appt): void; // fecha este modal e abre AppointmentPopupComponent
}
```

- Header com nome + número do dia (grande, estilo Google)
- Lista scrollável com dot colorido, horário, cliente, serviço
- Clicar em um item: fecha este modal e abre `AppointmentPopupComponent`

**panelClass:** `day-modal-panel`

---

### AppointmentDayComponent — Vista Completa (Dia / Semana / Mês)

**Sidebar esquerda:**
- Mini calendário de navegação (`miniMonth`, `miniCalDays: MiniCalDay[]`, `buildMiniCal()`)
- Legenda de cores: SCHEDULED, CONFIRMED, COMPLETED, NO_SHOW, CANCELLED
- Filtro de profissional: `<select>` nativo com `[(ngModel)]`

**Toggle de view:** `mat-button-toggle-group` — `[value]="currentView" (change)="setView($event.value)"`

**Navegação:** `navigate(dir: -1|1)` — ±1 dia, ±7 dias ou ±1 mês conforme view ativa

**View mês:**
- Grade 42 células (6 sem × 7 dias), começa na segunda da semana do dia 1
- Carregamento via `svc.listByDateRange(startDate, endDate)` (uma única requisição)
- Chips coloridos com fundo sólido (`status-chip-*`); máximo 3 por célula
- "+ N mais" → `openDayModal(date, event)` → `DayAppointmentsModalComponent`
- Chip click → `openAppointmentPopup(appt, event)` → `AppointmentPopupComponent`
- Número do dia click → `switchToDayView(date)`

**Constantes de layout:**
```typescript
readonly SLOT_H = 48;    // px por slot de 30 min
readonly START = 6 * 60; // 06:00 em minutos
readonly slots: string[] = Array.from({ length: 28 }, (_, i) => {
  const min = 6 * 60 + i * 30;
  return `${Math.floor(min/60).toString().padStart(2,'0')}:${(min%60).toString().padStart(2,'0')}`;
}); // ["06:00", "06:30", ..., "19:30"]
```

**Carregamento semanal (forkJoin):**
```typescript
private loadWeek(dateStr: string): void {
  const days = this.buildWeekDays(dateStr); // Seg a Dom da semana contendo dateStr
  this.weekLoading = true;
  const reqs: Record<string, Observable<Appointment[]>> = {};
  days.forEach(d => { reqs[d.date] = this.svc.listByDate(d.date); });
  forkJoin(reqs).subscribe({
    next: r => { this.weekData = r; this.weekLoading = false; },
    error: () => { this.weekLoading = false; }
  });
}
```

**Posicionamento absoluto dos agendamentos:**
```typescript
apptTop(a: Appointment): number {
  const [h, m] = a.scheduledTime.split(':').map(Number);
  return ((h * 60 + m - this.START) / 30) * this.SLOT_H;
}

apptHeight(a: Appointment): number {
  return Math.max((a.durationMinutes / 30) * this.SLOT_H - 3, 20);
}
```

**Click em coluna vazia — calcula horário pelo offset Y:**
```typescript
onColClick(event: MouseEvent, date: string): void {
  if ((event.target as HTMLElement).closest('.appt-block')) return;
  const col = event.currentTarget as HTMLElement;
  const y = event.clientY - col.getBoundingClientRect().top;
  const slotIdx = Math.max(0, Math.min(Math.floor(y / this.SLOT_H), this.slots.length - 1));
  this.router.navigate(['/appointments/novo'], { queryParams: { date, time: this.slots[slotIdx] } });
}
```

**Layout CSS:**
```
.agenda-view (flex column, height: calc(100vh - 96px), background: #fff)
  .week-nav (flex, border-bottom)
  .calendar-scroll (flex 1, overflow auto)
    .headers-row (grid: 56px + repeat(7, minmax(80px,1fr)), sticky top, z-index 20)
      .gutter-corner + 7× .day-head
    .calendar-body (grid: mesmo template)
      .time-col (sticky left, z-index 10)
        28× .time-cell (height: 48px)
      7× .day-col (position relative, min-height: 1344px, cursor pointer)
        28× .slot-bg (height: 48px, fundo/linhas)
        N× .appt-block (position absolute, top/height dinâmicos)
```

**Cores por status — blocos semana/dia:**
```css
.status-SCHEDULED { background: #E3F2FD; border-left-color: #1976D2; }
.status-CONFIRMED { background: #E0F7FA; border-left-color: #00838F; }
.status-COMPLETED { background: #E8F5E9; border-left-color: #388E3C; }
.status-NO_SHOW   { background: #FFF8E1; border-left-color: #F59E0B; opacity: 0.85; }
.status-CANCELLED { background: #FFEBEE; border-left-color: #E53935; opacity: 0.6; }
```

**Cores por status — chips mês (fundo sólido, texto branco):**
```css
.status-chip-SCHEDULED { background: #1976D2; color: #fff; }
.status-chip-CONFIRMED { background: #00838F; color: #fff; }
.status-chip-COMPLETED { background: #388E3C; color: #fff; }
.status-chip-NO_SHOW   { background: #F59E0B; color: #fff; }
.status-chip-CANCELLED { background: #e0e0e0; color: #888; }
```

---

### AppointmentFormComponent

- Detecta modo create/edit via `:id` na rota
- Lê query params `date` e `time` para pré-preencher (modo create)
- **Cliente**: `mat-autocomplete` com busca por nome (300ms debounce) — usa `/api/clients?search=`
- **Serviço**: `mat-select` com lista de serviços ativos — ao selecionar, preenche `durationMinutes`
- **Data**: `mat-datepicker`
- **Horário**: `input type="time"` com step de 30 min
- **Duração**: number input editável (pré-preenchido pelo serviço, pode ser ajustado)

---

### NgRx State

**Actions principais:**
```typescript
AppointmentActions = {
  loadAppointments: props<{ date: string }>(),
  setDate: props<{ date: string }>(),          // muda data e recarrega
  selectAppointment: props<{ id: string }>(),
  createAppointment: props<{ request: CreateAppointmentRequest }>(),
  updateAppointment: props<{ id: string; request: CreateAppointmentRequest }>(),
  confirmAppointment: props<{ id: string }>(),
  completeAppointment: props<{ id: string }>(),
  cancelAppointment: props<{ id: string }>(),
  // + Success/Failure variants
}
```

**Efeitos com navegação:**
- `createAppointmentSuccess$` → navega para `/appointments` (dia do agendamento criado)
- `completeAppointmentSuccess$` → recarrega `loadAppointments` (atualiza cor na agenda)
- `setDate$` → despacha `loadAppointments` com nova data

---

### Rotas

```typescript
export const appointmentsRoutes: Routes = [
  { path: '', component: AppointmentDayComponent },
  { path: 'novo', component: AppointmentFormComponent },
  { path: ':id', component: AppointmentDetailComponent },
  { path: ':id/editar', component: AppointmentFormComponent },
];
```

---

## Rastreabilidade

| Req ID | Componente responsável |
|---|---|
| AGD-01, AGD-05 | `AppointmentDayComponent` + CSS por status |
| AGD-02 | `AppointmentDayComponent` — navegação de data |
| AGD-03 | `AppointmentDayComponent.onSlotClick()` |
| AGD-04 | `AppointmentDayComponent` — botão "Novo agendamento" |
| AGD-06, AGD-07 | `AppointmentFormComponent` |
| AGD-08, AGD-09 | `CreateAppointmentUseCaseImpl` — validação de conflito e expediente |
| AGD-10 | `AppointmentDetailComponent` |
| AGD-11 | `ChangeAppointmentStatusUseCaseImpl.confirm()` |
| AGD-12 | `ChangeAppointmentStatusUseCaseImpl.complete()` + `FinancialEntryRepositoryImpl` |
| AGD-13 | `ChangeAppointmentStatusUseCaseImpl.cancel()` |
| AGD-14 | `AppointmentFormComponent` + `UpdateAppointmentUseCaseImpl` |
| AGD-15 | `DeactivateClientUseCaseImpl` (atualizado) + `AppointmentRepository.countFutureActiveByClientId` |
| AGD-16 | `DeactivateServiceUseCaseImpl` (atualizado) + `AppointmentRepository.countFutureActiveByServiceId` |
