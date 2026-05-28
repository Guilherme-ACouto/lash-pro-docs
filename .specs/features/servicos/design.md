# Serviços — Design

## Avaliação de Escopo

**Large** — Multi-componente (backend + frontend), porém estrutura idêntica ao módulo de Clientes já implementado. Decisões arquiteturais já estabelecidas; foco aqui é mapear componentes, diferenças de domínio e decisões específicas de Serviços.

---

## Reutilização do Módulo de Clientes

O módulo de Serviços segue o **mesmo padrão** do módulo de Clientes em todas as camadas. As diferenças são apenas de domínio (campos diferentes, sem unicidade de telefone).

| Camada | Clientes (referência) | Serviços (novo) |
|---|---|---|
| Domain model | `Client` (name, phone, email, birthDate, notes, active) | `Service` (name, description, price, durationMinutes, active) |
| Port in | 5 interfaces (Create/Update/Get/List/Deactivate) | 5 interfaces idênticas em estrutura |
| Use case impl | 5 classes `*UseCaseImpl` | 5 classes `*UseCaseImpl` |
| Infrastructure | `ClientEntity`, `ClientJpaRepository`, `ClientMapper`, `ClientRepositoryImpl` | Equivalentes para Service |
| Adapter | `CreateClientRequest`, `UpdateClientRequest`, `ClientResponse`, `ClientController` | Equivalentes para Service |
| NgRx | actions / reducer / effects / selectors | Mesmo padrão |
| Components | list / form / detail | list / form / detail |

---

## Backend

### Domain Model

```
lash-backend/src/main/java/com/lashmanager/app/
├── domain/
│   ├── model/
│   │   └── Service.java                          ← NOVO
│   ├── exception/
│   │   └── ServiceNotFoundException.java         ← NOVO
│   └── port/
│       ├── in/
│       │   ├── CreateServiceUseCase.java         ← NOVO
│       │   ├── UpdateServiceUseCase.java         ← NOVO
│       │   ├── GetServiceUseCase.java            ← NOVO
│       │   ├── ListServicesUseCase.java          ← NOVO
│       │   └── DeactivateServiceUseCase.java     ← NOVO
│       └── out/
│           └── ServiceRepository.java            ← NOVO
```

**Service.java** — campos: `UUID id`, `String name`, `String description`, `BigDecimal price`, `int durationMinutes`, `boolean active`, `LocalDateTime createdAt/updatedAt`

**ServiceNotFoundException** — extends `RuntimeException`, mapeada para HTTP 404 no `GlobalExceptionHandler`

**Sem unicidade de nome** — o schema não tem constraint unique em `services.name`; dois serviços podem ter nomes similares (ex: "Volume Russo" e "Volume Russo Premium")

### Use Cases

```
application/usecase/
├── CreateServiceUseCaseImpl.java    ← NOVO
├── UpdateServiceUseCaseImpl.java    ← NOVO
├── GetServiceUseCaseImpl.java       ← NOVO
├── ListServicesUseCaseImpl.java     ← NOVO
├── DeactivateServiceUseCaseImpl.java← NOVO (SVC-13 placeholder, como CLI-13)
└── ServiceUseCaseMapper.java        ← NOVO
```

**ServiceResult** (record no `CreateServiceUseCase`): `UUID id`, `String name`, `String description`, `BigDecimal price`, `int durationMinutes`, `boolean active`, `String createdAt`

**ServiceUseCaseMapper** — converte `Service` → `ServiceResult`; converte `BigDecimal price` para serialização (mantém como BigDecimal, o Jackson serializa corretamente)

### Infrastructure

```
infrastructure/persistence/
├── entity/ServiceEntity.java        ← NOVO
├── mapper/ServiceMapper.java        ← NOVO
└── repository/
    ├── ServiceJpaRepository.java    ← NOVO
    └── ServiceRepositoryImpl.java   ← NOVO
```

**ServiceJpaRepository** — JPQL query por nome (sem filtro de telefone):

```java
@Query("""
    SELECT s FROM ServiceEntity s
    WHERE (:activeOnly = false OR s.active = true)
    AND (:search IS NULL OR LOWER(s.name) LIKE LOWER(CONCAT('%', :search, '%')))
    ORDER BY s.name ASC
    """)
Page<ServiceEntity> search(@Param("search") String search,
                           @Param("activeOnly") boolean activeOnly,
                           Pageable pageable);
```

**ServiceRepository (port/out):**
```java
Service save(Service service);
Optional<Service> findById(UUID id);
Page<Service> findAll(String search, boolean activeOnly, Pageable pageable);
```

### Adapter

```
adapter/web/
├── dto/
│   ├── CreateServiceRequest.java    ← NOVO
│   ├── UpdateServiceRequest.java    ← NOVO
│   └── ServiceResponse.java        ← NOVO
└── controller/
    └── ServiceController.java      ← NOVO
```

**API Contract:**

| Método | Endpoint | Descrição | Status |
|---|---|---|---|
| POST | `/api/services` | Criar serviço | 201 Created |
| GET | `/api/services?search=&page=&size=` | Listar (paginado) | 200 OK |
| GET | `/api/services/{id}` | Buscar por ID | 200 OK |
| PUT | `/api/services/{id}` | Atualizar | 200 OK |
| PATCH | `/api/services/{id}/deactivate` | Desativar | 204 No Content |
| PATCH | `/api/services/{id}/reactivate` | Reativar | 204 No Content |

