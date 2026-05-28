# Fichas — Tasks

**Design**: `.specs/features/fichas/design.md`
**Status**: Draft

---

## Plano de Execução

### Fase 1 — Fundação do Domain (paralelo)

```
B01 ──┐
B02 ──┼──→ Fase 2
```

### Fase 2 — Ports (paralelo após B02)

```
B02 ──→ B03 ──┐
B02 ──→ B04 ──┼──→ Fase 3
```

### Fase 3 — Entities JPA (paralelo após B01+B04)

```
B01 + B04 ──→ B05 ──┐
B01 + B04 ──→ B06 ──┼──→ Fase 4
```

### Fase 4 — Mappers + Repository Impls (paralelo após Fase 3)

```
B04 + B05 ──→ B07 ──┐
B04 + B06 ──→ B08 ──┼──→ Fase 5
```

### Fase 5 — Use Cases (paralelo após Fase 4)

```
B03 + B07 ──────────────── B09 ──┐
B03 + B07 ──────────────── B10 ──┼──→ Fase 6
B03 + B07 + B08 ────────── B11 ──┘
```

### Fase 6 — DTOs, SecurityConfig, Handler e Controllers (sequencial)

```
B02 ──→ B15 ──┐
B03 ──→ B12 ──┤
              └──→ B13 (B09 + B10 + B12 + B15)
              └──→ B14 (B11 + B12 + B15)
```

### Fase 7 — Frontend (sequencial, após Fase 6)

```
F01

F01 ──→ F02 ──┐
F01 ──→ F03 ──┼──→ F04

F03 ──→ F05 ──┐
F03 ──→ F06 ──┤
F03 ──→ F07 ──┤──→ F11
F03 ──→ F08 ──┤
F09 ──→ F10 ──┘

F02 ──→ F12

F11 + F12 ──→ F13
F13 ──────────→ F14
F13 ──────────→ F15
```

---

## Breakdown das Tarefas

### B01: Migration V7 — tabelas anamneses, anamnese_tokens, lash_mappings

**O que**: Criar `V7__fichas_schema.sql` com as 3 novas tabelas e seus índices
**Onde**: `lash-backend/src/main/resources/db/migration/V7__fichas_schema.sql`
**Depende de**: Nenhum
**Reutiliza**: Padrão das migrations V1–V6
**Requisitos**: FIC-02, FIC-05

**Feito quando:**
- [ ] Tabela `anamneses` criada com `UNIQUE(client_id)` e todos os campos de saúde/termo conforme design
- [ ] Tabela `anamnese_tokens` criada com `token VARCHAR(64) UNIQUE`, `expires_at`, `used BOOLEAN`
- [ ] Tabela `lash_mappings` criada com `client_id` (FK), campos técnicos + `canvas_data TEXT`, `photo_before TEXT`, `photo_after TEXT`
- [ ] Índices criados: `idx_anamnese_tokens_token`, `idx_lash_mappings_client_id`, `idx_lash_mappings_date`
- [ ] Gate: `mvn compile` sem erros (Flyway valida na inicialização)

**Verificar**: `mvn compile`

---

### B02: Domain — Modelos + Exceções

**O que**: Criar os 3 modelos de domínio e as 5 exceções do módulo de Fichas
**Onde**:
- `domain/model/Anamnese.java` (novo)
- `domain/model/AnamneseToken.java` (novo)
- `domain/model/LashMapping.java` (novo)
- `domain/exception/AnamneseNotFoundException.java` (novo)
- `domain/exception/AnamneseTokenNotFoundException.java` (novo)
- `domain/exception/AnamneseTokenExpiredException.java` (novo)
- `domain/exception/AnamneseTokenAlreadyUsedException.java` (novo)
- `domain/exception/LashMappingNotFoundException.java` (novo)
**Depende de**: Nenhum
**Reutiliza**: Padrão de `Client.java`, `FinancialEntry.java`, `ClientNotFoundException.java`
**Requisitos**: FIC-02, FIC-05, FIC-06

**Feito quando:**
- [ ] `Anamnese`: todos os campos da tabela + `clientName` (string transiente para joins de leitura); campo `sleepSide` como `String` (valores: `DIREITO`, `ESQUERDO`, `AMBOS`)
- [ ] `AnamneseToken`: `id`, `clientId`, `token`, `expiresAt`, `used`
- [ ] `LashMapping`: todos os campos da tabela + `clientName` transiente
- [ ] 5 exceções estendendo `BusinessException` com mensagens e status HTTP corretos: 404/410/409/404/404 respectivamente
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B03: Ports/In — 11 interfaces de use cases

**O que**: Criar as 11 interfaces de port/in do módulo de Fichas
**Onde**: `domain/port/in/` — 11 arquivos novos:
- `GetOrCreateAnamneseUseCase.java`
- `SaveAnamneseUseCase.java`
- `ListAnamneseSummaryUseCase.java`
- `GenerateAnamneseLinkUseCase.java`
- `GetAnamneseByTokenUseCase.java`
- `SubmitAnamneseByTokenUseCase.java`
- `CreateLashMappingUseCase.java`
- `UpdateLashMappingUseCase.java`
- `GetLashMappingUseCase.java`
- `ListMappingsByClientUseCase.java`
- `ListMappingSummaryUseCase.java`
**Depende de**: B02
**Reutiliza**: Padrão de `CreateClientUseCase.java`, `ListClientsUseCase.java`
**Requisitos**: FIC-02, FIC-03, FIC-04, FIC-05, FIC-06, FIC-07, FIC-08

