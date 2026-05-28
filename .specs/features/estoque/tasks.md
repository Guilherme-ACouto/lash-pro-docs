# Estoque — Tasks

**Design**: `.specs/features/estoque/design.md`
**Status**: Draft

---

## Plano de Execução

### Fase 1 — Fundação do Domain (paralelo)

```
B01 ──┐
B02 ──┼──→ Fase 2
B03 ──┘
```

### Fase 2 — Ports (paralelo após Fase 1)

```
B02 ──→ B04 ──┐
B02 ──→ B05 ──┼──→ Fase 3
```

### Fase 3 — Entidades JPA (paralelo após Fase 2)

```
B01 + B05 ──→ B06 ──┐
B01 + B05 ──→ B08 ──┼──→ Fase 4
```

### Fase 4 — Repository Impls (paralelo após Fase 3)

```
B05 + B06 ──→ B07 ──┐
B05 + B08 ──→ B09 ──┼──→ Fase 5
```

### Fase 5 — Use Cases (paralelo após Fase 4)

```
B04 + B07 ──────────────── B10 ──┐
B04 + B07 + B09 + FinancialRepo ─ B11 ──┼──→ Fase 6
B04 + B07 + B09 ──────────────── B12 ──┘
```

### Fase 6 — DTOs, Handler e Controller (sequencial)

```
B04 ──→ B13 ──┐
B02 ──→ B15 ──┤
              └──→ B14 (B10 + B11 + B12 + B13 + B15)
```

### Fase 7 — Frontend (sequencial, após Fase 6)

```
F01 ──→ F02
F01 ──→ F03 ──→ F04

F03 ──→ F05 ──┐
F03 ──→ F06 ──┤
F03 ──→ F07 ──┤──→ F09
F03 ──→ F08 ──┘
F04 ──────────┘

F02 + F03 ──→ F10  [P2]
              F11  [P2]
```

---

## Breakdown das Tarefas

### B01: Migration V6 — schema de estoque completo

**O que**: Criar `V6__inventory_full_schema.sql` com os ALTERs em `inventory_items`, `inventory_movements` e a adição de `SUPPLY` ao constraint de `financial_entries`
**Onde**: `lash-backend/src/main/resources/db/migration/V6__inventory_full_schema.sql`
**Depende de**: Nenhum
**Reutiliza**: Padrão das migrations V3–V5
**Requisitos**: INV-01, INV-13

**Feito quando:**
- [ ] Colunas `internal_code`, `cost_price`, `supplier`, `notes` adicionadas a `inventory_items`
- [ ] Constraint `uq_inventory_code` criada em `internal_code`
- [ ] Colunas `reason`, `unit_cost`, `supplier`, `purchase_date`, `payment_type`, `due_date`, `financial_entry_id` adicionadas a `inventory_movements`
- [ ] Constraints `chk_movement_reason` e `chk_payment_type` criadas
- [ ] Constraint `chk_expense_type` de `financial_entries` atualizada para incluir `'SUPPLY'`
- [ ] Índices `idx_inventory_movements_item` e `idx_inventory_movements_date` criados
- [ ] Gate: `mvn compile` sem erros (Flyway valida o SQL na inicialização)

**Verificar**: `mvn compile` sem erros de validação Flyway

---

### B02: Domain — InventoryItem (estender) + InventoryMovement + enums + exceções

**O que**: Estender `InventoryItem` com 4 novos campos; criar `InventoryMovement`; criar 3 enums (`MovementType`, `MovementReason`, `PurchasePaymentType`); criar 3 exceções (`InventoryItemNotFoundException`, `InventoryItemCodeAlreadyExistsException`, `InventoryItemHasMovementsException`)
**Onde**:
- `domain/model/InventoryItem.java` (modificar)
- `domain/model/InventoryMovement.java` (novo)
- `domain/model/MovementType.java` (novo)
- `domain/model/MovementReason.java` (novo)
- `domain/model/PurchasePaymentType.java` (novo)
- `domain/exception/InventoryItemNotFoundException.java` (novo)
- `domain/exception/InventoryItemCodeAlreadyExistsException.java` (novo)
- `domain/exception/InventoryItemHasMovementsException.java` (novo)
**Depende de**: Nenhum
**Reutiliza**: Padrão de `Client.java`, `FinancialEntry.java`, `ClientNotFoundException.java`
**Requisitos**: INV-01, INV-03, INV-04, INV-11, INV-12

**Feito quando:**
- [ ] `InventoryItem` tem campos: `internalCode`, `costPrice`, `supplier`, `notes` + método `isOutOfStock()`
- [ ] `InventoryMovement` tem todos os campos do design (incluindo `financialEntryId`)
- [ ] 3 enums criados com os valores corretos
- [ ] 3 exceções criadas estendendo `BusinessException` com mensagem e status HTTP corretos
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B03: Domain — `FinancialEntryExpenseType` — adicionar `SUPPLY`

