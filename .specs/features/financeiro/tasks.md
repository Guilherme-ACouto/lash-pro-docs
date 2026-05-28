# Tasks — Módulo Financeiro

**Data:** 2026-05-23
**Status:** Pronto para implementação

Legenda: `[P]` = pode rodar em paralelo com outras tasks do mesmo grupo

---

## Grupo 1 — Backend: Domain + Ports (paralelo)

### TASK-01 [P] — Enum `FinancialEntryExpenseType`
**O quê:** Criar o enum de subcategoria de despesa.
**Onde:** `domain/model/FinancialEntryExpenseType.java`
```java
public enum FinancialEntryExpenseType {
    FIXED, VARIABLE, PEOPLE, TAX, TRANSFER
}
```
**Depende de:** nada
**Pronto quando:** enum compila

---

### TASK-02 [P] — Extender `FinancialEntry` domain model
**O quê:** Adicionar os 3 novos campos ao model existente.
**Onde:** `domain/model/FinancialEntry.java`
**Campos a adicionar:**
```java
private String paymentMethod;
private FinancialEntryExpenseType expenseType;
private String receivedFrom;
```
**Depende de:** TASK-01
**Pronto quando:** model compila; `FinancialEntryMapper` existente não quebra (campos nullable)

---

### TASK-03 [P] — Record `MonthlyFinancialStat`
**O quê:** Record de dado para a série do gráfico de 6 meses.
**Onde:** `domain/model/MonthlyFinancialStat.java`
```java
public record MonthlyFinancialStat(
    int year, int month,
    BigDecimal incomeTotal,
    BigDecimal expenseTotal
) {}
```
**Depende de:** nada
**Pronto quando:** record compila

---

### TASK-04 [P] — Exceções de domínio
**O quê:** Criar as duas exceções específicas do módulo financeiro.
**Onde:** `domain/exception/`
- `FinancialEntryNotFoundException.java` — estende `RuntimeException`; mensagem: `"Lançamento financeiro não encontrado: " + id`
- `FinancialEntryLinkedToAppointmentException.java` — estende `RuntimeException`; mensagem fixa: `"Não é possível excluir lançamento vinculado a agendamento"`

Registrar no `GlobalExceptionHandler`:
- `FinancialEntryNotFoundException` → 404
- `FinancialEntryLinkedToAppointmentException` → 409

**Depende de:** nada
**Pronto quando:** exceções compilam; handlers registrados

---

### TASK-05 [P] — Ports/in: 6 interfaces de use case
**O quê:** Criar as interfaces dos 6 use cases, cada uma com seu `Command`/`Query` record interno.
**Onde:** `domain/port/in/`

**`GetFinancialSummaryUseCase.java`:**
```java
public interface GetFinancialSummaryUseCase {
    record SummaryResult(
        BigDecimal predictedMonthResult, BigDecimal currentBalance,
        BigDecimal incomeReceived, BigDecimal incomePredicted,
        BigDecimal expensePaid, BigDecimal expensePredicted,
        List<MonthlyFinancialStat> monthlySeries
    ) {}
    SummaryResult execute();
}
```

**`ListFinancialEntriesUseCase.java`:**
```java
public interface ListFinancialEntriesUseCase {
    record ListQuery(LocalDate from, LocalDate to, String category,
                     String expenseType, String type, int page, int size) {}
    record EntryResult(UUID id, String type, String expenseType,
                       String description, BigDecimal amount,
                       LocalDate dueDate, LocalDate paymentDate,
                       String status, String category, String paymentMethod,
                       String counterpart, String notes,
                       boolean linkedToAppointment, UUID appointmentId) {}
    Page<EntryResult> execute(ListQuery query);
}
```

**`CreateFinancialEntryUseCase.java`:**
```java
public interface CreateFinancialEntryUseCase {
    record CreateCommand(String type, String expenseType, String description,
                         BigDecimal amount, LocalDate dueDate, LocalDate paymentDate,
                         String category, String paymentMethod, String receivedFrom,
                         String notes) {}
    ListFinancialEntriesUseCase.EntryResult execute(CreateCommand command);
}
```