**Feito quando:**
- [ ] Cada interface define comando record interno (ou parâmetros diretos) e tipo de retorno conforme design
- [ ] `SaveAnamneseUseCase` define `SaveAnamneseCommand` record com todos os campos editáveis da anamnese
- [ ] `SubmitAnamneseByTokenUseCase` define `SubmitAnamneseCommand` com `token` + dados da ficha
- [ ] `CreateLashMappingUseCase` e `UpdateLashMappingUseCase` definem commands com todos os campos do mapping (incluindo `canvasData`, `photoBefore`, `photoAfter`)
- [ ] `ListAnamneseSummaryUseCase` retorna `Page<AnamneseSummaryResult>` (record: clientId, clientName, hasAnamnese, updatedAt)
- [ ] `ListMappingSummaryUseCase` retorna `Page<MappingSummaryResult>` (record: clientId, clientName, mappingCount, lastMappingDate)
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B04: Ports/Out — 3 interfaces de repositório

**O que**: Criar as 3 interfaces de port/out do módulo de Fichas
**Onde**: `domain/port/out/` — 3 arquivos novos:
- `AnamneseRepository.java`
- `AnamneseTokenRepository.java`
- `LashMappingRepository.java`
**Depende de**: B02
**Reutiliza**: Padrão de `ClientRepository.java`, `FinancialEntryRepository.java`
**Requisitos**: FIC-02, FIC-04, FIC-05, FIC-06, FIC-08

**Feito quando:**
- [ ] `AnamneseRepository`: `save`, `findByClientId(UUID) → Optional<Anamnese>`, `findAllWithClientSummary(search, pageable)`, `existsByClientId(UUID)`
- [ ] `AnamneseTokenRepository`: `save`, `findByToken(String) → Optional<AnamneseToken>`, `deleteExpired()`
- [ ] `LashMappingRepository`: `save`, `findById(UUID)`, `findByClientId(UUID, pageable)`, `findAllWithClientSummary(search, pageable)`, `delete(UUID)`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B05: Infra — Entidades JPA de Anamnese e Token

**O que**: Criar as entidades JPA e repositórios Spring Data para `anamneses` e `anamnese_tokens`
**Onde**:
- `infrastructure/persistence/entity/AnamneseEntity.java` (novo)
- `infrastructure/persistence/entity/AnamneseTokenEntity.java` (novo)
- `infrastructure/persistence/repository/AnamneseJpaRepository.java` (novo)
- `infrastructure/persistence/repository/AnamneseTokenJpaRepository.java` (novo)
**Depende de**: B01, B04
**Reutiliza**: Padrão de `ClientEntity.java`, `FinancialEntryEntity.java`
**Requisitos**: FIC-02, FIC-04

**Feito quando:**
- [ ] `AnamneseEntity` mapeada para `anamneses` com todos os campos; `@ManyToOne(fetch=LAZY, insertable=false, updatable=false)` para `ClientEntity` (só para resolver `clientName`)
- [ ] `AnamneseTokenEntity` mapeada para `anamnese_tokens` com todos os campos
- [ ] `AnamneseJpaRepository` tem query JPQL `findAllWithClient(search, pageable)` com `LEFT JOIN FETCH client` + filtro `LOWER(client.name) LIKE LOWER(CONCAT('%',:search,'%'))`
- [ ] `AnamneseTokenJpaRepository` tem `findByTokenAndUsedFalseAndExpiresAtAfter(String token, LocalDateTime now)`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B06: Infra — Entidade JPA de LashMapping

**O que**: Criar a entidade JPA e repositório Spring Data para `lash_mappings`
**Onde**:
- `infrastructure/persistence/entity/LashMappingEntity.java` (novo)
- `infrastructure/persistence/repository/LashMappingJpaRepository.java` (novo)
**Depende de**: B01, B04
**Reutiliza**: Padrão de `AnamneseEntity` (B05)
**Requisitos**: FIC-05, FIC-06, FIC-08

**Feito quando:**
- [ ] `LashMappingEntity` mapeada para `lash_mappings`; `@ManyToOne(fetch=LAZY, insertable=false, updatable=false)` para `ClientEntity`
- [ ] `LashMappingJpaRepository` tem `findByClientIdOrderByMappingDateDesc(UUID clientId, Pageable)` e query `findAllWithClientSummary` que agrega `COUNT` e `MAX(mappingDate)` por cliente
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B07: Infra — Mappers + RepositoryImpls de Anamnese e Token

**O que**: Criar mappers e implementações de repositório para Anamnese e AnamneseToken
**Onde**:
- `infrastructure/persistence/mapper/AnamneseMapper.java` (novo)
- `infrastructure/persistence/mapper/AnamneseTokenMapper.java` (novo)
- `infrastructure/persistence/repository/AnamneseRepositoryImpl.java` (novo)
- `infrastructure/persistence/repository/AnamneseTokenRepositoryImpl.java` (novo)
**Depende de**: B04, B05
**Reutiliza**: Padrão de `ClientMapper.java`, `ClientRepositoryImpl.java`
**Requisitos**: FIC-02, FIC-04

**Feito quando:**
- [ ] `AnamneseMapper`: converte `AnamneseEntity` ↔ `Anamnese`; resolve `clientName` via associação lazy
- [ ] `AnamneseTokenMapper`: converte `AnamneseTokenEntity` ↔ `AnamneseToken`
- [ ] `AnamneseRepositoryImpl`: implementa todos os métodos do port/out; `findAllWithClientSummary` mapeia resultado da query de resumo para `AnamneseSummaryResult`
- [ ] `AnamneseTokenRepositoryImpl`: implementa `save`, `findByToken` (chama `findByTokenAndUsedFalseAndExpiresAtAfter`), `deleteExpired`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B08: Infra — Mapper + RepositoryImpl de LashMapping