**O que**: Adicionar o valor `SUPPLY` ao enum existente `FinancialEntryExpenseType`
**Onde**: `domain/model/FinancialEntryExpenseType.java`
**Depende de**: Nenhum
**Reutiliza**: Enum existente
**Requisitos**: INV-15

**Feito quando:**
- [ ] `SUPPLY` adicionado ao enum
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B04: Ports/In — interfaces dos 9 use cases

**O que**: Criar as 9 interfaces de port/in para o módulo de estoque
**Onde**: `domain/port/in/` — 9 arquivos novos:
- `CreateInventoryItemUseCase.java`
- `UpdateInventoryItemUseCase.java`
- `DeleteInventoryItemUseCase.java`
- `ListInventoryItemsUseCase.java`
- `GetInventoryItemUseCase.java`
- `DeactivateInventoryItemUseCase.java`
- `RegisterPurchaseUseCase.java`
- `RegisterManualExitUseCase.java`
- `ListMovementsUseCase.java`
**Depende de**: B02
**Reutiliza**: Padrão de `CreateClientUseCase.java`, `ListClientsUseCase.java`
**Requisitos**: INV-01 a INV-26

**Feito quando:**
- [ ] Cada interface define command record interno (ou parâmetros) e tipo de retorno conforme design
- [ ] `RegisterPurchaseUseCase` retorna `RegisterPurchaseResult` (record com item + movement + financialEntryId)
- [ ] `DeactivateInventoryItemUseCase` recebe `boolean activate` para cobrir reativar também
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B05: Ports/Out — `InventoryItemRepository` e `InventoryMovementRepository`

**O que**: Criar as 2 interfaces de port/out
**Onde**: `domain/port/out/` — 2 arquivos novos:
- `InventoryItemRepository.java`
- `InventoryMovementRepository.java`
**Depende de**: B02
**Reutiliza**: Padrão de `ClientRepository.java`, `FinancialEntryRepository.java`
**Requisitos**: INV-01, INV-07, INV-11, INV-12, INV-13, INV-21

**Feito quando:**
- [ ] `InventoryItemRepository` tem: `save`, `findById`, `listWithFilters(search, active, onlyLowStock, pageable)`, `existsByInternalCode`, `existsByInternalCodeAndIdNot`, `delete`
- [ ] `InventoryMovementRepository` tem: `save`, `findByItemId(itemId, pageable)`, `existsByItemId`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B06: Infra — `InventoryItemEntity` + `InventoryItemJpaRepository`

**O que**: Criar a entidade JPA e o repositório Spring Data para itens de estoque
**Onde**: `infrastructure/persistence/entity/InventoryItemEntity.java` + `infrastructure/persistence/repository/InventoryItemJpaRepository.java`
**Depende de**: B01, B05
**Reutiliza**: Padrão de `ClientEntity.java` e `ClientJpaRepository.java`
**Requisitos**: INV-01, INV-05, INV-07

**Feito quando:**
- [ ] `InventoryItemEntity` mapeada para `inventory_items` com todas as colunas do V6
- [ ] `InventoryItemJpaRepository` tem query JPQL de `findWithFilters` (busca + active + low stock)
- [ ] Parâmetro `search` tratado com `LOWER(CONCAT('%',:search,'%'))` — não passa null para CONCAT (padrão do projeto)
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B07: Infra — `InventoryItemMapper` + `InventoryItemRepositoryImpl`

**O que**: Criar o mapper domain ↔ entity e a implementação do `InventoryItemRepository`
**Onde**: `infrastructure/persistence/mapper/InventoryItemMapper.java` + `infrastructure/persistence/repository/InventoryItemRepositoryImpl.java`
**Depende de**: B05, B06
**Reutiliza**: Padrão de `ClientMapper.java` e `ClientRepositoryImpl.java`
**Requisitos**: INV-01, INV-02, INV-05, INV-09, INV-11, INV-12

**Feito quando:**
- [ ] `InventoryItemMapper` converte `InventoryItemEntity` ↔ `InventoryItem` (todos os campos)
- [ ] `InventoryItemRepositoryImpl` implementa todos os métodos do port/out
- [ ] `listWithFilters`: delega para JpaRepository + mapeia resultado
- [ ] `existsByInternalCode` e `existsByInternalCodeAndIdNot` delegam para JPA
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B08: Infra — `InventoryMovementEntity` + `InventoryMovementJpaRepository`

**O que**: Criar a entidade JPA e o repositório Spring Data para movimentações
**Onde**: `infrastructure/persistence/entity/InventoryMovementEntity.java` + `infrastructure/persistence/repository/InventoryMovementJpaRepository.java`
**Depende de**: B01, B05
**Reutiliza**: Padrão de `FinancialEntryEntity.java` (associação read-only com `@ManyToOne insertable=false`)
**Requisitos**: INV-13, INV-14, INV-16, INV-18, INV-19, INV-21