**`UpdateFinancialEntryUseCase.java`:**
```java
public interface UpdateFinancialEntryUseCase {
    record UpdateCommand(UUID id, String description, BigDecimal amount,
                         LocalDate dueDate, LocalDate paymentDate,
                         String category, String expenseType,
                         String paymentMethod, String receivedFrom, String notes) {}
    ListFinancialEntriesUseCase.EntryResult execute(UpdateCommand command);
}
```

**`DeleteFinancialEntryUseCase.java`:**
```java
public interface DeleteFinancialEntryUseCase {
    void execute(UUID id);
}
```

**`ToggleFinancialEntryPaidUseCase.java`:**
```java
public interface ToggleFinancialEntryPaidUseCase {
    ListFinancialEntriesUseCase.EntryResult execute(UUID id);
}
```

**Depende de:** TASK-01, TASK-03
**Pronto quando:** todas as interfaces compilam

---

### TASK-06 [P] — Ports/out: `FinancialEntryRepository` (estender) + `FinancialSummaryRepository` (novo)
**O quê:** Estender a interface do repositório CRUD existente e criar a de agregação.
**Onde:** `domain/port/out/`

**`FinancialEntryRepository.java`** — adicionar aos métodos existentes:
```java
Optional<FinancialEntry> findById(UUID id);
Page<FinancialEntryWithCounterpart> listWithFilters(
    LocalDate from, LocalDate to, String category,
    String expenseType, String type, Pageable pageable);
void delete(UUID id);
boolean existsByIdAndAppointmentIdIsNull(UUID id);

record FinancialEntryWithCounterpart(FinancialEntry entry, String counterpart) {}
```

**`FinancialSummaryRepository.java`** (novo):
```java
public interface FinancialSummaryRepository {
    BigDecimal sumIncomePaidInMonth(LocalDate monthStart, LocalDate monthEnd);
    BigDecimal sumIncomeTotalInMonth(LocalDate monthStart, LocalDate monthEnd);
    BigDecimal sumExpensePaidInMonth(LocalDate monthStart, LocalDate monthEnd);
    BigDecimal sumExpenseTotalInMonth(LocalDate monthStart, LocalDate monthEnd);
    BigDecimal sumAllTimePaidBalance();
    List<MonthlyFinancialStat> last6MonthsStats();
}
```

**Depende de:** TASK-01, TASK-02, TASK-03
**Pronto quando:** interfaces compilam

---

## Grupo 2 — Backend: Infraestrutura

### TASK-07 — V5 Migration
**O quê:** Script Flyway com as 3 novas colunas.
**Onde:** `resources/db/migration/V5__financial_entries_new_fields.sql`
```sql
ALTER TABLE financial_entries
  ADD COLUMN payment_method VARCHAR(50),
  ADD COLUMN expense_type   VARCHAR(20),
  ADD COLUMN received_from  VARCHAR(255);

ALTER TABLE financial_entries
  ADD CONSTRAINT chk_expense_type
  CHECK (expense_type IN ('FIXED', 'VARIABLE', 'PEOPLE', 'TAX', 'TRANSFER'));
```
**Depende de:** nada
**Pronto quando:** backend sobe sem erro de Flyway

---

### TASK-08 [P] — Atualizar `FinancialEntryEntity`
**O quê:** Adicionar 3 campos + associação read-only para AppointmentEntity.
**Onde:** `infrastructure/persistence/entity/FinancialEntryEntity.java`
**Adicionar:**
```java
@Column(name = "payment_method")
private String paymentMethod;

@Column(name = "expense_type")
private String expenseType;

@Column(name = "received_from")
private String receivedFrom;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "appointment_id", insertable = false, updatable = false)
private AppointmentEntity appointment;
```
> O campo `appointmentId UUID` existente permanece dono do FK; `appointment` é só para leitura em JOIN.

