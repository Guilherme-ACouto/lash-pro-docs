# Estoque — Design

**Spec**: `.specs/features/estoque/spec.md`
**Status**: Draft

---

## 1. Visão Geral da Arquitetura

```
POST/PUT  /api/inventory/items
GET       /api/inventory/items?search=&status=&filter=&page=&size=
GET       /api/inventory/items/{id}
DELETE    /api/inventory/items/{id}
PATCH     /api/inventory/items/{id}/deactivate
PATCH     /api/inventory/items/{id}/reactivate
POST      /api/inventory/items/{id}/purchase     ← entrada + cria FinancialEntry
POST      /api/inventory/items/{id}/exit         ← saída manual (sem financeiro)
GET       /api/inventory/items/{id}/movements
        │
        ▼
InventoryController
        │
        ├── CreateInventoryItemUseCase       → InventoryItemRepository
        ├── UpdateInventoryItemUseCase       → InventoryItemRepository
        ├── DeleteInventoryItemUseCase       → InventoryItemRepository + InventoryMovementRepository
        ├── ListInventoryItemsUseCase        → InventoryItemRepository
        ├── GetInventoryItemUseCase          → InventoryItemRepository
        ├── DeactivateInventoryItemUseCase   → InventoryItemRepository
        ├── RegisterPurchaseUseCase          → InventoryItemRepository
        │                                   + InventoryMovementRepository
        │                                   + FinancialEntryRepository  ← integração financeiro
        ├── RegisterManualExitUseCase        → InventoryItemRepository
        │                                   + InventoryMovementRepository
        └── ListMovementsUseCase             → InventoryMovementRepository
```

Frontend:

```
InventoryComponent (shell — lista + alertas)
   ├── NgRx Store (inventory slice)
   ├── inventory-list/        (tabela com busca, paginação, badges)
   ├── inventory-form/        (MatDialog — cadastro e edição de item)
   ├── inventory-purchase/    (MatDialog — registrar compra)
   └── inventory-exit/        (MatDialog — saída manual)
```

---

## 2. Migration V6

```sql
-- V6__inventory_full_schema.sql

-- Campos faltantes em inventory_items
ALTER TABLE inventory_items
  ADD COLUMN internal_code  VARCHAR(50),
  ADD COLUMN cost_price     NUMERIC(10,2) NOT NULL DEFAULT 0,
  ADD COLUMN supplier       VARCHAR(255),
  ADD COLUMN notes          TEXT;

ALTER TABLE inventory_items
  ADD CONSTRAINT uq_inventory_code UNIQUE (internal_code);

-- Enriquecer inventory_movements com campos de compra
ALTER TABLE inventory_movements
  ADD COLUMN reason           VARCHAR(20) NOT NULL DEFAULT 'ADJUSTMENT',
  ADD COLUMN unit_cost        NUMERIC(10,2),
  ADD COLUMN supplier         VARCHAR(255),
  ADD COLUMN purchase_date    DATE,
  ADD COLUMN payment_type     VARCHAR(10),
  ADD COLUMN due_date         DATE,
  ADD COLUMN financial_entry_id UUID REFERENCES financial_entries(id);

-- Ampliar constraint de tipo de movimento (IN/OUT → PURCHASE/USAGE/LOSS/ADJUSTMENT/OTHER)
ALTER TABLE inventory_movements DROP CONSTRAINT chk_movement_type;
ALTER TABLE inventory_movements
  ADD CONSTRAINT chk_movement_type CHECK (type IN ('IN', 'OUT'));
ALTER TABLE inventory_movements
  ADD CONSTRAINT chk_movement_reason CHECK (reason IN ('PURCHASE','USAGE','LOSS','ADJUSTMENT','OTHER'));
ALTER TABLE inventory_movements
  ADD CONSTRAINT chk_payment_type CHECK (payment_type IN ('CASH','INVOICE') OR payment_type IS NULL);

-- Adicionar SUPPLY ao tipo de despesa financeira
ALTER TABLE financial_entries DROP CONSTRAINT IF EXISTS chk_expense_type;
ALTER TABLE financial_entries
  ADD CONSTRAINT chk_expense_type
  CHECK (expense_type IN ('FIXED','VARIABLE','PEOPLE','TAX','TRANSFER','SUPPLY'));

CREATE INDEX idx_inventory_movements_item ON inventory_movements(item_id);
CREATE INDEX idx_inventory_movements_date ON inventory_movements(purchase_date);
```