**Feito quando:**
- [ ] `InventoryMovementEntity` mapeada para `inventory_movements` com todos os campos do V6
- [ ] Associação `@ManyToOne(fetch=LAZY)` read-only ao `InventoryItemEntity` para resolver `itemName`
- [ ] `InventoryMovementJpaRepository` tem `findByItemIdOrderByCreatedAtDesc` e `existsByItemId`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B09: Infra — `InventoryMovementMapper` + `InventoryMovementRepositoryImpl`

**O que**: Criar o mapper e implementação do `InventoryMovementRepository`
**Onde**: `infrastructure/persistence/mapper/InventoryMovementMapper.java` + `infrastructure/persistence/repository/InventoryMovementRepositoryImpl.java`
**Depende de**: B05, B08
**Reutiliza**: Padrão de `InventoryItemMapper` (B07) e `ClientRepositoryImpl`
**Requisitos**: INV-13, INV-18, INV-21, INV-22

**Feito quando:**
- [ ] `InventoryMovementMapper` converte entity ↔ domain; resolve `itemName` via associação read-only
- [ ] `totalCost` calculado no mapper: `quantity.multiply(unitCost)` quando `unitCost != null`
- [ ] `InventoryMovementRepositoryImpl` implementa `save`, `findByItemId`, `existsByItemId`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B10: Use Cases — CRUD (Create, Update, Delete, List, Get, Deactivate)

**O que**: Implementar 6 use cases de operações básicas no item de estoque
**Onde**: `application/usecase/` — 6 arquivos novos:
- `CreateInventoryItemUseCaseImpl.java`
- `UpdateInventoryItemUseCaseImpl.java`
- `DeleteInventoryItemUseCaseImpl.java`
- `ListInventoryItemsUseCaseImpl.java`
- `GetInventoryItemUseCaseImpl.java`
- `DeactivateInventoryItemUseCaseImpl.java`
**Depende de**: B04, B07
**Reutiliza**: Padrão de `CreateClientUseCaseImpl`, `DeleteClientUseCaseImpl`
**Requisitos**: INV-01 a INV-12, INV-17 a INV-18, INV-23 a INV-24

**Feito quando:**
- [ ] `CreateInventoryItemUseCaseImpl`: gera `internalCode` automaticamente se ausente; verifica unicidade do código; salva via `InventoryItemRepository`
- [ ] `UpdateInventoryItemUseCaseImpl`: não permite alterar `currentQuantity` diretamente; verifica unicidade do novo código
- [ ] `DeleteInventoryItemUseCaseImpl`: lança `InventoryItemHasMovementsException` se `movementRepo.existsByItemId(id)` for true
- [ ] `ListInventoryItemsUseCaseImpl`: converte `status` (ALL/ACTIVE/INACTIVE) e `filter` (LOW_STOCK) para parâmetros do repositório
- [ ] `GetInventoryItemUseCaseImpl`: lança `InventoryItemNotFoundException` se não encontrado
- [ ] `DeactivateInventoryItemUseCaseImpl`: recebe `activate: boolean`; atualiza campo `active`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B11: Use Case — `RegisterPurchaseUseCaseImpl`

**O que**: Implementar o use case de registro de compra — atualiza estoque + cria movimentação + cria `FinancialEntry` atomicamente
**Onde**: `application/usecase/RegisterPurchaseUseCaseImpl.java`
**Depende de**: B04, B07, B09 (+ `FinancialEntryRepository` já existe)
**Reutiliza**: `FinancialEntryRepository` (port/out existente), padrão `@Transactional` de outros use cases
**Requisitos**: INV-13, INV-14, INV-15, INV-16, INV-17

**Feito quando:**
- [ ] Classe anotada com `@Transactional` (import `org.springframework.transaction.annotation.Transactional`)
- [ ] Soma `quantity` ao `currentQuantity` do item; atualiza `costPrice` com `unitCost` da compra; atualiza `supplier` se fornecido
- [ ] Cria `InventoryMovement` com `type=IN`, `reason=PURCHASE` e todos os campos de compra
- [ ] Cria `FinancialEntry` com `type=EXPENSE`, `expenseType=SUPPLY`, `description="Compra: {item.name}"`, `amount=qty×unitCost`
- [ ] Se `paymentType=CASH`: `status=PAID`, `paymentDate=purchaseDate`, `dueDate=purchaseDate`
- [ ] Se `paymentType=INVOICE`: `status=PENDING`, `paymentDate=null`, `dueDate=cmd.dueDate()`
- [ ] Salva movimentação com `financialEntryId` da entry criada
- [ ] Retorna `RegisterPurchaseResult(item, movement, financialEntryId)`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B12: Use Cases — `RegisterManualExitUseCaseImpl` + `ListMovementsUseCaseImpl`