**Depende de:** TASK-07
**Pronto quando:** entity compila; backend sobe sem erro JPA

---

### TASK-09 [P] — Atualizar `FinancialEntryMapper`
**O quê:** Adicionar mapeamento dos 3 novos campos (Entity ↔ Domain).
**Onde:** `infrastructure/persistence/mapper/FinancialEntryMapper.java`
- `toEntity`: mapear `paymentMethod`, `expenseType.name()` (null-safe), `receivedFrom`
- `toDomain`: mapear `paymentMethod`, `FinancialEntryExpenseType.valueOf(expenseType)` (null-safe), `receivedFrom`

**Depende de:** TASK-02, TASK-08
**Pronto quando:** mapper compila; testes de round-trip entity↔domain passam (mental ou real)

---

### TASK-10 [P] — Atualizar `FinancialEntryJpaRepository`
**O quê:** Adicionar método de query com filtros e JOINs.
**Onde:** `infrastructure/persistence/repository/FinancialEntryJpaRepository.java`
```java
@Query("""
    SELECT f FROM FinancialEntryEntity f
    LEFT JOIN FETCH f.appointment a
    LEFT JOIN FETCH a.client c
    WHERE (:type IS NULL OR f.type = :type)
    AND (:expenseType IS NULL OR f.expenseType = :expenseType)
    AND f.dueDate >= :from AND f.dueDate <= :to
    AND (:category IS NULL OR f.category = :category)
    ORDER BY f.dueDate DESC
    """)
Page<FinancialEntryEntity> findWithFilters(
    @Param("from") LocalDate from,
    @Param("to") LocalDate to,
    @Param("category") String category,
    @Param("type") String type,
    @Param("expenseType") String expenseType,
    Pageable pageable);

boolean existsByIdAndAppointmentIdIsNull(UUID id);
```
**Depende de:** TASK-08
**Pronto quando:** interface compila

---

### TASK-11 — Estender `FinancialEntryRepositoryImpl`
**O quê:** Implementar os novos métodos do port/out.
**Onde:** `infrastructure/persistence/repository/FinancialEntryRepositoryImpl.java`
**Métodos a adicionar:**
- `findById(UUID id)` → `jpaRepository.findById(id).map(mapper::toDomain)`
- `listWithFilters(...)` → chama `jpaRepository.findWithFilters(...)`, mapeia cada entity resolvendo `counterpart`:
  - se `entity.getAppointment() != null && entity.getAppointment().getClient() != null` → `client.getName()`
  - senão → `entity.getReceivedFrom()`
  - retorna `Page<FinancialEntryWithCounterpart>`
- `delete(UUID id)` → `jpaRepository.deleteById(id)`
- `existsByIdAndAppointmentIdIsNull(UUID id)` → `jpaRepository.existsByIdAndAppointmentIdIsNull(id)`

**Depende de:** TASK-09, TASK-10
**Pronto quando:** implementação compila; `save` existente não quebra

---

### TASK-12 — `FinancialSummaryRepositoryImpl`
**O quê:** Implementação das queries de agregação via EntityManager.
**Onde:** `infrastructure/persistence/repository/FinancialSummaryRepositoryImpl.java`
**Detalhes:** `@Repository @RequiredArgsConstructor`, injetar `EntityManager em`.

