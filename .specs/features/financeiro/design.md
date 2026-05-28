# Design — Módulo Financeiro

**Data:** 2026-05-23
**Status:** Aprovado para implementação

---

## 1. Visão Geral da Arquitetura

```
GET /api/financial/summary
GET /api/financial/entries?from=&to=&category=&expenseType=&page=&size=
POST/PUT /api/financial/entries
DELETE /api/financial/entries/{id}
PATCH /api/financial/entries/{id}/toggle-paid
        │
        ▼
FinancialController
        │
        ├── GetFinancialSummaryUseCase    → FinancialSummaryRepositoryImpl (EntityManager)
        ├── ListFinancialEntriesUseCase  → FinancialEntryRepositoryImpl (Spring Data + JOIN)
        ├── CreateFinancialEntryUseCase  → FinancialEntryRepositoryImpl
        ├── UpdateFinancialEntryUseCase  → FinancialEntryRepositoryImpl
        ├── DeleteFinancialEntryUseCase  → FinancialEntryRepositoryImpl
        └── ToggleFinancialEntryPaidUseCase → FinancialEntryRepositoryImpl
```

Frontend:
```
FinanceiroComponent (shell com filtros + abas)
   ├── NgRx Store (financial slice)
   ├── financial-summary/     (header resultado + cards + gráfico)
   ├── financial-transactions/ (abas + tabela paginada)
   └── financial-form/        (MatDialog para nova/editar transação)
```

---

## 2. Migration V5

```sql
-- V5__financial_entries_new_fields.sql
ALTER TABLE financial_entries
  ADD COLUMN payment_method VARCHAR(50),
  ADD COLUMN expense_type VARCHAR(20),
  ADD COLUMN received_from VARCHAR(255);

ALTER TABLE financial_entries
  ADD CONSTRAINT chk_expense_type
  CHECK (expense_type IN ('FIXED', 'VARIABLE', 'PEOPLE', 'TAX', 'TRANSFER'));
```

---

## 3. Backend

### 3.1 Domain — novos artefatos

**`FinancialEntryExpenseType.java`** (enum)
```java
public enum FinancialEntryExpenseType {
    FIXED, VARIABLE, PEOPLE, TAX, TRANSFER
}
```

**`FinancialEntry.java`** — adicionar 3 campos ao modelo existente:
```java
private String paymentMethod;
private FinancialEntryExpenseType expenseType;
private String receivedFrom;
```

### 3.2 Ports/Out — dois repositórios

**`FinancialEntryRepository`** (estender o existente):
```java
public interface FinancialEntryRepository {
    FinancialEntry save(FinancialEntry entry);
    Optional<FinancialEntry> findById(UUID id);
    Page<FinancialEntryWithCounterpart> listWithFilters(
        LocalDate from, LocalDate to,
        String category, String expenseType, String type,
        Pageable pageable);
    void delete(UUID id);
    boolean existsByIdAndAppointmentIdIsNull(UUID id);  // guarda de exclusão
}
```

`FinancialEntryWithCounterpart` — record interno do repositório:
```java
public record FinancialEntryWithCounterpart(
    FinancialEntry entry,
    String counterpart   // nome do cliente (via appointment) ou received_from
) {}
```

**`FinancialSummaryRepository`** (port/out, novo):
```java
public interface FinancialSummaryRepository {
    BigDecimal sumIncomePaidInMonth(LocalDate monthStart, LocalDate monthEnd);
    BigDecimal sumIncomeTotalInMonth(LocalDate monthStart, LocalDate monthEnd);  // PAID + PENDING
    BigDecimal sumExpensePaidInMonth(LocalDate monthStart, LocalDate monthEnd);
    BigDecimal sumExpenseTotalInMonth(LocalDate monthStart, LocalDate monthEnd);
    BigDecimal sumAllTimePaidBalance();                    // saldo atual
    List<MonthlyFinancialStat> last6MonthsStats();         // série do gráfico
}
```

