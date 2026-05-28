# Serviços — Tasks

## Status Geral

```
T01 [x] Domain + Ports
T02 [x] Use Cases
T03 [x] Infrastructure
T04 [x] Controller + DTOs
T05 [x] Frontend Model
T06 [x] DurationPipe
T07 [x] ServiceHttp
T08 [x] NgRx Store
T09 [x] ServiceListComponent
T10 [x] ServiceFormComponent
T11 [x] ServiceDetailComponent
T12 [x] Integração Final
```

---

## Grafo de Dependências

```
T01 → T02 → T04
T01 → T03 → T04

T05 → T07 → T08 → T09 → T12
T06 ──────────────→ T09
T08 → T10 → T12
T08 → T11 → T12
```

**Paralelismo possível:**
- `[P]` T02 e T03 podem rodar em paralelo após T01
- `[P]` T05 e T06 podem rodar em paralelo (sem dependências)
- `[P]` T09, T10 e T11 podem rodar em paralelo após T08

---

## Tasks

### T01 — Domain + Ports

**O que:** Criar o modelo de domínio `Service`, exceção `ServiceNotFoundException`, 5 portas de entrada e 1 porta de saída.

**Onde:**
- `domain/model/Service.java`
- `domain/exception/ServiceNotFoundException.java`
- `domain/port/in/CreateServiceUseCase.java`
- `domain/port/in/UpdateServiceUseCase.java`
- `domain/port/in/GetServiceUseCase.java`
- `domain/port/in/ListServicesUseCase.java`
- `domain/port/in/DeactivateServiceUseCase.java`
- `domain/port/out/ServiceRepository.java`

**Depende de:** —

**Reutiliza:** Padrão de `Client.java` e portas do módulo de Clientes

**Done when:**
- `Service.java` com campos: `UUID id`, `String name`, `String description`, `BigDecimal price`, `int durationMinutes`, `boolean active`, `LocalDateTime createdAt`, `LocalDateTime updatedAt`; anotações `@Getter @Builder @NoArgsConstructor @AllArgsConstructor`; sem dependências de framework
- `ServiceNotFoundException extends RuntimeException`
- `CreateServiceUseCase` com `CreateServiceCommand(name, description, price, durationMinutes)` e `ServiceResult(id, name, description, price, durationMinutes, active, createdAt)`
- `UpdateServiceUseCase` com `UpdateServiceCommand` igual ao create
- `GetServiceUseCase.execute(UUID id)` retorna `ServiceResult`
- `ListServicesUseCase.execute(String search, Pageable pageable)` retorna `Page<ServiceResult>`
- `DeactivateServiceUseCase` com métodos `deactivate(UUID)` e `reactivate(UUID)`
- `ServiceRepository` com: `save`, `findById`, `findAll(search, activeOnly, pageable)`

**Gate:** `mvn compile` sem erros

---

### T02 — Use Cases [P após T01]

**O que:** Implementar os 5 use cases e o mapper utilitário.

**Onde:**
- `application/usecase/CreateServiceUseCaseImpl.java`
- `application/usecase/UpdateServiceUseCaseImpl.java`
- `application/usecase/GetServiceUseCaseImpl.java`
- `application/usecase/ListServicesUseCaseImpl.java`
- `application/usecase/DeactivateServiceUseCaseImpl.java`
- `application/usecase/ServiceUseCaseMapper.java`

**Depende de:** T01

**Reutiliza:** Padrão de `*ClientUseCaseImpl.java` e `ClientUseCaseMapper.java`

**Done when:**
- `CreateServiceUseCaseImpl` salva via `ServiceRepository.save()` e retorna `ServiceResult`
- `UpdateServiceUseCaseImpl` busca por ID (lança `ServiceNotFoundException` se não encontrar), atualiza e salva
- `GetServiceUseCaseImpl` busca por ID, lança `ServiceNotFoundException` se não encontrar
- `ListServicesUseCaseImpl` delega para `ServiceRepository.findAll(search, true, pageable)`
- `DeactivateServiceUseCaseImpl` busca, inverte flag `active` e salva; SVC-13 como TODO comentado
- `ServiceUseCaseMapper` converte `Service → ServiceResult`
- Todas as classes usam `@Service @RequiredArgsConstructor`