**O que**: Implementar 2 use cases restantes — saída manual e listagem de histórico
**Onde**: `application/usecase/` — 2 arquivos novos
**Depende de**: B04, B07, B09
**Reutiliza**: Padrão de B10/B11
**Requisitos**: INV-18, INV-19, INV-20, INV-21, INV-22

**Feito quando:**
- [ ] `RegisterManualExitUseCaseImpl`: subtrai `quantity` do `currentQuantity`; cria `InventoryMovement` com `type=OUT`, `reason=cmd.reason()`; **não** cria `FinancialEntry`
- [ ] `ListMovementsUseCaseImpl`: retorna `Page<InventoryMovementResult>` ordenada por `createdAt` DESC
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B13: DTOs — todos os records de request e response

**O que**: Criar todos os DTOs do módulo de estoque
**Onde**: `adapter/web/dto/` — 7 arquivos novos:
- `CreateInventoryItemRequest.java`
- `UpdateInventoryItemRequest.java`
- `InventoryItemResponse.java`
- `RegisterPurchaseRequest.java`
- `RegisterManualExitRequest.java`
- `InventoryMovementResponse.java`
- `RegisterPurchaseResponse.java`
**Depende de**: B04
**Reutiliza**: Padrão de `CreateClientRequest.java`, `ClientResponse.java`
**Requisitos**: INV-01, INV-09, INV-13, INV-18, INV-21

**Feito quando:**
- [ ] `InventoryItemResponse` inclui campos derivados: `belowMinimum`, `outOfStock`
- [ ] `RegisterPurchaseRequest` tem validação `@NotNull` em `paymentType`; `dueDate` sem `@NotNull` (validação de negócio no use case)
- [ ] `RegisterPurchaseResponse` agrega `item`, `movement` e `financialEntryId`
- [ ] Todos os campos de data como `String` (formato ISO — padrão do projeto)
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B14: `InventoryController` — 9 endpoints

**O que**: Criar o controller REST com todos os endpoints do módulo de estoque e registrar os use cases no contexto Spring
**Onde**: `adapter/web/controller/InventoryController.java`
**Depende de**: B10, B11, B12, B13, B15
**Reutiliza**: Padrão de `ClientController.java`, `FinancialController.java`
**Requisitos**: INV-01 a INV-26

**Feito quando:**
- [ ] `@RestController @RequestMapping("/api/inventory/items")` com `@RequiredArgsConstructor`
- [ ] 9 endpoints implementados conforme design: POST, PUT/{id}, GET, GET/{id}, DELETE/{id}, PATCH/{id}/deactivate, PATCH/{id}/reactivate, POST/{id}/purchase, POST/{id}/exit, GET/{id}/movements
- [ ] `GET /api/inventory/items` aceita `search`, `status` (default `ACTIVE`), `filter`, `page`, `size`
- [ ] `POST /{id}/purchase` retorna `RegisterPurchaseResponse` com status 201
- [ ] Todos os use cases injetados via construtor
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B15: `GlobalExceptionHandler` — handlers para exceções de estoque

**O que**: Adicionar 3 novos `@ExceptionHandler` para as exceções do módulo de estoque
**Onde**: `adapter/web/controller/GlobalExceptionHandler.java` (modificar)
**Depende de**: B02
**Reutiliza**: Padrão dos handlers existentes (404 e 409)
**Requisitos**: INV-03, INV-11, INV-12

**Feito quando:**
- [ ] `InventoryItemNotFoundException` → 404 com mensagem do design
- [ ] `InventoryItemCodeAlreadyExistsException` → 409
- [ ] `InventoryItemHasMovementsException` → 409 com mensagem "Este item possui movimentações — desative-o em vez de excluir"
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### F01: `inventory.model.ts` — interfaces TypeScript

**O que**: Criar o arquivo de modelos TypeScript do módulo de estoque
**Onde**: `lash-frontend/src/app/features/inventory/models/inventory.model.ts`
**Depende de**: Nenhum
**Reutiliza**: Padrão de `financial.model.ts`
**Requisitos**: INV-01, INV-13, INV-18, INV-21

**Feito quando:**
- [ ] Interfaces `InventoryItem`, `InventoryMovement`, `RegisterPurchaseResult` definidas conforme design
- [ ] Types: `InventoryUnit`, `MovementType`, `MovementReason`, `PurchasePaymentType`, `InventoryStatusFilter`
- [ ] Interfaces de request: `CreateInventoryItemRequest`, `UpdateInventoryItemRequest`, `RegisterPurchaseRequest`, `RegisterManualExitRequest`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F02: `InventoryService` — HTTP client

**O que**: Criar o serviço Angular de comunicação com a API de estoque
**Onde**: `lash-frontend/src/app/features/inventory/services/inventory.service.ts`
**Depende de**: F01
**Reutiliza**: Padrão de `client.service.ts`, `financial.service.ts`
**Requisitos**: INV-01, INV-05, INV-07, INV-13, INV-18, INV-21