`MonthlyFinancialStat` — record em `domain/model/`:
```java
public record MonthlyFinancialStat(
    int year, int month,
    BigDecimal incomeTotal,
    BigDecimal expenseTotal
) {}
```

### 3.3 Infrastructure — entidades e repositórios

**`FinancialEntryEntity.java`** — adicionar ao existente:
```java
@Column(name = "payment_method")
private String paymentMethod;

@Column(name = "expense_type")
private String expenseType;

@Column(name = "received_from")
private String receivedFrom;

// Associação read-only para JOIN com appointments → clients
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "appointment_id", insertable = false, updatable = false)
private AppointmentEntity appointment;
```

> O campo `appointmentId UUID` existente permanece como dono do FK. A associação `appointment` é apenas para leitura (JOIN no JPQL para resolver nome do cliente).

**`FinancialEntryJpaRepository.java`** — novo método de query:
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
```

**`FinancialSummaryRepositoryImpl.java`** — usa `EntityManager`:
```java
// Queries JPQL principais:

// Receita PAID no mês
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'INCOME' AND f.status = 'PAID'
AND f.paymentDate >= :start AND f.paymentDate <= :end

// Receita total prevista no mês (PAID + PENDING)
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'INCOME' AND f.status IN ('PAID','PENDING')
AND f.dueDate >= :start AND f.dueDate <= :end

// Despesa PAID no mês
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'EXPENSE' AND f.status = 'PAID'
AND f.paymentDate >= :start AND f.paymentDate <= :end

// Despesa total prevista no mês (PAID + PENDING)
SELECT COALESCE(SUM(f.amount), 0) FROM FinancialEntryEntity f
WHERE f.type = 'EXPENSE' AND f.status IN ('PAID','PENDING')
AND f.dueDate >= :start AND f.dueDate <= :end

// Saldo atual (all-time)
SELECT
  COALESCE(SUM(CASE WHEN f.type='INCOME' THEN f.amount ELSE -f.amount END), 0)
FROM FinancialEntryEntity f
WHERE f.status = 'PAID'

// Série dos últimos 6 meses (gráfico)
SELECT YEAR(f.paymentDate), MONTH(f.paymentDate), f.type, SUM(f.amount)
FROM FinancialEntryEntity f
WHERE f.status = 'PAID'
AND f.paymentDate >= :sixMonthsAgo
GROUP BY YEAR(f.paymentDate), MONTH(f.paymentDate), f.type
ORDER BY YEAR(f.paymentDate), MONTH(f.paymentDate)
```

### 3.4 Use Cases

**`GetFinancialSummaryUseCaseImpl`**
- Calcula `monthStart` / `monthEnd` do mês corrente
- Chama 6 métodos do `FinancialSummaryRepository` e monta `FinancialSummaryResult`
- `predictedResult = incomeTotal - expenseTotal` (ambos do mês)
- `currentBalance = allTimePaidBalance`

**`ListFinancialEntriesUseCaseImpl`**
- Mapeia `expenseType` param da aba → tipo+subtype (`INCOME`, ou `EXPENSE+FIXED`, etc.)
- Resolve `counterpart`: se `entry.appointmentId != null` → `appointment.client.name`; senão → `entry.receivedFrom`
- Retorna `Page<FinancialEntryResult>`

**`CreateFinancialEntryUseCaseImpl`**
- Se `command.paymentDate != null` → força `status = PAID`; senão → `status = PENDING`
- Salva via `FinancialEntryRepository.save()`

**`UpdateFinancialEntryUseCaseImpl`**
- Carrega entry; se `entry.appointmentId != null` → permite editar somente: `paymentMethod`, `paymentDate`, `status`, `notes`
- Mesma lógica de `paymentDate → status` do create

**`DeleteFinancialEntryUseCaseImpl`**
- Checa `existsByIdAndAppointmentIdIsNull(id)`: se false → lança `FinancialEntryLinkedToAppointmentException` (409)
- Chama `FinancialEntryRepository.delete(id)`

**`ToggleFinancialEntryPaidUseCaseImpl`**
- Se `status = PENDING` → `status = PAID`, `paymentDate = today`
- Se `status = PAID` → `status = PENDING`, `paymentDate = null`
- `OVERDUE` → não altera (lança `IllegalStateException`)

### 3.5 DTOs e Controller

**`FinancialSummaryResponse`** (record):
```java
public record FinancialSummaryResponse(
    BigDecimal predictedMonthResult,  // pode ser negativo
    BigDecimal currentBalance,
    BigDecimal incomeReceived,        // PAID no mês
    BigDecimal incomePredicted,       // PAID+PENDING no mês
    BigDecimal expensePaid,           // PAID no mês
    BigDecimal expensePredicted,      // PAID+PENDING no mês
    List<MonthlyStatDTO> monthlySeries // últimos 6 meses
) {}