**Gate:** `mvn compile` sem erros

---

### T03 — Infrastructure [P após T01]

**O que:** Criar a entidade JPA, o mapper, o repositório Spring Data e a implementação da porta de saída.

**Onde:**
- `infrastructure/persistence/entity/ServiceEntity.java`
- `infrastructure/persistence/mapper/ServiceMapper.java`
- `infrastructure/persistence/repository/ServiceJpaRepository.java`
- `infrastructure/persistence/repository/ServiceRepositoryImpl.java`

**Depende de:** T01

**Reutiliza:** Padrão de `ClientEntity`, `ClientMapper`, `ClientJpaRepository`, `ClientRepositoryImpl`

**Done when:**
- `ServiceEntity` com `@Entity @Table(name = "services")`; campos mapeados para o schema existente (`price` como `BigDecimal`, `duration_minutes` como `durationMinutes`); `@CreationTimestamp` / `@UpdateTimestamp`
- `ServiceMapper` com `toEntity(Service)` e `toDomain(ServiceEntity)` usando `@Component`
- `ServiceJpaRepository extends JpaRepository<ServiceEntity, UUID>` com query JPQL:
  ```sql
  SELECT s FROM ServiceEntity s
  WHERE (:activeOnly = false OR s.active = true)
  AND (:search IS NULL OR LOWER(s.name) LIKE LOWER(CONCAT('%', :search, '%')))
  ORDER BY s.name ASC
  ```
- `ServiceRepositoryImpl implements ServiceRepository` delegando para `ServiceJpaRepository` + `ServiceMapper`

**Gate:** `mvn compile` sem erros

---

### T04 — Controller + DTOs

**O que:** Criar os DTOs de request/response, o controller REST e adicionar handler no `GlobalExceptionHandler`.

**Onde:**
- `adapter/web/dto/CreateServiceRequest.java`
- `adapter/web/dto/UpdateServiceRequest.java`
- `adapter/web/dto/ServiceResponse.java`
- `adapter/web/controller/ServiceController.java`
- `adapter/web/controller/GlobalExceptionHandler.java` (atualizar)

**Depende de:** T02, T03

**Reutiliza:** Padrão de `ClientController` e DTOs de clientes

**Done when:**
- `CreateServiceRequest` record com `@NotBlank String name`, `String description`, `@NotNull @DecimalMin("0.01") BigDecimal price`, `@NotNull @Min(1) Integer durationMinutes`
- `UpdateServiceRequest` idêntico ao create
- `ServiceResponse` record com todos os campos do `ServiceResult`
- `ServiceController @RestController @RequestMapping("/api/services")` com 6 endpoints:
  - `POST /` → 201 Created
  - `GET /?search=&page=&size=` → 200 com `Page<ServiceResponse>`
  - `GET /{id}` → 200
  - `PUT /{id}` → 200
  - `PATCH /{id}/deactivate` → 204
  - `PATCH /{id}/reactivate` → 204
- `GlobalExceptionHandler` com `@ExceptionHandler(ServiceNotFoundException.class)` retornando 404

**Gate:** `mvn test` passa; `mvn spring-boot:run` sobe sem erros; `GET /api/services` retorna os 4 serviços do seed

---

### T05 — Frontend Model [P]

**O que:** Criar o model TypeScript para Serviços.

**Onde:**
- `src/app/core/models/service.model.ts`

**Depende de:** —