**CreateServiceRequest / UpdateServiceRequest:**
```java
record CreateServiceRequest(
    @NotBlank String name,
    String description,
    @NotNull @DecimalMin("0.01") BigDecimal price,
    @NotNull @Min(1) Integer durationMinutes
) {}
```

**GlobalExceptionHandler** — adicionar handler para `ServiceNotFoundException` → 404

---

## Frontend

### Estrutura de Arquivos

```
lash-frontend/src/app/
├── core/models/
│   └── service.model.ts             ← NOVO
├── features/services/
│   ├── services.component.ts        ← NOVO (shell, já existe como placeholder)
│   ├── services.routes.ts           ← ATUALIZAR (substituir placeholder)
│   ├── services/
│   │   └── service.service.ts      ← NOVO
│   ├── store/
│   │   ├── service.actions.ts      ← NOVO
│   │   ├── service.reducer.ts      ← NOVO
│   │   ├── service.effects.ts      ← NOVO
│   │   └── service.selectors.ts    ← NOVO
│   └── components/
│       ├── service-list/
│       │   └── service-list.component.ts   ← NOVO
│       ├── service-form/
│       │   └── service-form.component.ts   ← NOVO
│       └── service-detail/
│           └── service-detail.component.ts ← NOVO
└── app.config.ts                    ← ATUALIZAR (adicionar servicesReducer + ServiceEffects)
```

### Models

**service.model.ts:**
```typescript
export interface Service {
  id: string;
  name: string;
  description?: string;
  price: number;
  durationMinutes: number;
  active: boolean;
  createdAt: string;
}

export interface CreateServiceRequest {
  name: string;
  description?: string;
  price: number;
  durationMinutes: number;
}

export interface ServiceState {
  services: Service[];
  totalElements: number;
  currentPage: number;
  pageSize: number;
  search: string;
  selectedService: Service | null;
  isLoading: boolean;
  isSaving: boolean;
  error: string | null;
}
```

### Duração: DurationPipe

**Decisão:** Criar `DurationPipe` compartilhado em `core/pipes/` para formatar minutos como "Xh Ymin". Será reutilizado em Agendamentos.

```typescript
// core/pipes/duration.pipe.ts
// 180 → "3h"  |  90 → "1h 30min"  |  30 → "30min"
```

### Rotas

```typescript
// services.routes.ts
export const servicesRoutes: Routes = [
  { path: '', component: ServiceListComponent },
  { path: 'novo', component: ServiceFormComponent },
  { path: ':id', component: ServiceDetailComponent },
  { path: ':id/editar', component: ServiceFormComponent },
];
```

### Componentes

**ServiceListComponent:**
- Busca por nome com debounce 300ms (igual Clients)
- Exibe nome, preço (`currencyBr` pipe), duração (`DurationPipe`)
- Chip de status ativo/inativo
- Empty state condicional

**ServiceFormComponent:**
- Detecta modo create/edit via `:id` na rota
- Campos: nome, descrição, preço (number input), duração em minutos (number input)
- Validators: `Validators.required` + `Validators.min(0.01)` para preço, `Validators.min(1)` para duração
- Sem DatePicker (diferença do ClientForm)

**ServiceDetailComponent:**
- Exibe todos os campos com `DurationPipe` e `currencyBr`
- Chip ativo/inativo
- Botão desativar com confirm dialog (mesmo padrão do ClientDetail)

### NgRx — app.config.ts

```typescript
provideStore({ auth: authReducer, clients: clientReducer, services: serviceReducer }),
provideEffects([AuthEffects, ClientEffects, ServiceEffects]),
```

---

## Decisões de Design

| Decisão | Escolha | Razão |
|---|---|---|
| Unicidade de nome | Sem constraint | Schema não impõe; profissional pode ter variações ("Manutenção 1h", "Manutenção 30min") |
| Representação de duração | `durationMinutes: int` no backend, pipe no frontend | Dado atômico no banco, apresentação amigável na UI |
| `DurationPipe` em `core/pipes/` | Compartilhado | Será reutilizado em Agendamentos e futuramente em Dashboard |
| SVC-13 (desativar com agendamentos) | Placeholder igual CLI-13 | Implementação real depende do módulo de Agendamentos |
| Paginação | Mantida (`size=20`) | Consistência com Clients; catálogo cresce com o tempo |

---

## Rastreabilidade

| Req ID | Componente responsável |
|---|---|
| SVC-01 a SVC-04 | `ServiceFormComponent` + `CreateServiceUseCaseImpl` |
| SVC-05 a SVC-07 | `ServiceListComponent` + `ListServicesUseCaseImpl` + `DurationPipe` |
| SVC-08 | `ServiceDetailComponent` + `GetServiceUseCaseImpl` |
| SVC-09 a SVC-10 | `ServiceFormComponent` + `UpdateServiceUseCaseImpl` |
| SVC-11 a SVC-13 | `ServiceDetailComponent` + `DeactivateServiceUseCaseImpl` |