**O que**: Criar mapper e implementação de repositório para LashMapping
**Onde**:
- `infrastructure/persistence/mapper/LashMappingMapper.java` (novo)
- `infrastructure/persistence/repository/LashMappingRepositoryImpl.java` (novo)
**Depende de**: B04, B06
**Reutiliza**: Padrão de `AnamneseMapper` (B07)
**Requisitos**: FIC-05, FIC-06, FIC-08

**Feito quando:**
- [ ] `LashMappingMapper`: converte `LashMappingEntity` ↔ `LashMapping`; resolve `clientName` via associação lazy
- [ ] `LashMappingRepositoryImpl`: implementa todos os métodos do port/out; `findAllWithClientSummary` delega para a query de agregação do JpaRepository
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B09: Use Cases — Anamnese (profissional)

**O que**: Implementar os 4 use cases de anamnese para o fluxo autenticado
**Onde**: `application/usecase/` — 4 arquivos novos:
- `GetOrCreateAnamneseUseCaseImpl.java`
- `SaveAnamneseUseCaseImpl.java`
- `ListAnamneseSummaryUseCaseImpl.java`
- `GenerateAnamneseLinkUseCaseImpl.java`
**Depende de**: B03, B07
**Reutiliza**: Padrão de `CreateClientUseCaseImpl`, `GetClientUseCaseImpl`
**Requisitos**: FIC-02, FIC-03, FIC-04

**Feito quando:**
- [ ] `GetOrCreateAnamneseUseCaseImpl`: busca por `clientId`; se não existe, cria registro vazio (apenas com `clientId`); retorna a ficha existente ou nova
- [ ] `SaveAnamneseUseCaseImpl`: faz upsert — `findByClientId` para obter id existente, depois `save`; atualiza `updatedAt`
- [ ] `ListAnamneseSummaryUseCaseImpl`: delega para `AnamneseRepository.findAllWithClientSummary`; retorna `Page<AnamneseSummaryResult>`
- [ ] `GenerateAnamneseLinkUseCaseImpl`: gera token UUID de 64 chars; define `expiresAt = now + 7 dias`; salva via `AnamneseTokenRepository`; retorna URL completa no formato `{frontendBaseUrl}/ficha/{token}` (URL base configurada via `@Value("${app.frontend-url}")`)
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B10: Use Cases — Anamnese pública (sem auth)

**O que**: Implementar os 2 use cases de anamnese pública (fluxo da cliente via token)
**Onde**: `application/usecase/` — 2 arquivos novos:
- `GetAnamneseByTokenUseCaseImpl.java`
- `SubmitAnamneseByTokenUseCaseImpl.java`
**Depende de**: B03, B07
**Reutiliza**: Padrão de B09
**Requisitos**: FIC-02 (contexto: envio por link — fora do escopo da spec P1, mas o backend já deve suportar)

**Feito quando:**
- [ ] `GetAnamneseByTokenUseCaseImpl`: busca token via `AnamneseTokenRepository.findByToken`; lança `AnamneseTokenNotFoundException` se ausente; a query JPA já filtra `used=false` e `expiresAt > now`; retorna dados do cliente + anamnese atual (se existir)
- [ ] `SubmitAnamneseByTokenUseCaseImpl`: anotado com `@Transactional`; valida token (mesmo fluxo acima); salva dados da ficha via `SaveAnamneseUseCaseImpl`; marca token como `used=true`; salva token atualizado
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B11: Use Cases — LashMapping (5 use cases)

**O que**: Implementar os 5 use cases de mapping de cílios
**Onde**: `application/usecase/` — 5 arquivos novos:
- `CreateLashMappingUseCaseImpl.java`
- `UpdateLashMappingUseCaseImpl.java`
- `GetLashMappingUseCaseImpl.java`
- `ListMappingsByClientUseCaseImpl.java`
- `ListMappingSummaryUseCaseImpl.java`
**Depende de**: B03, B07, B08
**Reutiliza**: Padrão de B09, `CreateClientUseCaseImpl`
**Requisitos**: FIC-05, FIC-06, FIC-07, FIC-08

**Feito quando:**
- [ ] `CreateLashMappingUseCaseImpl`: cria novo `LashMapping` com `mappingDate = today`; salva e retorna
- [ ] `UpdateLashMappingUseCaseImpl`: busca por id; lança `LashMappingNotFoundException` se não encontrado; substitui todos os campos editáveis; atualiza `updatedAt`
- [ ] `GetLashMappingUseCaseImpl`: busca por id; lança `LashMappingNotFoundException` se não encontrado
- [ ] `ListMappingsByClientUseCaseImpl`: retorna `Page<LashMapping>` ordenado por `mappingDate DESC` para o clientId informado
- [ ] `ListMappingSummaryUseCaseImpl`: delega para `LashMappingRepository.findAllWithClientSummary`
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B12: DTOs — records de request e response

**O que**: Criar todos os DTOs do módulo de Fichas
**Onde**: `adapter/web/dto/` — 9 arquivos novos:
- `SaveAnamneseRequest.java`
- `AnamneseResponse.java`
- `AnamneseSummaryResponse.java`
- `AnamnesePublicResponse.java`
- `SubmitAnamneseRequest.java`
- `GenerateLinkResponse.java`
- `CreateMappingRequest.java`
- `MappingResponse.java`
- `MappingSummaryResponse.java`
**Depende de**: B03
**Reutiliza**: Padrão de `ClientResponse.java`, `FinancialEntryResponse.java`
**Requisitos**: FIC-02, FIC-04, FIC-05, FIC-08