**Feito quando:**
- [ ] `inject(HttpClient)` — sem injeção por construtor (padrão do projeto)
- [ ] 10 métodos: `list`, `get`, `create`, `update`, `delete`, `deactivate`, `reactivate`, `registerPurchase`, `registerExit`, `listMovements`
- [ ] URL base `/api/inventory/items`
- [ ] `list()` passa `search`, `status`, `filter`, `page`, `size` como query params
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F03: NgRx — actions, reducer e selectors

**O que**: Criar as actions, o reducer e os selectors do slice de inventory
**Onde**: `lash-frontend/src/app/features/inventory/store/`
- `inventory.actions.ts`
- `inventory.reducer.ts`
- `inventory.selectors.ts`
**Depende de**: F01
**Reutiliza**: Padrão de `financial.actions.ts`, `financial.reducer.ts`, `financial.selectors.ts`
**Requisitos**: INV-01 a INV-26

**Feito quando:**
- [ ] Actions criadas para todos os fluxos: Load Items, Create/Update/Delete, Deactivate/Reactivate, RegisterPurchase, RegisterExit, LoadMovements, SetSearch, SetStatusFilter, SetLowStockFilter (+ Success/Failure variants)
- [ ] `InventoryState` definida conforme design (items, totalItems, selectedItem, movements, search, statusFilter, lowStockOnly, page, isLoading, error)
- [ ] Reducer inicializa estado e trata todos os Success/Failure
- [ ] Seletores: `selectItems`, `selectTotalItems`, `selectLowStockCount` (derivado de `items`), `selectIsLoading`, `selectSearch`, `selectStatusFilter`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F04: NgRx — effects

**O que**: Criar os effects para orquestrar chamadas HTTP e side effects
**Onde**: `lash-frontend/src/app/features/inventory/store/inventory.effects.ts`
**Depende de**: F02, F03
**Reutiliza**: Padrão de `financial.effects.ts`, `client.effects.ts`
**Requisitos**: INV-01 a INV-26

**Feito quando:**
- [ ] `loadItems$`: chama `InventoryService.list()` com params do state
- [ ] `setSearch$`, `setStatusFilter$`, `setLowStockFilter$`: disparam `loadItems`
- [ ] `createItem$`, `updateItem$`: fecham dialog (`MatDialogRef`) + disparam `loadItems`
- [ ] `deleteItem$`, `deactivateItem$`, `reactivateItem$`: disparam `loadItems`
- [ ] `registerPurchase$`: fecha dialog + dispara `loadItems` + exibe `MatSnackBar` com resumo "X unidades adicionadas • R$ Y criado no Financeiro"
- [ ] `registerExit$`: fecha dialog + dispara `loadItems`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F05: `InventoryListComponent` — tabela com badges e paginação

**O que**: Criar o componente de lista de itens de estoque com busca, badges de alerta e ações
**Onde**: `lash-frontend/src/app/features/inventory/components/inventory-list/`
- `inventory-list.component.ts`
- `inventory-list.component.html`
- `inventory-list.component.css`
**Depende de**: F03
**Reutiliza**: Padrão de `client-list.component` (MatTable, MatPaginator, busca com debounce); pipes `currencyBr`, `datePtbr`
**Requisitos**: INV-05, INV-06, INV-07, INV-08, INV-23, INV-24

**Feito quando:**
- [ ] Tabela com colunas: Código, Nome, Unidade, Qtd. atual, Mínimo, Custo unit., Ações
- [ ] Badge vermelho "Sem estoque" quando `outOfStock = true`
- [ ] Badge laranja "Estoque baixo" quando `belowMinimum = true` (e não sem estoque)
- [ ] Coluna Custo unit. usa pipe `currencyBr`
- [ ] 4 ações por linha: Registrar compra (ícone `shopping_cart`), Editar (ícone `edit`), Desativar/Reativar (ícone `toggle_off`/`toggle_on`), Excluir (ícone `delete`)
- [ ] Botões Desativar/Reativar respeitam `item.active` para trocar ícone e label
- [ ] `MatPaginator` com tamanho padrão 20
- [ ] Outputs: `purchaseClick`, `editClick`, `deactivateClick`, `deleteClick`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F06: `InventoryFormComponent` — dialog de cadastro e edição

**O que**: Criar o dialog de formulário para cadastrar e editar itens de estoque
**Onde**: `lash-frontend/src/app/features/inventory/components/inventory-form/`
- `inventory-form.component.ts`
- `inventory-form.component.html`
- `inventory-form.component.css`
**Depende de**: F03
**Reutiliza**: Padrão de `financial-form.component`; `ReactiveFormsModule`, `MatFormFieldModule`, `MatInputModule`, `MatSelectModule`
**Requisitos**: INV-01, INV-02, INV-03, INV-04, INV-09, INV-10