---

## 3. Backend

### 3.1 Domain — novos artefatos e alterações

**`InventoryItem.java`** — adicionar 4 campos ao modelo existente:
```java
private String internalCode;
private BigDecimal costPrice;
private String supplier;
private String notes;

// método auxiliar já existente:
public boolean isBelowMinimum() { ... }

// novo:
public boolean isOutOfStock() {
    return currentQuantity.compareTo(BigDecimal.ZERO) == 0;
}
```

**`InventoryMovement.java`** — novo domain record:
```java
public class InventoryMovement {
    private UUID id;
    private UUID itemId;
    private String itemName;         // desnormalizado para exibição
    private MovementType type;       // IN | OUT
    private MovementReason reason;   // PURCHASE | USAGE | LOSS | ADJUSTMENT | OTHER
    private BigDecimal quantity;
    private BigDecimal unitCost;     // nulo em saídas
    private BigDecimal totalCost;    // quantity × unitCost, nulo em saídas
    private String supplier;
    private LocalDate purchaseDate;
    private PurchasePaymentType paymentType;   // CASH | INVOICE, nulo em saídas
    private LocalDate dueDate;                 // nulo se CASH ou saída
    private UUID financialEntryId;             // nulo em saídas manuais
    private String notes;
    private LocalDateTime createdAt;
}
```

**Novos enums** em `domain/model/`:
```java
public enum MovementType    { IN, OUT }
public enum MovementReason  { PURCHASE, USAGE, LOSS, ADJUSTMENT, OTHER }
public enum PurchasePaymentType { CASH, INVOICE }
```

**`FinancialEntryExpenseType.java`** — adicionar valor:
```java
public enum FinancialEntryExpenseType {
    FIXED, VARIABLE, PEOPLE, TAX, TRANSFER, SUPPLY  // ← novo
}
```

**Novas exceções** em `domain/exception/`:
- `InventoryItemNotFoundException` (404)
- `InventoryItemCodeAlreadyExistsException` (409)
- `InventoryItemHasMovementsException` (409) — bloqueia exclusão

### 3.2 Ports/In — use cases

```java
// CreateInventoryItemUseCase
CreateInventoryItemCommand(name, unit, costPrice, supplier, internalCode,
                           currentQuantity, minimumQuantity, notes)
InventoryItemResult execute(command)

// UpdateInventoryItemUseCase
UpdateInventoryItemCommand(id, name, unit, costPrice, supplier,
                           minimumQuantity, notes)
InventoryItemResult execute(command)

// DeleteInventoryItemUseCase
void execute(UUID id)   // lança InventoryItemHasMovementsException se tiver movimentações

// ListInventoryItemsUseCase
ListInventoryItemsQuery(search, status, filter, page, size)
// status: ALL | ACTIVE | INACTIVE
// filter: null | LOW_STOCK (current ≤ minimum)
Page<InventoryItemResult> execute(query)

// GetInventoryItemUseCase
InventoryItemResult execute(UUID id)

// DeactivateInventoryItemUseCase
InventoryItemResult execute(UUID id, boolean activate)

// RegisterPurchaseUseCase  ← fluxo principal
RegisterPurchaseCommand(itemId, quantity, unitCost, supplier, purchaseDate,
                        paymentType, dueDate, notes)
RegisterPurchaseResult execute(command)
// RegisterPurchaseResult: { item: InventoryItemResult, movement: InventoryMovementResult, financialEntryId: UUID }

// RegisterManualExitUseCase
RegisterManualExitCommand(itemId, quantity, reason, notes, exitDate)
InventoryMovementResult execute(command)

// ListMovementsUseCase
Page<InventoryMovementResult> execute(UUID itemId, int page, int size)
```

### 3.3 Ports/Out

```java
public interface InventoryItemRepository {
    InventoryItem save(InventoryItem item);
    Optional<InventoryItem> findById(UUID id);
    Page<InventoryItem> listWithFilters(String search, Boolean active,
                                        boolean onlyLowStock, Pageable pageable);
    boolean existsByInternalCode(String code);
    boolean existsByInternalCodeAndIdNot(String code, UUID id);
    void delete(UUID id);
}

public interface InventoryMovementRepository {
    InventoryMovement save(InventoryMovement movement);
    Page<InventoryMovement> findByItemId(UUID itemId, Pageable pageable);
    boolean existsByItemId(UUID itemId);
}
```