**Feito quando:**
- [ ] `AnamneseResponse`: todos os campos do modelo + `clientName`; datas como `String` ISO
- [ ] `AnamneseSummaryResponse`: `clientId`, `clientName`, `hasAnamnese`, `updatedAt`
- [ ] `AnamnesePublicResponse`: `clientName`, `clientPhone` (para pré-preencher), `anamnese` (`AnamneseResponse | null`)
- [ ] `GenerateLinkResponse`: `{ url: String }`
- [ ] `MappingResponse`: todos os campos do modelo incluindo `canvasData`, `photoBefore`, `photoAfter`
- [ ] `MappingSummaryResponse`: `clientId`, `clientName`, `mappingCount`, `lastMappingDate`
- [ ] `CreateMappingRequest` e `SaveAnamneseRequest`: campos sem `@NotNull` salvo os obrigatórios por negócio
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B13: Controllers de Anamnese (autenticado + público)

**O que**: Criar `AnamneseController` e `PublicAnamneseController`
**Onde**:
- `adapter/web/controller/AnamneseController.java` (novo)
- `adapter/web/controller/PublicAnamneseController.java` (novo)
**Depende de**: B09, B10, B12, B15
**Reutiliza**: Padrão de `ClientController.java`, `FinancialController.java`
**Requisitos**: FIC-02, FIC-03, FIC-04

**Feito quando:**
- [ ] `AnamneseController`: `@RequestMapping("/api/anamnese")`, 4 endpoints: `GET` (lista sumário), `GET /{clientId}` (busca ou 204), `PUT /{clientId}` (upsert), `POST /{clientId}/link` (gera token)
- [ ] `GET /api/anamnese` aceita `search`, `page`, `size`
- [ ] `PUT /{clientId}` retorna 200 com `AnamneseResponse`
- [ ] `POST /{clientId}/link` retorna 200 com `GenerateLinkResponse`
- [ ] `PublicAnamneseController`: `@RequestMapping("/api/public/anamnese")`, 2 endpoints: `GET /{token}` e `POST /{token}`
- [ ] `GET /{token}` retorna `AnamnesePublicResponse` (200) ou trata exceções de token via `@ExceptionHandler`
- [ ] `POST /{token}` retorna 200 OK após submissão
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B14: `LashMappingController` + SecurityConfig

**O que**: Criar `LashMappingController` e atualizar `SecurityConfig` para `permitAll` em `/api/public/**`
**Onde**:
- `adapter/web/controller/LashMappingController.java` (novo)
- `infrastructure/security/SecurityConfig.java` (modificar)
**Depende de**: B11, B12, B15
**Reutiliza**: Padrão de `AnamneseController` (B13)
**Requisitos**: FIC-05, FIC-06, FIC-07, FIC-08

**Feito quando:**
- [ ] `LashMappingController`: `@RequestMapping("/api/mappings")`, 6 endpoints conforme design
- [ ] `GET /api/mappings` aceita `search`, `page`, `size`; retorna `Page<MappingSummaryResponse>`
- [ ] `GET /api/mappings/client/{clientId}` aceita `page`, `size`; retorna `Page<MappingResponse>`
- [ ] `POST /api/mappings/client/{clientId}` retorna 201 com `MappingResponse`
- [ ] `DELETE /api/mappings/{id}` retorna 204
- [ ] `SecurityConfig`: adiciona `.requestMatchers("/api/public/**").permitAll()` antes das regras autenticadas
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### B15: `GlobalExceptionHandler` — handlers para exceções de Fichas

**O que**: Adicionar 5 novos `@ExceptionHandler` para as exceções do módulo de Fichas
**Onde**: `adapter/web/controller/GlobalExceptionHandler.java` (modificar)
**Depende de**: B02
**Reutiliza**: Padrão dos handlers existentes (404, 409, 410)
**Requisitos**: FIC-02, FIC-05

**Feito quando:**
- [ ] `AnamneseNotFoundException` → 404
- [ ] `AnamneseTokenNotFoundException` → 404 com mensagem "Link inválido ou expirado"
- [ ] `AnamneseTokenExpiredException` → 410 com mensagem "Este link expirou. Peça um novo à profissional."
- [ ] `AnamneseTokenAlreadyUsedException` → 409 com mensagem "Esta ficha já foi preenchida."
- [ ] `LashMappingNotFoundException` → 404
- [ ] Gate: `mvn compile` sem erros

**Verificar**: `mvn compile`

---

### F01: `fichas.model.ts` — interfaces TypeScript

**O que**: Criar o arquivo de modelos TypeScript do módulo de Fichas
**Onde**: `lash-frontend/src/app/features/fichas/models/fichas.model.ts`
**Depende de**: Nenhum
**Reutiliza**: Padrão de `financial.model.ts`, `inventory.model.ts`
**Requisitos**: FIC-02, FIC-04, FIC-05, FIC-06, FIC-08

**Feito quando:**
- [ ] `Anamnese` interface com todos os campos + `sleepSide: 'DIREITO' | 'ESQUERDO' | 'AMBOS'`
- [ ] `AnamneseSummary` interface: `clientId`, `clientName`, `hasAnamnese`, `updatedAt`
- [ ] `LashMapping` interface com todos os campos incluindo `canvasData`, `photoBefore`, `photoAfter`
- [ ] `MappingSummary` interface: `clientId`, `clientName`, `mappingCount`, `lastMappingDate`
- [ ] Interfaces de request: `SaveAnamneseRequest`, `CreateMappingRequest`, `UpdateMappingRequest`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F02: Serviços Angular — `AnamneseService` e `MappingService`

**O que**: Criar os 2 serviços de comunicação com a API de Fichas
**Onde**:
- `lash-frontend/src/app/features/fichas/services/anamnese.service.ts` (novo)
- `lash-frontend/src/app/features/fichas/services/mapping.service.ts` (novo)
**Depende de**: F01
**Reutiliza**: Padrão de `client.service.ts`, `financial.service.ts`
**Requisitos**: FIC-02, FIC-04, FIC-05, FIC-06, FIC-08