**Feito quando:**
- [ ] `MAT_DIALOG_DATA` recebe `{ item: InventoryItem | null }` — null = novo
- [ ] Formulário reativo com todos os campos: `name`, `internalCode`, `unit`, `costPrice`, `supplier`, `currentQuantity` (só no create, disabled no edit), `minimumQuantity`, `notes`
- [ ] `unit` usa `mat-select` com opções: un / ml / g / cm / par
- [ ] `internalCode` com placeholder "Gerado automaticamente"
- [ ] `costPrice` e `currentQuantity`/`minimumQuantity` com validação `min: 0.01` / `min: 0`
- [ ] Ao salvar: despacha `createItem` ou `updateItem` conforme contexto
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F07: `InventoryPurchaseDialogComponent` — dialog de registrar compra

**O que**: Criar o dialog de registro de compra com integração financeira visível ao usuário
**Onde**: `lash-frontend/src/app/features/inventory/components/inventory-purchase/`
- `inventory-purchase.component.ts`
- `inventory-purchase.component.html`
- `inventory-purchase.component.css`
**Depende de**: F03
**Reutiliza**: Padrão de `FinancialFormComponent`; `MatDatepickerModule` para datas; pipe `currencyBr`
**Requisitos**: INV-13, INV-14, INV-15, INV-16, INV-17

**Feito quando:**
- [ ] `MAT_DIALOG_DATA` recebe `{ item: InventoryItem }`
- [ ] Título: "Registrar compra — {item.name}"
- [ ] Campos: `quantity`, `unitCost` (pré-preenchido com `item.costPrice`), `supplier` (pré-preenchido com `item.supplier`), `purchaseDate` (MatDatepicker, default hoje), `paymentType` (mat-select: À vista / Faturado)
- [ ] Campo `dueDate` (MatDatepicker) aparece com `*ngIf="paymentType === 'INVOICE'"`
- [ ] Total calculado em tempo real: `quantity × unitCost` exibido abaixo dos campos
- [ ] Resumo antes do botão confirmar: "X unidades • R$ Y • À vista" ou "X unidades • R$ Y • Faturado até DD/MM/YYYY"
- [ ] Ao salvar: despacha `registerPurchase({ itemId, ...req })`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F08: `InventoryExitDialogComponent` — dialog de saída manual

**O que**: Criar o dialog de saída manual de estoque
**Onde**: `lash-frontend/src/app/features/inventory/components/inventory-exit/`
- `inventory-exit.component.ts`
- `inventory-exit.component.html`
- `inventory-exit.component.css`
**Depende de**: F03
**Reutiliza**: Padrão de dialogs existentes; `MatSelectModule`
**Requisitos**: INV-18, INV-19, INV-20

**Feito quando:**
- [ ] `MAT_DIALOG_DATA` recebe `{ item: InventoryItem }`
- [ ] Campos: `quantity`, `reason` (mat-select: Uso / Perda / Ajuste / Outro), `notes`, `exitDate` (MatDatepicker, default hoje)
- [ ] Aviso amarelo inline `*ngIf="willGoNegative"`: "⚠️ Estoque ficará negativo após esta saída" — não bloqueia confirmação
- [ ] `willGoNegative` calculado em tempo real: `quantity > item.currentQuantity`
- [ ] Ao salvar: despacha `registerExit({ itemId, ...req })`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F09: `InventoryComponent` (shell) + rotas + banner de alertas

**O que**: Montar o componente shell integrando todos os sub-componentes; adicionar banner de estoque baixo; registrar a feature no app e na sidebar
**Onde**:
- `lash-frontend/src/app/features/inventory/inventory.component.ts` (modificar — já é skeleton)
- `lash-frontend/src/app/features/inventory/inventory.component.html` (modificar)
- `lash-frontend/src/app/features/inventory/inventory.component.css` (modificar)
- `lash-frontend/src/app/features/inventory/inventory.routes.ts` (já existe — verificar se precisa ajuste)
**Depende de**: F04, F05, F06, F07, F08
**Reutiliza**: Padrão de `clients.component`, layout da sidebar com título + botão adicionar
**Requisitos**: INV-05 a INV-26

**Feito quando:**
- [ ] Shell carrega `items` e `lowStockCount` no `ngOnInit` via store
- [ ] Banner `*ngIf="(lowStockCount$ | async) > 0"` exibe "X itens com estoque baixo" — clique define `lowStockOnly = true`
- [ ] Filtros: campo busca (debounce 300ms), mat-select status, toggle "Só estoque baixo"
- [ ] Botão "+ Adicionar" abre `InventoryFormComponent` via `MatDialog`
- [ ] `InventoryListComponent` recebe `items$`, `isLoading$` como inputs; outputs conectados aos dialogs corretos
- [ ] Rota `/inventory` lazy-loaded registrada em `app.routes.ts` (verificar se já consta e adicionar se não)
- [ ] Item "Estoque" na sidebar e bottom nav aponta para `/inventory`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F10: `InventoryHistoryDialogComponent` — histórico de movimentações [P2]