record MonthlyStatDTO(int year, int month, BigDecimal income, BigDecimal expense) {}
```

**`FinancialEntryResponse`** (record):
```java
public record FinancialEntryResponse(
    String id, String type, String expenseType,
    String description, BigDecimal amount,
    String dueDate, String paymentDate,
    String status, String category,
    String paymentMethod, String counterpart,   // resolved: client name ou receivedFrom
    String notes, boolean linkedToAppointment,
    String appointmentId
) {}
```

**`CreateFinancialEntryRequest`** / **`UpdateFinancialEntryRequest`** — campos editáveis conforme spec FIN-09.

**`FinancialController`**:
```java
@RestController
@RequestMapping("/api/financial")
@RequiredArgsConstructor
public class FinancialController {

    @GetMapping("/summary")
    public ResponseEntity<FinancialSummaryResponse> getSummary() { ... }

    @GetMapping("/entries")
    public ResponseEntity<PageResponse<FinancialEntryResponse>> listEntries(
        @RequestParam LocalDate from, @RequestParam LocalDate to,
        @RequestParam(required=false) String category,
        @RequestParam(required=false) String expenseType,
        @RequestParam(defaultValue="0") int page,
        @RequestParam(defaultValue="20") int size) { ... }

    @PostMapping("/entries")
    public ResponseEntity<FinancialEntryResponse> create(@RequestBody @Valid CreateFinancialEntryRequest req) { ... }

    @PutMapping("/entries/{id}")
    public ResponseEntity<FinancialEntryResponse> update(@PathVariable UUID id,
        @RequestBody @Valid UpdateFinancialEntryRequest req) { ... }

    @DeleteMapping("/entries/{id}")
    public ResponseEntity<Void> delete(@PathVariable UUID id) { ... }

    @PatchMapping("/entries/{id}/toggle-paid")
    public ResponseEntity<FinancialEntryResponse> togglePaid(@PathVariable UUID id) { ... }
}
```

**Exceções novas** (`domain/exception/`):
- `FinancialEntryNotFoundException` (404)
- `FinancialEntryLinkedToAppointmentException` (409)

---

## 4. Frontend

### 4.1 Modelo TypeScript (`financial.model.ts`)

```typescript
export type FinancialEntryType = 'INCOME' | 'EXPENSE';
export type FinancialExpenseType = 'FIXED' | 'VARIABLE' | 'PEOPLE' | 'TAX' | 'TRANSFER';
export type FinancialStatus = 'PENDING' | 'PAID' | 'OVERDUE';
export type FinancialPeriod = 'THIS_MONTH' | 'THIS_WEEK' | 'TODAY' | 'LAST_30' | 'LAST_90' | 'THIS_YEAR' | 'CUSTOM';
export type FinancialTab = 'INCOME' | 'FIXED' | 'VARIABLE' | 'PEOPLE' | 'TAX' | 'TRANSFER';

export interface FinancialSummary {
  predictedMonthResult: number;
  currentBalance: number;
  incomeReceived: number;
  incomePredicted: number;
  expensePaid: number;
  expensePredicted: number;
  monthlySeries: MonthlyStatItem[];
}