**Feito quando:**
- [ ] `AnamneseService`: `inject(HttpClient)`; métodos: `list(search, page, size)`, `get(clientId)`, `save(clientId, req)`, `generateLink(clientId)`, `getByToken(token)`, `submitByToken(token, req)`
- [ ] `MappingService`: `inject(HttpClient)`; métodos: `list(search, page, size)`, `listByClient(clientId, page, size)`, `get(id)`, `create(clientId, req)`, `update(id, req)`, `delete(id)`
- [ ] URLs base: `/api/anamnese` e `/api/mappings`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F03: NgRx — actions, reducer, selectors

**O que**: Criar o slice NgRx completo para o módulo de Fichas
**Onde**:
- `lash-frontend/src/app/features/fichas/store/fichas.actions.ts` (novo)
- `lash-frontend/src/app/features/fichas/store/fichas.reducer.ts` (novo)
- `lash-frontend/src/app/features/fichas/store/fichas.selectors.ts` (novo)
**Depende de**: F01
**Reutiliza**: Padrão de `financial.actions.ts`, `inventory.actions.ts`
**Requisitos**: FIC-01 a FIC-08

**Feito quando:**
- [ ] Actions criadas para todos os fluxos: LoadAnamneseSummaries, LoadAnamnese, SaveAnamnese, GenerateLink, GetByToken, SubmitByToken, LoadMappingSummaries, LoadClientMappings, LoadMapping, CreateMapping, UpdateMapping, DeleteMapping, SetSearch, SetPage (+ Success/Failure variants)
- [ ] `FichasState` conforme design (anamneseSummaries, totalAnamneses, currentAnamnese, generatedLink, mappingSummaries, totalMappings, clientMappings, currentMapping, search, page, isLoading, error)
- [ ] Reducer inicializa e trata todos os Success/Failure
- [ ] Seletores: `selectAnamneseSummaries`, `selectCurrentAnamnese`, `selectGeneratedLink`, `selectMappingSummaries`, `selectClientMappings`, `selectCurrentMapping`, `selectIsLoading`, `selectSearch`, `selectPage`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F04: NgRx — effects

**O que**: Criar os effects do módulo de Fichas
**Onde**: `lash-frontend/src/app/features/fichas/store/fichas.effects.ts` (novo)
**Depende de**: F02, F03
**Reutiliza**: Padrão de `financial.effects.ts`, `inventory.effects.ts`
**Requisitos**: FIC-01 a FIC-08

**Feito quando:**
- [ ] `loadAnamneseSummaries$`: chama `AnamneseService.list()` com search+page do state
- [ ] `setSearch$`, `setPage$`: disparam `loadAnamneseSummaries` ou `loadMappingSummaries` conforme contexto ativo
- [ ] `loadAnamnese$`: chama `AnamneseService.get(clientId)`
- [ ] `saveAnamnese$`: chama `AnamneseService.save(clientId, req)`; exibe snackbar "Ficha salva com sucesso"
- [ ] `generateLink$`: chama `AnamneseService.generateLink(clientId)`; despacha `generateLinkSuccess` com URL
- [ ] `loadMappingSummaries$`, `loadClientMappings$`, `loadMapping$`: chamam `MappingService`
- [ ] `createMapping$`, `updateMapping$`: salvam + navegam para histórico; exibem snackbar
- [ ] `deleteMapping$`: deleta + recarrega histórico
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F05: `AnamneseListComponent` — lista de clientes com status da ficha

**O que**: Criar o componente de listagem de anamneses
**Onde**: `lash-frontend/src/app/features/fichas/components/anamnese-list/`
- `anamnese-list.component.ts`
- `anamnese-list.component.html`
- `anamnese-list.component.css`
**Depende de**: F03
**Reutiliza**: Padrão de `client-list.component`; `MatTableModule`, `MatPaginatorModule`
**Requisitos**: FIC-04

**Feito quando:**
- [ ] Lista clientes em tabela: foto/avatar, nome, status ("Preenchida" em verde / "Não preenchida" em cinza com data quando preenchida)
- [ ] Campo de busca com debounce 300ms; despacha `setSearch`
- [ ] `MatPaginator` com tamanho padrão 20
- [ ] Clique em linha navega para `/fichas/anamnese/:clientId`
- [ ] Estado vazio: "Nenhum cliente encontrado. Selecione um cliente para começar."
- [ ] Mobile: layout de card em vez de tabela (colunas col-date ocultas)
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F06: `AnamneseFormComponent` — formulário completo de anamnese

**O que**: Criar o formulário de anamnese (preenchimento pela profissional)
**Onde**: `lash-frontend/src/app/features/fichas/components/anamnese-form/`
- `anamnese-form.component.ts`
- `anamnese-form.component.html`
- `anamnese-form.component.css`
**Depende de**: F03
**Reutiliza**: `ReactiveFormsModule`, `MatSlideToggleModule`, `MatCheckboxModule`, `MatRadioModule`
**Requisitos**: FIC-02, FIC-03

**Feito quando:**
- [ ] Seção 1 "Dados do cliente": campos de texto; nome e celular pré-preenchidos do cadastro (readonly); demais campos editáveis
- [ ] Seção 2 "Saúde": 10 campos `mat-slide-toggle` (sim/não) + 1 `mat-radio-group` para `sleepSide` (Direito / Esquerdo / Dois lados)
- [ ] Seção 3 "Termo": texto estático + `mat-checkbox`; ao marcar exibe badge verde "Termo aceito ✓" com data/hora
- [ ] Botão "Salvar ficha": despacha `saveAnamnese`; desabilitado durante `isLoading$`
- [ ] Botão "Gerar link para cliente": abre dialog com a URL copiável; despacha `generateLink`
- [ ] `ngOnInit`: despacha `loadAnamnese({ clientId })` para pré-preencher
- [ ] Mobile: campos em coluna única, seções com `mat-expansion-panel`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F07: `MappingListComponent` — lista de clientes com contagem de fichas