**O que**: Criar o dialog de histórico de movimentações de um item
**Onde**: `lash-frontend/src/app/features/inventory/components/inventory-history/`
- `inventory-history.component.ts`
- `inventory-history.component.html`
- `inventory-history.component.css`
**Depende de**: F02, F03
**Reutiliza**: Padrão de dialogs com lista; pipe `currencyBr`, `datePtbr`
**Requisitos**: INV-21, INV-22

**Feito quando:**
- [ ] `MAT_DIALOG_DATA` recebe `{ item: InventoryItem }`
- [ ] Lista movimentações em ordem reversa com paginação
- [ ] Linha de compra: badge azul "Entrada", mostra qtd, preço unit., total, fornecedor, forma pgto
- [ ] Link "Ver no Financeiro" exibido quando `financialEntryId != null`
- [ ] Linha de saída: badge cinza "Saída", mostra qtd, motivo
- [ ] Estado vazio: "Nenhuma movimentação registrada"
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F11: Dashboard — card de alerta de estoque baixo [P2]

**O que**: Adicionar ao Dashboard um card de alerta com contagem de itens com estoque baixo e link para `/inventory?filter=LOW_STOCK`
**Onde**:
- `lash-backend` — endpoint `GET /api/dashboard` já deve incluir `lowStockItemCount` (adicionar ao `DashboardResponse`)
- `lash-frontend/src/app/features/dashboard/` — `dashboard.model.ts` + componente ou KPI card existente
**Depende de**: B14 (backend com inventory funcionando)
**Reutiliza**: `KpiCardComponent` existente no Dashboard
**Requisitos**: INV-26

**Feito quando:**
- [ ] Backend: `DashboardRepositoryImpl` adiciona query `SELECT COUNT(*) FROM inventory_items WHERE active = true AND current_quantity <= minimum_quantity`; `DashboardResponse` inclui `lowStockItemCount: int`
- [ ] Frontend: `dashboard.model.ts` inclui `lowStockItemCount`; `DashboardComponent` exibe card "Estoque baixo" com contagem; clique navega para `/inventory?filter=LOW_STOCK`
- [ ] Card não aparece quando `lowStockItemCount = 0`
- [ ] Gate: `mvn compile` + `npm run build` sem erros

**Verificar**: `mvn compile` + `npm run build`

---

## Mapa de Paralelismo

```
BACKEND
Phase 1:  B01 ─┐
          B02 ─┤ (todos em paralelo)
          B03 ─┘

Phase 2:  B04 ─┐ (em paralelo, após B02)
          B05 ─┘

Phase 3:  B06 ─┐ (em paralelo, após B01+B05)
          B08 ─┘

Phase 4:  B07 ─┐ (em paralelo, após B05+B06 / B05+B08)
          B09 ─┘

Phase 5:  B10 ─┐
          B11 ─┤ (em paralelo, após B04+B07 / B04+B07+B09)
          B12 ─┘

Phase 6:  B13 ─┐ (B13 e B15 em paralelo)
          B15 ─┘
          B14   (após B10+B11+B12+B13+B15)

FRONTEND (após Fase 6 backend)
Phase 7a: F01

Phase 7b: F02 ─┐ (em paralelo, após F01)
          F03 ─┘

Phase 7c: F04   (após F02+F03)

Phase 7d: F05 ─┐
          F06 ─┤ (em paralelo, após F03)
          F07 ─┤
          F08 ─┘

Phase 7e: F09   (após F04+F05+F06+F07+F08)

Phase 7f: F10 ─┐ (P2 — após F09)
          F11 ─┘
```

---

## Validação de Granularidade

| Task | Escopo | Status |
|---|---|---|
| B01: Migration V6 | 1 arquivo SQL | ✅ |
| B02: Domain models + enums + exceções | 8 arquivos — coesos (domain layer inteira) | ✅ |
| B03: Adicionar SUPPLY ao enum | 1 linha em 1 arquivo | ✅ |
| B04: Ports/In (9 interfaces) | 9 interfaces coesas, mesma camada | ✅ |
| B05: Ports/Out (2 interfaces) | 2 interfaces coesas, mesma camada | ✅ |
| B06: Entity + JpaRepository de item | 2 arquivos de infra coesos | ✅ |
| B07: Mapper + RepositoryImpl de item | 2 arquivos de infra coesos | ✅ |
| B08: Entity + JpaRepository de movement | 2 arquivos de infra coesos | ✅ |
| B09: Mapper + RepositoryImpl de movement | 2 arquivos de infra coesos | ✅ |
| B10: 6 use cases CRUD | 6 arquivos — mesma camada, mesma entidade | ✅ |
| B11: RegisterPurchaseUseCaseImpl | 1 arquivo, lógica transacional crítica | ✅ |
| B12: RegisterManualExit + ListMovements | 2 use cases simples, coesos | ✅ |
| B13: 7 DTOs | 7 arquivos — mesma camada, sem lógica | ✅ |
| B14: InventoryController | 1 arquivo de controller | ✅ |
| B15: GlobalExceptionHandler (3 handlers) | 1 arquivo modificado, 3 adições | ✅ |
| F01: inventory.model.ts | 1 arquivo TypeScript | ✅ |
| F02: InventoryService | 1 arquivo de serviço | ✅ |
| F03: actions + reducer + selectors | 3 arquivos NgRx coesos | ✅ |
| F04: effects | 1 arquivo NgRx | ✅ |
| F05: InventoryListComponent | 1 componente (3 arquivos) | ✅ |
| F06: InventoryFormComponent | 1 componente (3 arquivos) | ✅ |
| F07: InventoryPurchaseDialogComponent | 1 componente (3 arquivos) | ✅ |
| F08: InventoryExitDialogComponent | 1 componente (3 arquivos) | ✅ |
| F09: InventoryComponent shell + rotas | 1 componente shell + wiring | ✅ |
| F10: InventoryHistoryDialogComponent [P2] | 1 componente (3 arquivos) | ✅ |
| F11: Dashboard card estoque baixo [P2] | 1 feature backend + 1 frontend coesas | ✅ |