export interface MonthlyStatItem { year: number; month: number; income: number; expense: number; }

export interface FinancialEntry {
  id: string;
  type: FinancialEntryType;
  expenseType: FinancialExpenseType | null;
  description: string;
  amount: number;
  dueDate: string;
  paymentDate: string | null;
  status: FinancialStatus;
  category: string | null;
  paymentMethod: string | null;
  counterpart: string | null;
  notes: string | null;
  linkedToAppointment: boolean;
  appointmentId: string | null;
}
```

### 4.2 NgRx Store

**State:**
```typescript
interface FinancialState {
  summary: FinancialSummary | null;
  entries: FinancialEntry[];
  totalEntries: number;
  activeTab: FinancialTab;
  period: FinancialPeriod;
  customFrom: string | null;
  customTo: string | null;
  category: string | null;
  page: number;
  isLoadingSummary: boolean;
  isLoadingEntries: boolean;
  error: string | null;
}
```

**Actions:**
```
[Financial] Load Summary
[Financial] Load Summary Success/Failure
[Financial] Load Entries
[Financial] Load Entries Success/Failure
[Financial] Set Tab
[Financial] Set Period
[Financial] Set Category
[Financial] Toggle Paid
[Financial] Toggle Paid Success/Failure
[Financial] Create Entry
[Financial] Create Entry Success/Failure
[Financial] Update Entry
[Financial] Update Entry Success/Failure
[Financial] Delete Entry
[Financial] Delete Entry Success/Failure
```

**Effects:**
- `setTab$`, `setPeriod$`, `setCategory$` → disparam `loadEntries`
- `loadSummary$` → `FinancialService.getSummary()`
- `loadEntries$` → resolve datas a partir do `period` + `activeTab` → `FinancialService.listEntries(...)`
- `togglePaid$` → `FinancialService.togglePaid(id)` → `loadEntries` (recarrega a aba atual)
- `createEntry$` e `updateEntry$` → fecha dialog + `loadSummary` + `loadEntries`
- `deleteEntry$` → `loadSummary` + `loadEntries`

**Seletores:** `selectSummary`, `selectEntries`, `selectActiveTab`, `selectPeriod`, `selectIsLoadingSummary`, `selectIsLoadingEntries`, `selectTabCounts` (contagem por aba derivada do total)

### 4.3 Componentes

#### `FinanceiroComponent` (shell)
- Carrega `summary` e `entries` no `ngOnInit`
- Contém filtros de período e categoria (mat-select)
- Contém o toggle de abas
- Orquestra `FinancialSummaryComponent`, `FinancialTransactionsComponent`

#### `FinancialSummaryComponent`
Inputs: `summary$`, `isLoading$`

Layout:
```
┌──────────────────────────────────────────────────────┐
│  Resultado previsto: R$ 2.340,00  [verde/vermelho]   │  ← FIN-01
│  Previsão para Maio 2026                             │
├─────────────────────┬──────────────────────────────── ┤
│ RECEBIMENTOS        │ DESPESAS                        │  ← FIN-02/03
│ R$ 1.800 / R$ 2.500 │ R$ 860 / R$ 1.200             │
│ [████████░░] 72%    │ [███████░░░] 64%               │
├─────────────────────┴────────────────────────────────┤
│  SALDO ATUAL: R$ 4.120,00    Previsão: +R$ 2.340,00  │  ← FIN-04
├──────────────────────────────────────────────────────┤
│  [Gráfico de barras — Recebimentos vs Despesas]      │  ← FIN-05
│  últimos 6 meses                                     │
└──────────────────────────────────────────────────────┘
```

- Gráfico: ApexCharts `bar`, grouped (não stacked), 2 séries, `ng-apexcharts@1.11.0`
- Rosé `#C8A2A2` para Recebimentos; cinza `#9E9E9E` para Despesas

#### `FinancialTransactionsComponent`
Inputs: `entries$`, `activeTab$`, `isLoading$`
Outputs: `tabChange`, `togglePaid`, `editEntry`, `deleteEntry`