**O que**: Criar o componente de listagem de summaries de mapping
**Onde**: `lash-frontend/src/app/features/fichas/components/mapping-list/`
- `mapping-list.component.ts`
- `mapping-list.component.html`
- `mapping-list.component.css`
**Depende de**: F03
**Reutiliza**: Padrão de `AnamneseListComponent` (F05)
**Requisitos**: FIC-08

**Feito quando:**
- [ ] Lista clientes: foto/avatar, nome, badge "X fichas" (ou "Nenhuma ficha" em cinza), data da última ficha
- [ ] Campo de busca com debounce 300ms
- [ ] `MatPaginator` com tamanho padrão 20
- [ ] Clique em linha navega para `/fichas/mapping/:clientId`
- [ ] Estado vazio: "Nenhum cliente encontrado."
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F08: `MappingHistoryComponent` — histórico cronológico de fichas de um cliente

**O que**: Criar o componente de histórico de mappings por cliente
**Onde**: `lash-frontend/src/app/features/fichas/components/mapping-history/`
- `mapping-history.component.ts`
- `mapping-history.component.html`
- `mapping-history.component.css`
**Depende de**: F03
**Reutiliza**: Padrão de listas com cards; `datePtbr` pipe
**Requisitos**: FIC-06, FIC-07

**Feito quando:**
- [ ] `ngOnInit`: despacha `loadClientMappings({ clientId })` via route params
- [ ] Lista cards cronológicos (mais recente primeiro): data, tipo de mapping, curvatura, resumo dos campos principais
- [ ] Botão "+ Nova ficha" navega para `/fichas/mapping/:clientId/nova`
- [ ] Clique em card navega para `/fichas/mapping/:clientId/:id` (modo edição/visualização)
- [ ] Estado vazio: "Nenhuma ficha registrada. Crie a primeira ficha."
- [ ] Mobile: cards empilhados com campos resumidos
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F09: `EyeCanvasComponent` — canvas de desenho do olho

**O que**: Criar o componente standalone de canvas com SVG de fundo, paleta e controles
**Onde**: `lash-frontend/src/app/features/fichas/components/eye-canvas/`
- `eye-canvas.component.ts`
- `eye-canvas.component.html`
- `eye-canvas.component.css`
**Depende de**: Nenhum (componente standalone sem dependência de store)
**Reutiliza**: HTML5 Canvas API nativa; sem bibliotecas externas
**Requisitos**: FIC-05

**Feito quando:**
- [ ] `@Input() initialData: string | null` — carrega imagem base64 ao inicializar; `@Output() canvasChange = new EventEmitter<string>()`
- [ ] SVG de olho (contorno ovóide estilizado) como fundo via `<svg>` inline; canvas sobreposto via `position: absolute`
- [ ] Eventos de desenho: `mousedown/mousemove/mouseup` + `touchstart/touchmove/touchend` (preventDefault nos touch events para não rolar a página)
- [ ] `history: ImageData[]` com `pushState()` a cada `mouseup`/`touchend`; botão "Desfazer" chama `undo()` (pop + restore); botão "Refazer" com `redoStack`
- [ ] Paleta: 6 chips de cor fixa (`#000000`, `#C8A2A2`, `#D4AF37`, `#e53935`, `#1565c0`, `#2e7d32`) + `<input type="color">` para cor personalizada
- [ ] Seletor de espessura: 3 botões (Fina=2px, Média=5px, Grossa=10px) — botão ativo destacado
- [ ] Botão "Limpar" chama `clearCanvas()` (fill branco + recarrega SVG de fundo se necessário)
- [ ] `canvasChange.emit(canvas.toDataURL('image/png'))` em cada `mouseup`/`touchend`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F10: `MappingFormComponent` — formulário completo + canvas + fotos

**O que**: Criar o formulário de mapping com campos técnicos, canvas e upload de fotos
**Onde**: `lash-frontend/src/app/features/fichas/components/mapping-form/`
- `mapping-form.component.ts`
- `mapping-form.component.html`
- `mapping-form.component.css`
**Depende de**: F03, F09
**Reutiliza**: `ReactiveFormsModule`, `MatSelectModule`, `MatInputModule`; `EyeCanvasComponent` (F09)
**Requisitos**: FIC-05

**Feito quando:**
- [ ] `ngOnInit`: se route tem `:id` despacha `loadMapping({ id })`; modo "novo" inicia form em branco com `mappingDate = today`
- [ ] Grid 2 colunas no desktop, 1 coluna no mobile; todos os campos do design com tipos corretos (select, number, text, textarea)
- [ ] `<app-eye-canvas>` integrado; `(canvasChange)` atualiza `formGroup.canvasData`
- [ ] Upload de foto "Antes": `<input type="file" accept="image/*">`; ao selecionar, comprime via Canvas resize (max 800px, quality 0.7) + converte para base64
- [ ] Upload de foto "Depois": mesmo fluxo
- [ ] Preview das fotos ao carregar ficha existente (`<img [src]="photoBefore">`)
- [ ] Botão "Salvar ficha": despacha `createMapping` ou `updateMapping` conforme contexto
- [ ] Botão "Voltar": navega para `/fichas/mapping/:clientId`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F11: `FichasComponent` (shell) + rotas da feature