### 3.4 Infrastructure

**`InventoryItemEntity.java`**:
```java
@Entity @Table(name = "inventory_items")
@Getter @Builder @NoArgsConstructor @AllArgsConstructor
public class InventoryItemEntity {
    @Id @GeneratedValue private UUID id;
    @Column(nullable = false) private String name;
    @Column(name = "internal_code", unique = true) private String internalCode;
    @Column(nullable = false) private String unit;
    @Column(name = "cost_price", nullable = false) private BigDecimal costPrice;
    private String supplier;
    @Column(name = "current_quantity", nullable = false) private BigDecimal currentQuantity;
    @Column(name = "minimum_quantity", nullable = false) private BigDecimal minimumQuantity;
    private boolean active;
    private String notes;
    @Column(name = "created_at") private LocalDateTime createdAt;
    @Column(name = "updated_at") private LocalDateTime updatedAt;
}
```

**`InventoryMovementEntity.java`**:
```java
@Entity @Table(name = "inventory_movements")
@Getter @Builder @NoArgsConstructor @AllArgsConstructor
public class InventoryMovementEntity {
    @Id @GeneratedValue private UUID id;
    @Column(name = "item_id", nullable = false) private UUID itemId;
    @Column(nullable = false) private String type;          // IN | OUT
    @Column(nullable = false) private String reason;
    @Column(nullable = false) private BigDecimal quantity;
    @Column(name = "unit_cost") private BigDecimal unitCost;
    private String supplier;
    @Column(name = "purchase_date") private LocalDate purchaseDate;
    @Column(name = "payment_type") private String paymentType;
    @Column(name = "due_date") private LocalDate dueDate;
    @Column(name = "financial_entry_id") private UUID financialEntryId;
    private String notes;
    @Column(name = "created_at") private LocalDateTime createdAt;

    // Associação read-only ao item para resolver o nome no histórico
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "item_id", insertable = false, updatable = false)
    private InventoryItemEntity item;
}
```

**`InventoryItemJpaRepository`**: extends `JpaRepository<InventoryItemEntity, UUID>`
```java
@Query("""
    SELECT i FROM InventoryItemEntity i
    WHERE (:active IS NULL OR i.active = :active)
    AND (:search IS NULL OR LOWER(i.name) LIKE LOWER(CONCAT('%',:search,'%'))
         OR LOWER(i.internalCode) LIKE LOWER(CONCAT('%',:search,'%')))
    AND (:onlyLowStock = false OR i.currentQuantity <= i.minimumQuantity)
    ORDER BY i.name ASC
""")
Page<InventoryItemEntity> findWithFilters(@Param("active") Boolean active,
                                          @Param("search") String search,
                                          @Param("onlyLowStock") boolean onlyLowStock,
                                          Pageable pageable);
```

**`InventoryMovementJpaRepository`**: extends `JpaRepository<InventoryMovementEntity, UUID>`
```java
Page<InventoryMovementEntity> findByItemIdOrderByCreatedAtDesc(UUID itemId, Pageable pageable);
boolean existsByItemId(UUID itemId);
```

### 3.5 Use Cases — lógica crítica