Queries JPQL:
```java
// sumIncomePaidInMonth
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'INCOME' AND f.status = 'PAID'
AND f.paymentDate >= :start AND f.paymentDate <= :end

// sumIncomeTotalInMonth (PAID + PENDING)
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'INCOME' AND f.status IN ('PAID', 'PENDING')
AND f.dueDate >= :start AND f.dueDate <= :end

// sumExpensePaidInMonth
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'EXPENSE' AND f.status = 'PAID'
AND f.paymentDate >= :start AND f.paymentDate <= :end

// sumExpenseTotalInMonth (PAID + PENDING)
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'EXPENSE' AND f.status IN ('PAID', 'PENDING')
AND f.dueDate >= :start AND f.dueDate <= :end

// sumAllTimePaidBalance
SELECT COALESCE(SUM(CASE WHEN f.type = 'INCOME' THEN f.amount ELSE -f.amount END), 0)
FROM FinancialEntryEntity f WHERE f.status = 'PAID'

// last6MonthsStats — nativa com DATE_TRUNC ou EXTRACT
SELECT EXTRACT(YEAR FROM f.paymentDate), EXTRACT(MONTH FROM f.paymentDate),
       f.type, COALESCE(SUM(f.amount), 0)
FROM FinancialEntryEntity f
WHERE f.status = 'PAID' AND f.paymentDate >= :sixMonthsAgo
GROUP BY EXTRACT(YEAR FROM f.paymentDate), EXTRACT(MONTH FROM f.paymentDate), f.type
ORDER BY 1, 2
```
Para `last6MonthsStats`: agrupa resultado em Map por `(year, month)`, constrói lista de `MonthlyFinancialStat` com os 6 meses (preencher meses sem lançamento com zero).

**Depende de:** TASK-06
**Pronto quando:** implementação compila; bean registrado no contexto Spring

---

## Grupo 3 — Backend: Use Cases (paralelo)

### TASK-13 [P] — `CreateFinancialEntryUseCaseImpl`
**O quê:** Use case de criação de lançamento manual.
**Onde:** `application/usecase/CreateFinancialEntryUseCaseImpl.java`
**Lógica:**
- Se `command.paymentDate() != null` → `status = PAID`; senão → `status = PENDING`
- Construir `FinancialEntry` via builder com todos os campos do command
- Salvar via `FinancialEntryRepository.save()`
- Retornar `EntryResult` com `counterpart = command.receivedFrom()`

**Depende de:** TASK-05, TASK-11
**Pronto quando:** compila; Spring injeta sem erros

---

### TASK-14 [P] — `UpdateFinancialEntryUseCaseImpl`
**O quê:** Use case de edição de lançamento.
**Onde:** `application/usecase/UpdateFinancialEntryUseCaseImpl.java`
**Lógica:**
- Carrega via `FinancialEntryRepository.findById(id)` → lança `FinancialEntryNotFoundException` se ausente
- Se `entry.appointmentId != null`: permite editar somente `paymentMethod`, `paymentDate`, `status`, `notes`; ignorar outros campos do command
- Se `entry.appointmentId == null`: editar todos os campos
- Regra de `paymentDate → status`: mesmo comportamento do Create
- Salvar e retornar `EntryResult`

**Depende de:** TASK-05, TASK-11
**Pronto quando:** compila

---

### TASK-15 [P] — `DeleteFinancialEntryUseCaseImpl`
**O quê:** Use case de exclusão.
**Onde:** `application/usecase/DeleteFinancialEntryUseCaseImpl.java`
**Lógica:**
- `existsByIdAndAppointmentIdIsNull(id)`:
  - `false` (vinculado) → lança `FinancialEntryLinkedToAppointmentException`
  - `true` → `FinancialEntryRepository.delete(id)`
- Verificar também se o ID existe (`findById`) → `FinancialEntryNotFoundException` se ausente

**Depende de:** TASK-04, TASK-05, TASK-11
**Pronto quando:** compila

---

### TASK-16 [P] — `ToggleFinancialEntryPaidUseCaseImpl`
**O quê:** Use case de alternância de status PENDING ↔ PAID.
**Onde:** `application/usecase/ToggleFinancialEntryPaidUseCaseImpl.java`
**Lógica:**
- Carrega entry; lança `FinancialEntryNotFoundException` se ausente
- `OVERDUE` → lança `IllegalStateException("Lançamento vencido não pode ser alternado via toggle")`
- `PENDING` → `status = PAID`, `paymentDate = LocalDate.now()`
- `PAID` → `status = PENDING`, `paymentDate = null`
- Salva e retorna `EntryResult`