**Done when:**
- Interfaces `Service`, `CreateServiceRequest`, `ServiceState` conforme design:
  - `Service`: `id`, `name`, `description?`, `price`, `durationMinutes`, `active`, `createdAt`
  - `CreateServiceRequest`: `name`, `description?`, `price`, `durationMinutes`
  - `ServiceState`: `services[]`, `totalElements`, `currentPage`, `pageSize`, `search`, `selectedService`, `isLoading`, `isSaving`, `error`

**Gate:** `npm run build` sem erros de tipo

---

### T06 — DurationPipe [P]

**O que:** Criar pipe compartilhado para formatar minutos em formato legível.

**Onde:**
- `src/app/core/pipes/duration.pipe.ts`

**Depende de:** —

**Done when:**
- `@Pipe({ name: 'duration', standalone: true })` em `core/pipes/`
- Lógica: `180 → "3h"` | `90 → "1h 30min"` | `30 → "30min"` | `0 ou null → "-"`
- Exportado corretamente para uso nos componentes standalone

**Gate:** `npm run build` sem erros de tipo

---

### T07 — ServiceHttp

**O que:** Criar o serviço Angular de comunicação HTTP com a API.

**Onde:**
- `src/app/features/services/services/service.service.ts`

**Depende de:** T05

**Reutiliza:** Padrão de `client.service.ts`

**Done when:**
- `@Injectable({ providedIn: 'root' })` com `inject(HttpClient)`
- Métodos: `list(search?, page?, size?)`, `getById(id)`, `create(req)`, `update(id, req)`, `deactivate(id)`, `reactivate(id)`
- Base URL: `/api/services`
- Retornos tipados com os models de `service.model.ts`

**Gate:** `npm run build` sem erros de tipo

---

### T08 — NgRx Store

**O que:** Criar actions, reducer, selectors e effects do módulo de Serviços.

**Onde:**
- `src/app/features/services/store/service.actions.ts`
- `src/app/features/services/store/service.reducer.ts`
- `src/app/features/services/store/service.selectors.ts`
- `src/app/features/services/store/service.effects.ts`

**Depende de:** T05, T07

**Reutiliza:** Padrão completo do módulo de Clientes (client.actions/reducer/selectors/effects)

**Done when:**
- **Actions** sob `ServiceActions`: `loadServices`, `loadServicesSuccess`, `loadServicesFailure`, `selectService`, `selectServiceSuccess`, `selectServiceFailure`, `clearSelectedService`, `createService`, `createServiceSuccess`, `createServiceFailure`, `updateService`, `updateServiceSuccess`, `updateServiceFailure`, `deactivateService`, `deactivateServiceSuccess`, `deactivateServiceFailure`, `reactivateService`, `reactivateServiceSuccess`, `reactivateServiceFailure`
- **Reducer** `serviceReducer` com `initialState` e todos os `on()` handlers; slice key: `services`
- **Selectors**: `selectServicesState`, `selectAllServices`, `selectServicesLoading`, `selectServicesSaving`, `selectServicesError`, `selectSelectedService`, `selectServicesTotalElements`, `selectServicesCurrentPage`, `selectServicesSearch`
- **Effects**: `loadServices$`, `selectService$`, `createService$` (navega para listagem no sucesso), `updateService$` (navega para detalhe no sucesso), `deactivateService$`, `reactivateService$`

**Gate:** `npm run build` sem erros de tipo

---

### T09 — ServiceListComponent [P após T08]

**O que:** Criar o componente de listagem com busca e exibição de preço e duração.

**Onde:**
- `src/app/features/services/components/service-list/service-list.component.ts`

**Depende de:** T08, T06

**Reutiliza:** Padrão de `client-list.component.ts`

**Done when:**
- Standalone component com `DurationPipe`, `CurrencyBrPipe` e pipes do Angular Material
- Busca por nome com `debounceTime(300)` e `distinctUntilChanged()`
- Exibe nome, preço formatado, duração formatada, chip ativo/inativo
- Empty state: "Nenhum serviço encontrado"
- Skeleton loader enquanto `isLoading`
- Click em item navega para `/services/:id`
- Botão "Novo Serviço" navega para `/services/novo`