**`RegisterPurchaseUseCaseImpl`** — coordena 3 operações atomicamente (`@Transactional`):
```java
// 1. Carrega o item
InventoryItem item = itemRepo.findById(cmd.itemId()) ...

// 2. Atualiza o currentQuantity e costPrice do item
InventoryItem updated = InventoryItem.builder()
    .id(item.getId())
    ...campos existentes...
    .currentQuantity(item.getCurrentQuantity().add(cmd.quantity()))
    .costPrice(cmd.unitCost())    // atualiza custo unitário com o da nota
    .supplier(cmd.supplier() != null ? cmd.supplier() : item.getSupplier())
    .updatedAt(LocalDateTime.now())
    .build();
itemRepo.save(updated);

// 3. Cria a movimentação
BigDecimal total = cmd.quantity().multiply(cmd.unitCost());
InventoryMovement movement = InventoryMovement.builder()
    .itemId(cmd.itemId())
    .type(MovementType.IN)
    .reason(MovementReason.PURCHASE)
    .quantity(cmd.quantity())
    .unitCost(cmd.unitCost())
    .supplier(cmd.supplier())
    .purchaseDate(cmd.purchaseDate())
    .paymentType(cmd.paymentType())
    .dueDate(cmd.paymentType() == CASH ? null : cmd.dueDate())
    .build();

// 4. Cria a FinancialEntry de EXPENSE
boolean isPaid = cmd.paymentType() == PurchasePaymentType.CASH;
FinancialEntry expense = FinancialEntry.builder()
    .type(FinancialEntryType.EXPENSE)
    .expenseType(FinancialEntryExpenseType.SUPPLY)
    .description("Compra: " + item.getName())
    .amount(total)
    .dueDate(isPaid ? cmd.purchaseDate() : cmd.dueDate())
    .paymentDate(isPaid ? cmd.purchaseDate() : null)
    .status(isPaid ? FinancialEntryStatus.PAID : FinancialEntryStatus.PENDING)
    .notes(cmd.notes())
    .createdAt(LocalDateTime.now())
    .build();
FinancialEntry savedExpense = financialEntryRepo.save(expense);

// 5. Salva a movimentação com o financialEntryId
InventoryMovement savedMovement = movementRepo.save(
    movement.toBuilder().financialEntryId(savedExpense.getId()).build()
);
```

**`DeleteInventoryItemUseCaseImpl`**:
```java
if (movementRepo.existsByItemId(id)) {
    throw new InventoryItemHasMovementsException(id);
}
itemRepo.delete(id);
```

**`CreateInventoryItemUseCaseImpl`** — geração de código:
```java
// Se internalCode vier vazio:
String code = "INS-" + String.format("%06d",
    Math.abs(UUID.randomUUID().getLeastSignificantBits() % 1_000_000));
// Garante unicidade com loop (raro de colidir)
while (itemRepo.existsByInternalCode(code)) { ... regenara ... }
```

### 3.6 DTOs e Controller

**DTOs** em `adapter/web/dto/`:

```java
// Request — cadastro
public record CreateInventoryItemRequest(
    @NotBlank String name,
    @NotBlank String unit,
    @NotNull @Positive BigDecimal costPrice,
    String internalCode,
    String supplier,
    @NotNull @PositiveOrZero BigDecimal currentQuantity,
    @NotNull @PositiveOrZero BigDecimal minimumQuantity,
    String notes
) {}

// Request — edição (sem currentQuantity — alterada apenas via movimentação)
public record UpdateInventoryItemRequest(
    @NotBlank String name,
    @NotBlank String unit,
    @NotNull @Positive BigDecimal costPrice,
    String supplier,
    @NotNull @PositiveOrZero BigDecimal minimumQuantity,
    String notes
) {}

// Request — compra
public record RegisterPurchaseRequest(
    @NotNull @Positive BigDecimal quantity,
    @NotNull @Positive BigDecimal unitCost,
    String supplier,
    @NotNull LocalDate purchaseDate,
    @NotNull String paymentType,   // CASH | INVOICE
    LocalDate dueDate,
    String notes
) {}

// Request — saída manual
public record RegisterManualExitRequest(
    @NotNull @Positive BigDecimal quantity,
    @NotBlank String reason,       // USAGE | LOSS | ADJUSTMENT | OTHER
    String notes,
    @NotNull LocalDate exitDate
) {}

// Response — item
public record InventoryItemResponse(
    String id, String name, String internalCode, String unit,
    BigDecimal costPrice, String supplier,
    BigDecimal currentQuantity, BigDecimal minimumQuantity,
    boolean active, boolean belowMinimum, boolean outOfStock,
    String notes, String createdAt
) {}

// Response — movimentação
public record InventoryMovementResponse(
    String id, String itemId, String itemName,
    String type, String reason,
    BigDecimal quantity, BigDecimal unitCost, BigDecimal totalCost,
    String supplier, String purchaseDate, String paymentType, String dueDate,
    String financialEntryId, String notes, String createdAt
) {}
```