**Depende de:** TASK-04, TASK-05, TASK-11
**Pronto quando:** compila

---

### TASK-17 [P] — `ListFinancialEntriesUseCaseImpl`
**O quê:** Use case de listagem paginada com filtros.
**Onde:** `application/usecase/ListFinancialEntriesUseCaseImpl.java`
**Lógica:**
- Normalizar: `category = ""` → `null` (Hibernate 6)
- Chamar `FinancialEntryRepository.listWithFilters(from, to, category, expenseType, type, pageable)`
- Mapear cada `FinancialEntryWithCounterpart` → `EntryResult`

**Depende de:** TASK-05, TASK-11
**Pronto quando:** compila

---

### TASK-18 — `GetFinancialSummaryUseCaseImpl`
**O quê:** Use case de resumo do mês.
**Onde:** `application/usecase/GetFinancialSummaryUseCaseImpl.java`
**Lógica:**
- `monthStart = LocalDate.now().withDayOfMonth(1)`
- `monthEnd = monthStart.plusMonths(1).minusDays(1)`
- Chamar os 6 métodos do `FinancialSummaryRepository`
- `predictedMonthResult = incomePredicted - expensePredicted`
- Retornar `SummaryResult`

**Depende de:** TASK-05, TASK-12
**Pronto quando:** compila

---

## Grupo 4 — Backend: Controller + DTOs

### TASK-19 [P] — DTOs de resposta e request
**O quê:** Records de HTTP na camada adapter.
**Onde:** `adapter/web/dto/`

**`FinancialSummaryResponse.java`:**
```java
public record FinancialSummaryResponse(
    BigDecimal predictedMonthResult, BigDecimal currentBalance,
    BigDecimal incomeReceived, BigDecimal incomePredicted,
    BigDecimal expensePaid, BigDecimal expensePredicted,
    List<MonthlyStatDTO> monthlySeries
) {
    public record MonthlyStatDTO(int year, int month, BigDecimal income, BigDecimal expense) {}
}
```

**`FinancialEntryResponse.java`:**
```java
public record FinancialEntryResponse(
    String id, String type, String expenseType, String description,
    BigDecimal amount, String dueDate, String paymentDate,
    String status, String category, String paymentMethod,
    String counterpart, String notes,
    boolean linkedToAppointment, String appointmentId
) {}
```

**`CreateFinancialEntryRequest.java`:**
```java
public record CreateFinancialEntryRequest(
    @NotBlank String type,
    String expenseType,
    @NotBlank String description,
    @NotNull @DecimalMin("0.01") BigDecimal amount,
    @NotNull LocalDate dueDate,
    LocalDate paymentDate,
    String category, String paymentMethod, String receivedFrom, String notes
) {}
```

**`UpdateFinancialEntryRequest.java`** — mesmos campos sem `type` (imutável após criação):
```java
public record UpdateFinancialEntryRequest(
    String description, BigDecimal amount, LocalDate dueDate,
    LocalDate paymentDate, String category, String expenseType,
    String paymentMethod, String receivedFrom, String notes
) {}
```

**Depende de:** TASK-05
**Pronto quando:** records compilam

---

### TASK-20 — `FinancialController`
**O quê:** Controller REST com os 6 endpoints.
**Onde:** `adapter/web/controller/FinancialController.java`
**Detalhes:**
- `@RestController @RequestMapping("/api/financial") @RequiredArgsConstructor`
- Injetar os 6 use cases como interfaces
- Mapper inline (record → DTO) em método privado `toResponse(EntryResult r)`
- `GET /summary` → `getSummaryUseCase.execute()` → mapear para `FinancialSummaryResponse`
- `GET /entries` → `@RequestParam LocalDate from, to, category, expenseType, type, page, size` → `listUseCase.execute(...)`; retornar `PageResponse<FinancialEntryResponse>`
- `POST /entries` → `@Valid @RequestBody CreateFinancialEntryRequest` → `createUseCase.execute(...)` → 201
- `PUT /entries/{id}` → `@Valid @RequestBody UpdateFinancialEntryRequest` → `updateUseCase.execute(...)` → 200
- `DELETE /entries/{id}` → `deleteUseCase.execute(id)` → 204
- `PATCH /entries/{id}/toggle-paid` → `toggleUseCase.execute(id)` → 200