**O que**: Montar o componente shell com tabs Anamnese | Mapping e registrar as rotas do módulo
**Onde**:
- `lash-frontend/src/app/features/fichas/fichas.component.ts` (novo)
- `lash-frontend/src/app/features/fichas/fichas.component.html` (novo)
- `lash-frontend/src/app/features/fichas/fichas.component.css` (novo)
- `lash-frontend/src/app/features/fichas/fichas.routes.ts` (novo)
**Depende de**: F04, F05, F06, F07, F08, F10
**Reutiliza**: Padrão de `financial.component` (shell com sub-nav); `MatTabsModule`
**Requisitos**: FIC-01, FIC-04, FIC-08

**Feito quando:**
- [ ] Shell com `mat-tab-group`: aba "Anamnese" (route child) + aba "Mapping" (route child); troca de aba navega entre `/fichas/anamnese` e `/fichas/mapping`
- [ ] `fichas.routes.ts` define: `/fichas` (shell), `/fichas/anamnese`, `/fichas/anamnese/:clientId`, `/fichas/mapping`, `/fichas/mapping/:clientId`, `/fichas/mapping/:clientId/nova`, `/fichas/mapping/:clientId/:id`
- [ ] Feature registrada no `app.routes.ts` como lazy: `{ path: 'fichas', loadChildren: () => import('./features/fichas/fichas.routes').then(m => m.fichasRoutes), canActivate: [authGuard] }`
- [ ] `provideState(fichasFeature)` e `provideEffects(FichasEffects)` registrados no `fichas.routes.ts` via `providers`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F12: `PublicAnamneseComponent` — layout público sem auth

**O que**: Criar a página pública para preenchimento de anamnese via token
**Onde**:
- `lash-frontend/src/app/features/fichas/public/public-anamnese.component.ts` (novo)
- `lash-frontend/src/app/features/fichas/public/public-anamnese.component.html` (novo)
- `lash-frontend/src/app/features/fichas/public/public-anamnese.component.css` (novo)
- `lash-frontend/src/app/features/fichas/public/public-anamnese.routes.ts` (novo)
**Depende de**: F02
**Reutiliza**: Mesmos campos de `AnamneseFormComponent` mas sem NgRx (chama service diretamente); sem sidebar/bottom-nav
**Requisitos**: FIC-02 (contexto de link público)

**Feito quando:**
- [ ] `ngOnInit`: lê `token` do route params; chama `AnamneseService.getByToken(token)` direto (sem store)
- [ ] Estados: loading → formulário pré-preenchido → sucesso ("Ficha salva com sucesso!") / erro ("Link inválido", "Link expirado", "Ficha já preenchida")
- [ ] Layout limpo: logo no topo, formulário centralizado, sem sidebar nem bottom nav
- [ ] Formulário idêntico ao `AnamneseFormComponent` mas self-contained (ReactiveForm local)
- [ ] Ao salvar: chama `AnamneseService.submitByToken(token, req)`; exibe tela de sucesso
- [ ] `public-anamnese.routes.ts`: define rota `''` com `PublicAnamneseComponent`
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F13: App routing — rota pública `/ficha/:token` + proteção authGuard

**O que**: Registrar rota pública `/ficha/:token` fora do `authGuard` no `app.routes.ts`
**Onde**: `lash-frontend/src/app/app.routes.ts` (modificar)
**Depende de**: F11, F12
**Reutiliza**: Padrão de rotas lazy existentes
**Requisitos**: FIC-02 (link público)

**Feito quando:**
- [ ] Rota `/ficha/:token` adicionada **fora** do bloco protegido pelo `authGuard`: `{ path: 'ficha/:token', loadChildren: () => import('./features/fichas/public/public-anamnese.routes').then(m => m.publicAnamneseRoutes) }`
- [ ] Rota `/fichas` já adicionada em F11 dentro do bloco protegido
- [ ] Nenhuma rota existente alterada ou quebrada
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F14: Sidebar + Bottom Nav — adicionar Fichas + reestruturar "Mais"

**O que**: Adicionar "Fichas" na sidebar e reestruturar o bottom nav para 5 fixos + MatBottomSheet "Mais"
**Onde**:
- `lash-frontend/src/app/shared/components/layout/sidebar/sidebar.component.ts` (modificar)
- `lash-frontend/src/app/shared/components/layout/bottom-nav/bottom-nav.component.ts` (modificar)
- `lash-frontend/src/app/shared/components/layout/bottom-nav/bottom-nav.component.html` (modificar)
- `lash-frontend/src/app/shared/components/layout/bottom-nav/bottom-nav.component.css` (modificar)
**Depende de**: F13
**Reutiliza**: Padrão da sidebar existente; `MatBottomSheetModule`
**Requisitos**: FIC-01

**Feito quando:**
- [ ] Sidebar: item "Fichas" com ícone `assignment` adicionado após "Estoque", rota `/fichas`
- [ ] Bottom nav reestruturado para 5 fixos: Dashboard, Agenda, Clientes, Fichas, Financeiro
- [ ] Item "Mais" (ícone `menu_open`) abre `MatBottomSheet` com lista contendo: Serviços, Estoque
- [ ] `BottomSheetMoreComponent` criado inline (componente simples com lista de nav items)
- [ ] Bottom nav CSS: 5 itens cabem confortavelmente (flex 1/5 cada); labels ajustados se necessário
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

### F15: `ClientDetailComponent` — botões de acesso às fichas

**O que**: Adicionar botões "Ficha de Anamnese" e "Fichas de Mapping" na tela de detalhe do cliente
**Onde**: `lash-frontend/src/app/features/clients/components/client-detail/client-detail.component.html` (modificar)  
`lash-frontend/src/app/features/clients/components/client-detail/client-detail.component.css` (modificar)
**Depende de**: F13
**Reutiliza**: Botões existentes no client-detail; `RouterModule`
**Requisitos**: FIC-03, FIC-07