**`InventoryController`**:
```java
@RestController
@RequestMapping("/api/inventory/items")
@RequiredArgsConstructor
public class InventoryController {

    @PostMapping
    ResponseEntity<InventoryItemResponse> create(@RequestBody @Valid CreateInventoryItemRequest req)

    @PutMapping("/{id}")
    ResponseEntity<InventoryItemResponse> update(@PathVariable UUID id,
                                                  @RequestBody @Valid UpdateInventoryItemRequest req)

    @GetMapping
    ResponseEntity<PageResponse<InventoryItemResponse>> list(
        @RequestParam(required=false) String search,
        @RequestParam(defaultValue="ACTIVE") String status,   // ALL | ACTIVE | INACTIVE
        @RequestParam(required=false) String filter,          // LOW_STOCK
        @RequestParam(defaultValue="0") int page,
        @RequestParam(defaultValue="20") int size)

    @GetMapping("/{id}")
    ResponseEntity<InventoryItemResponse> get(@PathVariable UUID id)

    @DeleteMapping("/{id}")
    ResponseEntity<Void> delete(@PathVariable UUID id)

    @PatchMapping("/{id}/deactivate")
    ResponseEntity<InventoryItemResponse> deactivate(@PathVariable UUID id)

    @PatchMapping("/{id}/reactivate")
    ResponseEntity<InventoryItemResponse> reactivate(@PathVariable UUID id)

    @PostMapping("/{id}/purchase")
    ResponseEntity<RegisterPurchaseResponse> purchase(@PathVariable UUID id,
                                                       @RequestBody @Valid RegisterPurchaseRequest req)

    @PostMapping("/{id}/exit")
    ResponseEntity<InventoryMovementResponse> exit(@PathVariable UUID id,
                                                    @RequestBody @Valid RegisterManualExitRequest req)

    @GetMapping("/{id}/movements")
    ResponseEntity<PageResponse<InventoryMovementResponse>> movements(
        @PathVariable UUID id,
        @RequestParam(defaultValue="0") int page,
        @RequestParam(defaultValue="20") int size)
}
```

`RegisterPurchaseResponse` (record):
```java
public record RegisterPurchaseResponse(
    InventoryItemResponse item,
    InventoryMovementResponse movement,
    String financialEntryId
) {}
```

---

## 4. Frontend

### 4.1 Modelo TypeScript (`inventory.model.ts`)

```typescript
export type InventoryUnit = 'un' | 'ml' | 'g' | 'cm' | 'par';
export type MovementType = 'IN' | 'OUT';
export type MovementReason = 'PURCHASE' | 'USAGE' | 'LOSS' | 'ADJUSTMENT' | 'OTHER';
export type PurchasePaymentType = 'CASH' | 'INVOICE';
export type InventoryStatusFilter = 'ALL' | 'ACTIVE' | 'INACTIVE';

export interface InventoryItem {
  id: string;
  name: string;
  internalCode: string;
  unit: InventoryUnit;
  costPrice: number;
  supplier: string | null;
  currentQuantity: number;
  minimumQuantity: number;
  active: boolean;
  belowMinimum: boolean;
  outOfStock: boolean;
  notes: string | null;
  createdAt: string;
}

export interface InventoryMovement {
  id: string;
  itemId: string;
  itemName: string;
  type: MovementType;
  reason: MovementReason;
  quantity: number;
  unitCost: number | null;
  totalCost: number | null;
  supplier: string | null;
  purchaseDate: string | null;
  paymentType: PurchasePaymentType | null;
  dueDate: string | null;
  financialEntryId: string | null;
  notes: string | null;
  createdAt: string;
}

export interface RegisterPurchaseResult {
  item: InventoryItem;
  movement: InventoryMovement;
  financialEntryId: string;
}
```

### 4.2 NgRx Store

**State:**
```typescript
interface InventoryState {
  items: InventoryItem[];
  totalItems: number;
  selectedItem: InventoryItem | null;
  movements: InventoryMovement[];
  totalMovements: number;
  search: string;
  statusFilter: InventoryStatusFilter;
  lowStockOnly: boolean;
  page: number;
  isLoading: boolean;
  error: string | null;
}
```

**Actions:**
```
[Inventory] Load Items
[Inventory] Load Items Success/Failure
[Inventory] Create Item
[Inventory] Create Item Success/Failure
[Inventory] Update Item
[Inventory] Update Item Success/Failure
[Inventory] Delete Item
[Inventory] Delete Item Success/Failure
[Inventory] Deactivate Item
[Inventory] Deactivate Item Success/Failure
[Inventory] Reactivate Item
[Inventory] Reactivate Item Success/Failure
[Inventory] Register Purchase
[Inventory] Register Purchase Success/Failure
[Inventory] Register Exit
[Inventory] Register Exit Success/Failure
[Inventory] Load Movements
[Inventory] Load Movements Success/Failure
[Inventory] Set Search
[Inventory] Set Status Filter
[Inventory] Set Low Stock Filter
```