**Depende de:** TASK-13, TASK-14, TASK-15, TASK-16, TASK-17, TASK-18, TASK-19
**Pronto quando:** todos os endpoints respondem corretamente com status HTTP corretos; backend sobe sem erro

---

## Grupo 5 — Frontend: Base (paralelo com Grupos 1-4)

### TASK-21 [P] — `financial.model.ts`
**O quê:** Tipos e interfaces TypeScript do módulo.
**Onde:** `features/financeiro/models/financial.model.ts`
Conforme design.md §4.1: `FinancialEntryType`, `FinancialExpenseType`, `FinancialStatus`, `FinancialPeriod`, `FinancialTab`, `FinancialSummary`, `MonthlyStatItem`, `FinancialEntry`.

**Depende de:** nada
**Pronto quando:** arquivo compila sem erros TypeScript

---

### TASK-22 [P] — `financeiro.routes.ts`
**O quê:** Arquivo de rotas da feature e registro no `app.routes.ts`.
**Onde:**
- `features/financeiro/financeiro.routes.ts`
- `app.routes.ts` (adicionar entry `{ path: 'financial', loadChildren: ... }`)

**Depende de:** nada
**Pronto quando:** rota `/financial` carrega (mesmo com componente vazio)

---

## Grupo 6 — Frontend: Store + Service

### TASK-23 — `FinancialService`
**O quê:** Service HTTP para todos os endpoints do módulo.
**Onde:** `features/financeiro/services/financial.service.ts`
**Métodos:**
- `getSummary(): Observable<FinancialSummary>`
- `listEntries(from: string, to: string, tab: FinancialTab, category?: string, page = 0, size = 20): Observable<PageResponse<FinancialEntry>>`
  - resolver `type` e `expenseType` a partir do `tab` conforme tabela do design.md §4.5
- `create(req: CreateFinancialEntryRequest): Observable<FinancialEntry>`
- `update(id: string, req: UpdateFinancialEntryRequest): Observable<FinancialEntry>`
- `delete(id: string): Observable<void>`
- `togglePaid(id: string): Observable<FinancialEntry>`

**Depende de:** TASK-21
**Pronto quando:** service compila; métodos tipados corretamente

---

### TASK-24 — NgRx Store do módulo financeiro
**O quê:** Actions, reducer, effects e selectors.
**Onde:** `features/financeiro/store/`

**`financial.actions.ts`** — conforme design.md §4.2

**`financial.reducer.ts`:**
- State conforme design.md §4.2
- Estado inicial: `activeTab = 'INCOME'`, `period = 'THIS_MONTH'`, `page = 0`
- `setTab` → atualiza `activeTab`, reset `page = 0`
- `setPeriod` → atualiza `period`, reset `page = 0`
- `setCategory` → atualiza `category`
- `loadSummarySuccess` / `loadEntriesSuccess` → atualiza dados, desliga loading
- Failure → `error`, desliga loading

**`financial.effects.ts`:**
- `loadSummary$` → `FinancialService.getSummary()`
- `loadEntries$` → resolve datas de `period` (helper: `resolveDateRange(period, customFrom, customTo)`) + `activeTab` → `FinancialService.listEntries(...)`
- `setTab$`, `setPeriod$`, `setCategory$` → disparam `loadEntries`
- `togglePaid$` → `FinancialService.togglePaid(id)` → `loadEntries`
- `createEntry$`, `updateEntry$` → `loadSummary` + `loadEntries`
- `deleteEntry$` → `loadSummary` + `loadEntries`