- `mat-tab-group` com 6 abas; cada aba despacha `setTab`
- Tabela com `MatTable` + `MatPaginator`
- Coluna "Pago": chip clicável — `(click)` despacha `togglePaid({id})`; chip OVERDUE em vermelho não clicável
- Ícone de link (🔗) na linha para entradas vinculadas a agendamento
- Ícone lixeira apenas para entradas sem vínculo (`linkedToAppointment = false`)
- Click na linha → abre `FinancialFormComponent` via `MatDialog`

#### `FinancialFormComponent` (MatDialog)
Data recebida: `{ entry: FinancialEntry | null }` (null = nova transação)
- `ReactiveFormsGroup` com todos os campos de FIN-09
- Campo `expenseType` aparece somente quando `type = EXPENSE`
- Campo `paymentDate` preenchido automaticamente se "Marcar como pago" for selecionado
- Se `entry.linkedToAppointment = true`: somente `paymentMethod`, `paymentDate`, `status`, `notes` habilitados
- Ao salvar: despacha `createEntry` ou `updateEntry`

### 4.4 Rotas

```typescript
// financeiro.routes.ts
export const financeiroRoutes: Routes = [
  { path: '', component: FinanceiroComponent }
];
```

```typescript
// app.routes.ts — adicionar:
{
  path: 'financial',
  loadChildren: () => import('./features/financeiro/financeiro.routes')
    .then(m => m.financeiroRoutes)
}
```

Sidebar e bottom nav já têm o item Financial como placeholder — só precisam da rota ativada.

### 4.5 `FinancialService` (HTTP)

```typescript
@Injectable({ providedIn: 'root' })
export class FinancialService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = '/api/financial';

  getSummary(): Observable<FinancialSummary> { ... }

  listEntries(from: string, to: string, tab: FinancialTab,
              category?: string, page = 0, size = 20): Observable<PageResponse<FinancialEntry>> {
    // resolve type + expenseType a partir do tab
  }

  create(req: CreateFinancialEntryRequest): Observable<FinancialEntry> { ... }
  update(id: string, req: UpdateFinancialEntryRequest): Observable<FinancialEntry> { ... }
  delete(id: string): Observable<void> { ... }
  togglePaid(id: string): Observable<FinancialEntry> { ... }
}
```

**Mapeamento tab → query params:**
| Tab | `type` param | `expenseType` param |
|---|---|---|
| INCOME | INCOME | — |
| FIXED | EXPENSE | FIXED |
| VARIABLE | EXPENSE | VARIABLE |
| PEOPLE | EXPENSE | PEOPLE |
| TAX | EXPENSE | TAX |
| TRANSFER | EXPENSE | TRANSFER |

---

## 5. Decisões de Design

| Decisão | Razão |
|---|---|
| `FinancialSummaryRepository` separado | Segue o padrão de `DashboardRepositoryImpl` — queries de agregação com EntityManager ficam isoladas das operações CRUD |
| ManyToOne read-only em `FinancialEntryEntity` | Permite JPQL com LEFT JOIN FETCH para resolver nome do cliente sem query extra; `insertable=false, updatable=false` garante que o FK é controlado só pelo campo `appointmentId UUID` |
| `counterpart` resolvido no repositório | Use case não conhece JPA; a resolução do nome do cliente é um detalhe de persistência |
| `MatDialog` para o formulário (não rota) | Consistência com `AppointmentPopupComponent`; o contexto financeiro não tem rota de detalhe — o formulário é sempre contextual |
| Filtro de período no store, não na URL | Simplifica navegação; estado de filtro não precisa ser bookmarkável nesta fase |
| Tabs mapeiam para `expenseType` no backend | Evita criar um novo campo `tab` — `expenseType` já carrega a semântica; backend não precisa saber sobre UI |
| `OVERDUE` não alterna via toggle | Status OVERDUE implica vencimento passado sem pagamento — alternar para PENDING seria incorreto; usuário deve editar a data de vencimento |