**Effects:**
- `setSearch$`, `setStatusFilter$`, `setLowStockFilter$` → disparam `loadItems`
- `loadItems$` → `InventoryService.list(...)`
- `createItem$`, `updateItem$` → fecha dialog + `loadItems`
- `deleteItem$`, `deactivateItem$`, `reactivateItem$` → `loadItems`
- `registerPurchase$` → fecha dialog + `loadItems` + snackbar com resumo
- `registerExit$` → fecha dialog + `loadItems`

**Seletores:** `selectItems`, `selectTotalItems`, `selectLowStockCount`, `selectIsLoading`, `selectSearch`, `selectStatusFilter`

### 4.3 Componentes

#### `InventoryComponent` (shell) — `features/inventory/`
- Barra superior: título "Estoque", botão "+ Adicionar"
- Banner de alerta: `*ngIf="lowStockCount > 0"` → "X itens com estoque baixo" (clique filtra)
- Filtros: busca (debounce 300ms), status (Ativos/Inativos/Todos), toggle "Só estoque baixo"
- Hospeda `InventoryListComponent`

#### `InventoryListComponent` — `components/inventory-list/`
Layout tabela (mesmo padrão de `ClientListComponent`):
```
┌──────┬────────────────┬───────┬─────────┬─────────┬────────────┬─────────┐
│ Cód. │ Nome           │ Unid. │ Qtd.    │ Mínimo  │ Custo unit.│ Ações   │
├──────┼────────────────┼───────┼─────────┼─────────┼────────────┼─────────┤
│ INS-1│ Cola de cílios │ un    │ [0] ●   │ 2       │ R$ 45,00   │ 🛒 ✏️ ⊖ 🗑 │
│ INS-2│ Fio C 0.07     │ un    │ [1] ▲   │ 3       │ R$ 28,00   │ 🛒 ✏️ ⊖ 🗑 │
│ INS-3│ Primer         │ ml    │ 15      │ 10      │ R$ 12,00   │ 🛒 ✏️ ⊖ 🗑 │
└──────┴────────────────┴───────┴─────────┴─────────┴────────────┴─────────┘
```
- `●` vermelho = sem estoque (`outOfStock`)
- `▲` laranja = estoque baixo (`belowMinimum`)
- Ações: Registrar compra (🛒) | Editar (✏️) | Desativar/Reativar (⊖/↺) | Excluir (🗑)
- `MatPaginator` no rodapé
- Linha clicável abre o histórico de movimentações (dialog)

#### `InventoryFormComponent` — `components/inventory-form/` (MatDialog)
Data: `{ item: InventoryItem | null }` (null = novo)
- Reactive Form com todos os campos do `CreateInventoryItemRequest`
- `internalCode` com placeholder "gerado automaticamente"
- `costPrice` com `currencyBr` pipe na exibição, `MatInput` numérico
- Ao salvar: despacha `createItem` ou `updateItem`

#### `InventoryPurchaseDialogComponent` — `components/inventory-purchase/` (MatDialog)
Data: `{ item: InventoryItem }`
- Título: "Registrar compra — {nome do item}"
- Campos: qtd, preço unitário (pré-preenchido), fornecedor (pré-preenchido), data, forma de pagamento
- Campo "Data de vencimento" aparece somente quando `paymentType = INVOICE`
- Total calculado em tempo real: `quantity × unitCost`
- Resumo antes de confirmar: "X unidades • R$ Y • {À vista/Faturado até DD/MM}"
- Ao salvar: despacha `registerPurchase`; efeito exibe snackbar com o resumo

#### `InventoryExitDialogComponent` — `components/inventory-exit/` (MatDialog)
Data: `{ item: InventoryItem }`
- Campos: qtd, motivo (select), observação, data
- Aviso inline `*ngIf="willGoNegative"` — sem bloquear o botão
- Ao salvar: despacha `registerExit`

#### `InventoryHistoryDialogComponent` — `components/inventory-history/` (MatDialog) [P2]
Data: `{ item: InventoryItem }`
- Lista movimentações paginada em ordem reversa
- Linha de compra: badge IN azul + link para Financeiro se `financialEntryId != null`
- Linha de saída: badge OUT cinza + motivo + saldo resultante