**Gate:** `npm run build` sem erros de tipo

---

### T10 — ServiceFormComponent [P após T08]

**O que:** Criar o formulário compartilhado de criação e edição.

**Onde:**
- `src/app/features/services/components/service-form/service-form.component.ts`

**Depende de:** T08

**Reutiliza:** Padrão de `client-form.component.ts` (sem DatePicker)

**Done when:**
- Detecta modo create/edit via presença de `:id` na rota
- No modo edit: despacha `selectService`, preenche o form com `patchValue` ao receber `selectedService$`
- Campos: `name` (required), `description` (optional), `price` (required, min 0.01), `durationMinutes` (required, min 1)
- Mensagens de erro inline por campo
- Botão Salvar despacha `createService` ou `updateService`; desabilitado quando `isSaving`
- Botão Cancelar volta para a rota anterior sem salvar

**Gate:** `npm run build` sem erros de tipo

---

### T11 — ServiceDetailComponent [P após T08]

**O que:** Criar o componente de visualização do perfil do serviço.

**Onde:**
- `src/app/features/services/components/service-detail/service-detail.component.ts`

**Depende de:** T08, T06

**Reutiliza:** Padrão de `client-detail.component.ts`

**Done when:**
- Despacha `selectService` no `ngOnInit` com o `:id` da rota
- Exibe todos os campos: nome, descrição, preço (`currencyBr`), duração (`DurationPipe`), status (chip), data de cadastro
- Botão "Editar" navega para `/services/:id/editar`
- Botão "Desativar" (serviço ativo) abre `MatDialog` de confirmação → despacha `deactivateService`
- Botão "Reativar" (serviço inativo) despacha `reactivateService` diretamente
- Exibe 404 com link para listagem se `ServiceNotFoundException` (erro no state)

**Gate:** `npm run build` sem erros de tipo

---

### T12 — Integração Final

**O que:** Atualizar rotas do módulo, registrar reducer/effects no app.config e verificar navegação.

**Onde:**
- `src/app/features/services/services.routes.ts`
- `src/app/app.config.ts`

**Depende de:** T09, T10, T11

**Done when:**
- `services.routes.ts` com: `''` → `ServiceListComponent`, `'novo'` → `ServiceFormComponent`, `':id'` → `ServiceDetailComponent`, `':id/editar'` → `ServiceFormComponent`
- `app.config.ts` atualizado:
  ```typescript
  provideStore({ auth: authReducer, clients: clientReducer, services: serviceReducer }),
  provideEffects([AuthEffects, ClientEffects, ServiceEffects]),
  ```
- Navegação completa funciona: listagem → detalhe → editar → voltar
- Backend rodando: `GET /api/services` retorna os 4 serviços do seed na UI

**Gate:** `npm start` sobe sem erros; listagem de serviços exibe os 4 do seed com preço e duração formatados

---

## Resumo

| Task | Arquivo(s) | Depende | Paralelo |
|---|---|---|---|
| T01 | 8 arquivos domain/ports | — | — |
| T02 | 6 arquivos use cases | T01 | [P] com T03 |
| T03 | 4 arquivos infra | T01 | [P] com T02 |
| T04 | 5 arquivos adapter | T02, T03 | — |
| T05 | service.model.ts | — | [P] com T06 |
| T06 | duration.pipe.ts | — | [P] com T05 |
| T07 | service.service.ts | T05 | — |
| T08 | 4 arquivos NgRx | T05, T07 | — |
| T09 | service-list | T08, T06 | [P] com T10, T11 |
| T10 | service-form | T08 | [P] com T09, T11 |
| T11 | service-detail | T08, T06 | [P] com T09, T10 |
| T12 | routes + config | T09, T10, T11 | — |

**Total:** 12 tasks, ~36 arquivos novos, 2 atualizados