**`financial.selectors.ts`:**
- `selectSummary`, `selectEntries`, `selectTotalEntries`, `selectActiveTab`
- `selectPeriod`, `selectCategory`, `selectIsLoadingSummary`, `selectIsLoadingEntries`

**Depende de:** TASK-21, TASK-23
**Pronto quando:** store compila sem erros TypeScript

---

### TASK-25 — Registrar slice `financial` no app config
**O quê:** Adicionar reducer e effects ao `app.config.ts`.
**Onde:** `app.config.ts`
```typescript
provideStore({ ..., financial: financialReducer })
provideEffects([..., FinancialEffects])
```
**Depende de:** TASK-24
**Pronto quando:** app compila; DevTools mostra slice `financial`

---

## Grupo 7 — Frontend: Componentes (paralelo)

### TASK-26 [P] — `FinancialSummaryComponent`
**O quê:** Componente de header com resultado previsto, cards e gráfico.
**Onde:** `features/financeiro/components/financial-summary/`
**Inputs:** `summary$: Observable<FinancialSummary | null>`, `isLoading$: Observable<boolean>`

**Layout:**
- Bloco de destaque: "Resultado previsto" em fonte grande, cor verde/vermelho conforme sinal
- Subtítulo: "Previsão para [mês ano]"
- Linha de 2 cards lado a lado:
  - Card Recebimentos: valor recebido / previsto + `<mat-progress-bar>` + percentual
  - Card Despesas: valor pago / previsto + `<mat-progress-bar>` + percentual
- Card de saldo: "Saldo atual R$ X.XXX,00" + "Previsão do mês: +/- R$ X.XXX,00"
- Gráfico ApexCharts: `bar`, 2 séries (Recebimentos `#C8A2A2`, Despesas `#9E9E9E`), 6 meses, grouped
- Skeleton via `@if (isLoading$ | async)` antes de renderizar dados

Usar `currencyBr` pipe nos valores monetários.

**Depende de:** TASK-24, TASK-25
**Pronto quando:** componente renderiza com dados da store; gráfico exibe 6 barras; cores corretas

---

### TASK-27 [P] — `FinancialTransactionsComponent`
**O quê:** Abas de categorias + tabela de transações.
**Onde:** `features/financeiro/components/financial-transactions/`
**Inputs:** `entries$`, `totalEntries$`, `activeTab$`, `isLoading$`
**Outputs:** `tabChange: EventEmitter<FinancialTab>`, `togglePaid: EventEmitter<string>`, `editEntry: EventEmitter<FinancialEntry>`, `deleteEntry: EventEmitter<string>`

**`mat-tab-group`** com 6 abas; `(selectedTabChange)` emite `tabChange`

**`MatTable`** com colunas: `dueDate`, `description`, `counterpart`, `category`, `amount`, `paymentMethod`, `status`

Formatação de colunas:
- `dueDate` → `datePtbr` pipe
- `amount` → `currencyBr` pipe; prefixo "+" verde para INCOME, "−" vermelho para EXPENSE
- `status`: chip `[class.paid]` / `[class.pending]` / `[class.overdue]`; PENDING e PAID são clicáveis `(click)="togglePaid.emit(entry.id)"`; OVERDUE não clicável
- Ícone 🔗 (`link`) ao lado da descrição se `entry.linkedToAppointment`
- Botão lixeira `mat-icon-button` visível somente se `!entry.linkedToAppointment`

**`MatPaginator`** com `pageSize = 20`

Clique na linha → `editEntry.emit(entry)` (exceto clique nos botões de ação)

**Depende de:** TASK-24, TASK-25
**Pronto quando:** tabela exibe dados; toggle de pago dispara corretamente; exclusão emite evento

---

### TASK-28 [P] — `FinancialFormComponent` (MatDialog)
**O quê:** Formulário de nova/editar transação em dialog.
**Onde:** `features/financeiro/components/financial-form/`
**Data recebida via `MAT_DIALOG_DATA`:** `{ entry: FinancialEntry | null }`