### 4.4 `InventoryService` (HTTP)

```typescript
@Injectable({ providedIn: 'root' })
export class InventoryService {
  private readonly http = inject(HttpClient);
  private readonly base = '/api/inventory/items';

  list(search: string, status: InventoryStatusFilter,
       lowStockOnly: boolean, page = 0, size = 20): Observable<PageResponse<InventoryItem>>

  get(id: string): Observable<InventoryItem>

  create(req: CreateInventoryItemRequest): Observable<InventoryItem>

  update(id: string, req: UpdateInventoryItemRequest): Observable<InventoryItem>

  delete(id: string): Observable<void>

  deactivate(id: string): Observable<InventoryItem>

  reactivate(id: string): Observable<InventoryItem>

  registerPurchase(id: string, req: RegisterPurchaseRequest): Observable<RegisterPurchaseResult>

  registerExit(id: string, req: RegisterManualExitRequest): Observable<InventoryMovement>

  listMovements(id: string, page = 0, size = 20): Observable<PageResponse<InventoryMovement>>
}
```

### 4.5 Rotas

```typescript
// inventory.routes.ts — já existe, manter como está
export const inventoryRoutes: Routes = [
  { path: '', component: InventoryComponent }
];
```

A rota `/inventory` já deve estar registrada no `app.routes.ts` (o skeleton existe) — verificar na execução.

---

## 5. Reuso de Código Existente

| O que reutilizar | Onde existe | Como usar |
|---|---|---|
| `currencyBr` pipe | `shared/pipes/` | Preço de custo na tabela e formulário |
| `datePtbr` pipe | `shared/pipes/` | Datas de compra e vencimento |
| `PageResponse<T>` | padrão existente | Paginação de itens e movimentações |
| `MatTable + MatPaginator` | Clientes / Serviços | Mesmo padrão de tabela |
| Busca com debounce (300ms) | Clientes | `Subject + debounceTime` no shell |
| `MatDialog` para formulário | Financeiro / Agendamentos | Dialog de cadastro/edição |
| `GlobalExceptionHandler` | backend existente | Adicionar os 3 novos 409s |
| `FinancialEntryRepository` | infra/financeiro | Injetado diretamente em `RegisterPurchaseUseCaseImpl` |

---

## 6. Tratamento de Erros

| Cenário | HTTP | Mensagem ao usuário |
|---|---|---|
| Item não encontrado | 404 | "Item não encontrado" |
| Código interno duplicado | 409 | "Já existe um item com este código" |
| Excluir item com movimentações | 409 | "Este item possui movimentações — desative-o em vez de excluir" |
| Erro ao criar despesa financeira | 500 | "Compra registrada mas falha ao criar despesa — verifique o Financeiro" |
| Saída deixa estoque negativo | — | Aviso no frontend, não bloqueia backend |

---

## 7. Decisões de Design

| Decisão | Escolha | Razão |
|---|---|---|
| `RegisterPurchaseUseCase` injeta `FinancialEntryRepository` diretamente | Pragmatismo | Segue o padrão já adotado no projeto; criar port/in inter-módulo seria over-engineering para este porte |
| `@Transactional` em `RegisterPurchaseUseCaseImpl` | Necessário | Estoque + movimentação + financeiro devem ser atômicos — qualquer falha faz rollback de tudo |
| `currentQuantity` só alterada via movimentações (não editável diretamente) | Rastreabilidade | Qualquer alteração de quantidade deve ter histórico; edição direta quebraria a auditoria |
| Histórico como dialog (não rota) | Consistência UX | Padrão já estabelecido com `AppointmentPopupComponent` e dialogs do Financeiro |
| `supplier` como texto livre no item e na movimentação | Fase 2 | Entidade Fornecedor fica para Fase 3; texto livre cobre 100% dos casos de uso atuais |
| `InventoryMovementEntity` com `@ManyToOne` read-only ao item | Performance | Permite resolver `itemName` via JOIN sem query extra — mesmo padrão de `FinancialEntryEntity` com `appointment` |
| `SUPPLY` adicionado a `FinancialEntryExpenseType` (não categoria separada) | Integração natural | O módulo Financeiro já filtra por `expenseType`; SUPPLY aparece automaticamente na aba "Variáveis" ou cria aba própria se necessário |