**Feito quando:**
- [ ] Seção "Fichas" adicionada na tela de detalhe do cliente com 2 botões:
  - "Ficha de Anamnese" com ícone `assignment` → `[routerLink]="['/fichas/anamnese', client.id]"`
  - "Fichas de Mapping" com ícone `draw` → `[routerLink]="['/fichas/mapping', client.id]"`
- [ ] Botões com estilo consistente com os demais botões de ação do client-detail
- [ ] Gate: `npm run build` sem erros

**Verificar**: `npm run build`

---

## Mapa de Paralelismo

```
BACKEND
Phase 1:  B01 ─┐
          B02 ─┘ (em paralelo)

Phase 2:  B03 ─┐ (em paralelo, após B02)
          B04 ─┘

Phase 3:  B05 ─┐ (em paralelo, após B01+B04)
          B06 ─┘

Phase 4:  B07 ─┐ (em paralelo, após B04+B05 / B04+B06)
          B08 ─┘

Phase 5:  B09 ─┐
          B10 ─┤ (em paralelo, após B03+B07 / B03+B07+B08)
          B11 ─┘

Phase 6:  B12 ─┐ (em paralelo, após B03 / B02)
          B15 ─┘
          B13   (após B09+B10+B12+B15)
          B14   (após B11+B12+B15)

FRONTEND (após Fase 6 backend)
Phase 7a: F01

Phase 7b: F02 ─┐ (em paralelo, após F01)
          F03 ─┘

Phase 7c: F04   (após F02+F03)

Phase 7d: F05 ─┐
          F06 ─┤
          F07 ─┤ (em paralelo, após F03)
          F08 ─┤
          F09 ─┘

Phase 7e: F10   (após F03+F09)

Phase 7f: F11   (após F04+F05+F06+F07+F08+F10)
          F12   (após F02)

Phase 7g: F13   (após F11+F12)

Phase 7h: F14 ─┐ (em paralelo, após F13)
          F15 ─┘
```

---

## Validação de Granularidade

| Task | Escopo | Status |
|---|---|---|
| B01: Migration V7 | 1 arquivo SQL | ✅ |
| B02: Domain models + exceções | 8 arquivos — camada domain completa | ✅ |
| B03: Ports/In (11 interfaces) | 11 interfaces coesas, mesma camada | ✅ |
| B04: Ports/Out (3 interfaces) | 3 interfaces coesas, mesma camada | ✅ |
| B05: Anamnese+Token Entity + JpaRepo | 4 arquivos de infra — tabelas relacionadas | ✅ |
| B06: LashMapping Entity + JpaRepo | 2 arquivos de infra coesos | ✅ |
| B07: Anamnese Mappers + RepoImpls | 4 arquivos de infra coesos | ✅ |
| B08: LashMapping Mapper + RepoImpl | 2 arquivos de infra coesos | ✅ |
| B09: 4 use cases anamnese (profissional) | 4 arquivos — mesma entidade, mesma camada | ✅ |
| B10: 2 use cases anamnese pública | 2 arquivos — fluxo de token | ✅ |
| B11: 5 use cases LashMapping | 5 arquivos — mesma entidade, mesma camada | ✅ |
| B12: 9 DTOs | 9 arquivos — mesma camada, sem lógica | ✅ |
| B13: AnamneseController + PublicController | 2 controllers relacionados | ✅ |
| B14: LashMappingController + SecurityConfig | 1 controller + 1 config | ✅ |
| B15: GlobalExceptionHandler (5 handlers) | 1 arquivo modificado | ✅ |
| F01: fichas.model.ts | 1 arquivo TypeScript | ✅ |
| F02: AnamneseService + MappingService | 2 serviços coesos do mesmo domínio | ✅ |
| F03: actions + reducer + selectors | 3 arquivos NgRx coesos | ✅ |
| F04: effects | 1 arquivo NgRx | ✅ |
| F05: AnamneseListComponent | 1 componente (3 arquivos) | ✅ |
| F06: AnamneseFormComponent | 1 componente (3 arquivos) | ✅ |
| F07: MappingListComponent | 1 componente (3 arquivos) | ✅ |
| F08: MappingHistoryComponent | 1 componente (3 arquivos) | ✅ |
| F09: EyeCanvasComponent | 1 componente standalone (3 arquivos) | ✅ |
| F10: MappingFormComponent | 1 componente (3 arquivos) | ✅ |
| F11: FichasComponent shell + rotas | 1 shell + routes | ✅ |
| F12: PublicAnamneseComponent | 1 componente público (3 arquivos + routes) | ✅ |
| F13: App routing update | 1 arquivo modificado | ✅ |
| F14: Sidebar + Bottom Nav | 2 arquivos modificados + 1 componente novo | ✅ |
| F15: ClientDetail integration | 2 arquivos modificados | ✅ |

---

## Rastreabilidade de Requisitos

| Requisito | Task(s) |
|---|---|
| FIC-01 Menu Fichas | F11, F14 |
| FIC-02 Anamnese preenchimento | B01, B02, B03, B04, B05, B07, B09, B10, B12, B13, B15, F01, F02, F03, F04, F06, F12 |
| FIC-03 Anamnese pelo cadastro | B13, F06, F15 |
| FIC-04 Listagem anamneses | B09, B12, B13, F01, F03, F04, F05 |
| FIC-05 Mapping preenchimento + canvas | B01, B02, B03, B04, B06, B08, B11, B12, B14, F01, F02, F03, F04, F09, F10 |
| FIC-06 Histórico de mappings | B11, B12, B14, F03, F04, F08 |
| FIC-07 Mapping pelo cadastro | B14, F08, F15 |
| FIC-08 Listagem mappings | B11, B12, B14, F01, F03, F04, F07 |
| FIC-09 Tipos de mapping | F10 (valores fixos nos selects) |
| FIC-10 Impressão anamnese | — (P3 — fora desta entrega) |