**Reactive Form** com campos:
- `type` (mat-radio-group INCOME/EXPENSE) — somente para nova transação
- `expenseType` (mat-select) — visível apenas quando `type === 'EXPENSE'`
- `description` (mat-input, obrigatório)
- `amount` (mat-input number, > 0, obrigatório)
- `dueDate` (mat-datepicker, obrigatório)
- `category` (mat-input com datalist de sugestões por tipo)
- `paymentMethod` (mat-select: PIX, Dinheiro, Cartão Débito, Cartão Crédito, Transferência, Outros)
- `receivedFrom` (mat-input)
- `paymentDate` (mat-datepicker)
- `notes` (mat-textarea)

**Modo edição de lançamento vinculado:** campos editáveis = `paymentMethod`, `paymentDate`, `notes`; demais com `[readonly]`; chip informativo "Gerado por agendamento" + link para `/appointments/:appointmentId`

**Ao salvar:**
- Nova transação → despacha `FinancialActions.createEntry({ request })`
- Edição → despacha `FinancialActions.updateEntry({ id, request })`
- Dialog fecha via `MatDialogRef.close()`

**Depende de:** TASK-24, TASK-25
**Pronto quando:** dialog abre em modo novo e modo edição; salva e fecha corretamente

---

## Grupo 8 — Frontend: Shell final

### TASK-29 — `FinanceiroComponent` (shell + integração)
**O quê:** Componente principal que orquestra tudo.
**Onde:** `features/financeiro/financeiro.component.ts/html/css`

**`ngOnInit`:**
```typescript
this.store.dispatch(FinancialActions.loadSummary());
this.store.dispatch(FinancialActions.loadEntries());
```

**Template — estrutura:**
```
Cabeçalho da página: "Financeiro"

Linha de filtros:
  [Período: mat-select]    [Categoria: mat-select]    [+ Nova transação]

FinancialSummaryComponent

FinancialTransactionsComponent
```

**Filtros:**
- `mat-select` de período com opções de `FinancialPeriod`; quando "Personalizado" → exibir 2 `mat-datepicker`
- `mat-select` de categoria populado com `selectDistinctCategories` (seletor derivado das entries carregadas)
- Mudança despacha `setPeriod` ou `setCategory`

**Botão "Nova transação":** abre `FinancialFormComponent` via `MatDialog.open()`

**Eventos dos sub-componentes:**
- `tabChange` → `store.dispatch(setTab({ tab }))`
- `togglePaid(id)` → `store.dispatch(togglePaid({ id }))`
- `editEntry(entry)` → `dialog.open(FinancialFormComponent, { data: { entry } })`
- `deleteEntry(id)` → exibir `MatDialog` de confirmação → confirmar → `store.dispatch(deleteEntry({ id }))`

**CSS:**
- Layout geral com `display: flex; flex-direction: column; gap: 24px`
- Responsivo: filtros em coluna em mobile

**Depende de:** TASK-25, TASK-26, TASK-27, TASK-28
**Pronto quando:** tela abre, summary carrega, tabela carrega, nova transação salva, toggle pago funciona, exclusão funciona com confirmação

---

## Ordem de Execução Recomendada

```
[Paralelo]  TASK-01 + TASK-03 + TASK-04 + TASK-21 + TASK-22
     ↓
[Paralelo]  TASK-02 + TASK-05 + TASK-06 + TASK-07
     ↓
[Paralelo]  TASK-08 + TASK-09 + TASK-10
     ↓
[Paralelo]  TASK-11 + TASK-12
     ↓
[Paralelo]  TASK-13 + TASK-14 + TASK-15 + TASK-16 + TASK-17 + TASK-18
     ↓
[Paralelo]  TASK-19 + TASK-23
     ↓
            TASK-20 → TASK-24 → TASK-25
     ↓
[Paralelo]  TASK-26 + TASK-27 + TASK-28
     ↓
            TASK-29
```

**Total: 29 tasks | Backend: 20 tasks | Frontend: 9 tasks**