---

## Cross-Check Diagrama × Dependências

| Task | Depende de (body) | Diagrama mostra | Status |
|---|---|---|---|
| B01 | Nenhum | Fase 1 raiz | ✅ |
| B02 | Nenhum | Fase 1 raiz | ✅ |
| B03 | Nenhum | Fase 1 raiz | ✅ |
| B04 | B02 | B02 → Fase 2 | ✅ |
| B05 | B02 | B02 → Fase 2 | ✅ |
| B06 | B01, B05 | B01+B05 → Fase 3 | ✅ |
| B08 | B01, B05 | B01+B05 → Fase 3 | ✅ |
| B07 | B05, B06 | Fase 3 → Fase 4 | ✅ |
| B09 | B05, B08 | Fase 3 → Fase 4 | ✅ |
| B10 | B04, B07 | Fase 4+B04 → Fase 5 | ✅ |
| B11 | B04, B07, B09 | Fase 4+B04 → Fase 5 | ✅ |
| B12 | B04, B07, B09 | Fase 4+B04 → Fase 5 | ✅ |
| B13 | B04 | B04 → Fase 6 | ✅ |
| B15 | B02 | B02 → Fase 6 | ✅ |
| B14 | B10, B11, B12, B13, B15 | Fase 5+B13+B15 → B14 | ✅ |
| F01 | Nenhum | Phase 7a raiz | ✅ |
| F02 | F01 | F01 → Phase 7b | ✅ |
| F03 | F01 | F01 → Phase 7b | ✅ |
| F04 | F02, F03 | F02+F03 → Phase 7c | ✅ |
| F05–F08 | F03 | F03 → Phase 7d | ✅ |
| F09 | F04, F05, F06, F07, F08 | Phase 7c+7d → F09 | ✅ |
| F10 | F02, F03 | F09 → Phase 7f | ✅ |
| F11 | B14 | F09 → Phase 7f | ✅ |

---

## Validação de Co-localização de Testes

Conforme `TESTING.md`: zero cobertura implementada — todos os layers estão com `Tests: none`.
Gate padrão é `mvn compile` (backend) e `npm run build` (frontend).

| Task | Camada criada/modificada | Matrix exige | Task diz | Status |
|---|---|---|---|---|
| B01 | Migration SQL | none | none | ✅ |
| B02 | Domain model | unit (recomendado, não obrigatório) | none | ✅ |
| B03–B15 | Todas as camadas backend | none (zero cobertura) | none | ✅ |
| F01–F11 | Todas as camadas frontend | none (zero cobertura) | none | ✅ |

> ⚠️ Blocker B01 ativo: zero cobertura de testes. Tasks não incluem testes por consistência com o resto do projeto — mas o B11 (`RegisterPurchaseUseCaseImpl`) é candidato prioritário para o primeiro teste quando a cobertura for iniciada (lógica transacional crítica, alto impacto financeiro).

---

## Rastreabilidade de Requisitos

| Requisito | Task(s) |
|---|---|
| INV-01 a INV-04 | B02, B04, B05, B10, B13, B14, F01, F06 |
| INV-05 a INV-08 | B06, B07, B10, B13, B14, F01, F05, F09 |
| INV-09 a INV-10 | B10, B13, B14, F06 |
| INV-11 a INV-12 | B02, B04, B05, B10, B15, F05 |
| INV-13 a INV-17 | B01, B02, B03, B04, B05, B08, B09, B11, B13, B14, F07 |
| INV-18 a INV-20 | B04, B05, B08, B09, B12, B13, B14, F08 |
| INV-21 a INV-22 | B08, B09, B12, B13, B14, F10 |
| INV-23 a INV-24 | B10, B13, B14, F05, F09 |
| INV-25 | F09 |
| INV-26 | F11 |
