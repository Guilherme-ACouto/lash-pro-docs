# Refatoração Multi-Módulo Maven — Tasks

**Design**: `.specs/features/refactor-multimodulo/design.md`
**Status**: Draft

---

## Plano de Execução

```
T01 (parent POM)
  └──→ T02 (lash-core estrutura)
         └──→ T03 (lash-clients)
         └──→ T04 (lash-services)
              └──→ T05 (lash-appointments)  ← depende de T03 + T04
                   └──→ T06 (lash-finance)
                        └──→ T07 (lash-stock)
  └──→ T08 (lash-fichas)                   ← depende de T02 + T03
  └──→ T09 (lash-dashboard)                ← depende de T03–T07
  └──→ T10 (lash-app + Flyway)             ← depende de todos
  └──→ T11 (migrations: redistribuir)      ← depende de T02–T09
  └──→ T12 (GlobalExceptionHandler)        ← depende de T10
  └──→ T13 (testes de clientes migrados)   ← depende de T03
  └──→ T14 (verificação final + limpeza)   ← depende de todos
```

---

## Breakdown das Tarefas

### T01: Parent POM

**O que**: Transformar `lash-backend/pom.xml` em parent POM com todos os sub-módulos declarados e dependências gerenciadas centralmente
**Onde**: `lash-backend/pom.xml`
**Depende de**: Nenhum

**Feito quando:**
- [ ] `<packaging>pom</packaging>` definido
- [ ] `<modules>` lista todos os 9 sub-módulos: lash-core, lash-clients, lash-services, lash-appointments, lash-finance, lash-stock, lash-fichas, lash-dashboard, lash-app
- [ ] `<dependencyManagement>` centraliza versões de: spring-boot-starter-*, jjwt, lombok, postgresql, flyway
- [ ] `<build><pluginManagement>` define `spring-boot-maven-plugin` com `skip=true` (default para todos exceto lash-app)
- [ ] Gate: `mvn validate` sem erros no parent

---

### T02: lash-core — estrutura + código

**O que**: Criar o módulo lash-core com toda a infraestrutura compartilhada
**Onde**: `lash-backend/lash-core/`
**Depende de**: T01

**Arquivos a criar (pom.xml):**
- `lash-core/pom.xml` com dependências: spring-boot-starter-security, spring-boot-starter-web, spring-boot-starter-data-jpa, spring-boot-starter-mail, jjwt-*, lombok; sem spring-boot-maven-plugin repackage

**Arquivos a mover (do monólito atual → lash-core, atualizando pacote para `com.lashmanager.core`):**
- `domain/model/User.java`, `UserRole.java`
- `domain/exception/BusinessException.java`, `DomainException.java`, `InvalidCredentialsException.java`, `TokenExpiredException.java`, `UserNotFoundException.java`
- `domain/port/in/LoginUseCase.java`, `RefreshTokenUseCase.java`, `ForgotPasswordUseCase.java`
- `domain/port/out/UserRepository.java`, `TokenPort.java`, `EmailPort.java`
- `application/usecase/LoginUseCaseImpl.java`, `RefreshTokenUseCaseImpl.java`, `ForgotPasswordUseCaseImpl.java`
- `infrastructure/security/` (JwtService, JwtAuthFilter, SecurityConfig, UserDetailsServiceImpl)
- `infrastructure/email/EmailPortImpl.java`
- `infrastructure/persistence/entity/UserEntity.java`
- `infrastructure/persistence/repository/UserJpaRepository.java`, `UserRepositoryImpl.java`
- `infrastructure/persistence/mapper/UserMapper.java`
- `adapter/web/controller/AuthController.java`
- `adapter/web/dto/LoginRequest.java`, `LoginResponse.java`, `RefreshTokenRequest.java`, `ForgotPasswordRequest.java`, `ErrorResponse.java`

**Arquivos a criar em test:**
- `lash-core/src/test/java/com/lashmanager/core/AbstractIntegrationTest.java` (migrado, pacote atualizado)
- `lash-core/src/test/resources/application-test.yml` (migrado)

**Feito quando:**
- [ ] Todos os arquivos criados com pacote `com.lashmanager.core.*`
- [ ] `lash-core/pom.xml` compila isoladamente: `mvn compile -pl lash-core`
- [ ] Nenhuma referência a pacotes de outros módulos de negócio

---

### T03: lash-clients — módulo

**O que**: Criar o módulo lash-clients com todo o código de clientes
**Onde**: `lash-backend/lash-clients/`
**Depende de**: T02

**pom.xml**: depende de `lash-core`; inclui spring-boot-starter-web, spring-boot-starter-data-jpa, spring-boot-starter-validation, lombok

**Arquivos a mover (pacote → `com.lashmanager.clients`):**
- `domain/model/Client.java`
- `domain/exception/ClientNotFoundException.java`, `ClientAlreadyExistsException.java`, `HasFutureAppointmentsException.java`
- `domain/model/AppointmentSummary.java` (fica em clients pois é retornado pela exceção do módulo)
- `domain/port/in/` (Create/Update/Delete/Get/List/DeactivateClientUseCase)
- `domain/port/out/ClientRepository.java`
- `application/usecase/` (*ClientUseCaseImpl, ClientUseCaseMapper)
- `infrastructure/persistence/` (ClientEntity, ClientJpaRepository, ClientMapper, ClientRepositoryImpl)
- `adapter/web/` (ClientController, CreateClientRequest, UpdateClientRequest, ClientResponse)

**Feito quando:**
- [ ] `mvn compile -pl lash-clients` sem erros
- [ ] Nenhuma referência a pacotes de outros módulos de negócio

---

### T04: lash-services — módulo

**O que**: Criar o módulo lash-services com todo o código de serviços
**Onde**: `lash-backend/lash-services/`
**Depende de**: T02

**pom.xml**: depende de `lash-core`; mesmas starters de T03

**Arquivos a mover (pacote → `com.lashmanager.services`):**
- `domain/model/Service.java`, `ServiceOffered.java`
- `domain/exception/ServiceNotFoundException.java`
- `domain/port/in/` (Create/Update/Delete/Get/List/DeactivateServiceUseCase)
- `domain/port/out/ServiceRepository.java`
- `application/usecase/` (*ServiceUseCaseImpl, ServiceUseCaseMapper)
- `infrastructure/persistence/` (ServiceEntity, ServiceJpaRepository, ServiceMapper, ServiceRepositoryImpl)
- `adapter/web/` (ServiceController, CreateServiceRequest, UpdateServiceRequest, ServiceResponse)

> ⚠️ O `@org.springframework.stereotype.Service` qualificado ainda é necessário — `domain/model/Service.java` conflita com a annotation. Anotar as implementações com `@org.springframework.stereotype.Service` ou usar nome do pacote qualificado. (RENAME-01 ainda deferido)

**Feito quando:**
- [ ] `mvn compile -pl lash-services` sem erros

---

### T05: lash-appointments — módulo

**O que**: Criar o módulo lash-appointments com todo o código de agendamentos
**Onde**: `lash-backend/lash-appointments/`
**Depende de**: T03, T04

**pom.xml**: depende de `lash-core`, `lash-clients`, `lash-services`

**Arquivos a mover (pacote → `com.lashmanager.appointments`):**
- `domain/model/Appointment.java`, `AppointmentStatus.java`
- `domain/exception/AppointmentNotFoundException.java`, `AppointmentConflictException.java`
- `domain/port/in/` (Create/Update/Get/List/ChangeStatusAppointmentUseCase)
- `domain/port/out/AppointmentRepository.java`
- `application/usecase/` (*AppointmentUseCaseImpl, AppointmentUseCaseMapper)
- `infrastructure/persistence/` (AppointmentEntity, AppointmentJpaRepository, AppointmentMapper, AppointmentRepositoryImpl)
- `adapter/web/` (AppointmentController, *AppointmentRequest, AppointmentResponse, CompleteAppointmentRequest)

> `AppointmentEntity` usa `@ManyToOne` para `ClientEntity` e `ServiceEntity` → imports de `com.lashmanager.clients` e `com.lashmanager.services`

**Feito quando:**
- [ ] `mvn compile -pl lash-appointments` sem erros

---

### T06: lash-finance — módulo

**O que**: Criar o módulo lash-finance com todo o código financeiro
**Onde**: `lash-backend/lash-finance/`
**Depende de**: T05

**pom.xml**: depende de `lash-core`, `lash-appointments`

**Arquivos a mover (pacote → `com.lashmanager.finance`):**
- `domain/model/FinancialEntry.java`, `FinancialEntryExpenseType.java`, `FinancialEntryStatus.java`, `FinancialEntryType.java`, `MonthlyFinancialStat.java`
- `domain/exception/FinancialEntryNotFoundException.java`, `FinancialEntryLinkedToAppointmentException.java`
- `domain/port/in/` (Create/Update/Delete/List/Get/Toggle/GetSummaryFinancialUseCase)
- `domain/port/out/FinancialEntryRepository.java`, `FinancialSummaryRepository.java`
- `application/usecase/` (*FinancialUseCaseImpl)
- `infrastructure/persistence/` (FinancialEntryEntity, FinancialEntryJpaRepository, FinancialEntryMapper, FinancialEntryRepositoryImpl, FinancialSummaryRepositoryImpl)
- `adapter/web/` (FinancialController, *FinancialEntryRequest/Response, FinancialSummaryResponse)

**Feito quando:**
- [ ] `mvn compile -pl lash-finance` sem erros

---

### T07: lash-stock — módulo

**O que**: Criar o módulo lash-stock com todo o código de estoque
**Onde**: `lash-backend/lash-stock/`
**Depende de**: T06

**pom.xml**: depende de `lash-core`, `lash-finance` (para criar FinancialEntry ao registrar compra)

**Arquivos a mover (pacote → `com.lashmanager.stock`):**
- `domain/model/InventoryItem.java`, `InventoryMovement.java`, `MovementType.java`, `MovementReason.java`, `PurchasePaymentType.java`
- `domain/exception/InventoryItem*Exception.java`
- `domain/port/in/` (*InventoryItemUseCase, *MovementUseCase)
- `domain/port/out/InventoryItemRepository.java`, `InventoryMovementRepository.java`
- `application/usecase/` (*InventoryUseCaseImpl, InventoryUseCaseMapper)
- `infrastructure/persistence/` (InventoryItem*/Movement* Entity/Jpa/Mapper/Impl)
- `adapter/web/` (InventoryController, *InventoryItemRequest/Response, *MovementRequest/Response)

**Feito quando:**
- [ ] `mvn compile -pl lash-stock` sem erros

---

### T08: lash-fichas — módulo

**O que**: Criar o módulo lash-fichas com anamneses e mappings
**Onde**: `lash-backend/lash-fichas/`
**Depende de**: T02, T03

**pom.xml**: depende de `lash-core`, `lash-clients`

**Arquivos a mover (pacote → `com.lashmanager.fichas`):**
- `domain/model/Anamnese.java`, `AnamneseToken.java`, `LashMapping.java`
- `domain/exception/Anamnese*Exception.java`, `LashMappingNotFoundException.java`
- `domain/port/in/` (*AnamneseUseCase, *LashMappingUseCase)
- `domain/port/out/AnamneseRepository.java`, `AnamneseTokenRepository.java`, `LashMappingRepository.java`
- `application/usecase/` (*AnamneseUseCaseImpl, *LashMappingUseCaseImpl)
- `infrastructure/persistence/` (Anamnese*/LashMapping* Entity/Jpa/Mapper/Impl)
- `adapter/web/` (AnamneseController, PublicAnamneseController, LashMappingController + DTOs)

**Feito quando:**
- [ ] `mvn compile -pl lash-fichas` sem erros

---

### T09: lash-dashboard — módulo

**O que**: Criar o módulo lash-dashboard com o dashboard
**Onde**: `lash-backend/lash-dashboard/`
**Depende de**: T03, T04, T05, T06, T07

**pom.xml**: depende de `lash-core`, `lash-clients`, `lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`

**Arquivos a mover (pacote → `com.lashmanager.dashboard`):**
- `domain/model/dashboard/` (AppointmentCounts, AppointmentDayStat, CashFlowDayStat, TodayAppointmentStat)
- `domain/port/in/GetDashboardUseCase.java`
- `domain/port/out/DashboardRepository.java`
- `application/usecase/GetDashboardUseCaseImpl.java`
- `infrastructure/persistence/DashboardRepositoryImpl.java`
- `adapter/web/controller/DashboardController.java`
- `adapter/web/dto/DashboardResponse.java`

**Feito quando:**
- [ ] `mvn compile -pl lash-dashboard` sem erros

---

### T10: lash-app — entry point + application.yml

**O que**: Criar o módulo lash-app com a classe principal e configuração
**Onde**: `lash-backend/lash-app/`
**Depende de**: T02–T09

**pom.xml**: depende de todos os módulos; `spring-boot-maven-plugin` com repackage habilitado

**Arquivos:**
- `LashManagerApplication.java` com `@SpringBootApplication(scanBasePackages = "com.lashmanager")`
- `application.yml` completo (DB, JWT, CORS, Mail, Flyway com todos os locations)

**Flyway locations em application.yml:**
```yaml
spring:
  flyway:
    locations:
      - classpath:db/migration/core
      - classpath:db/migration/clients
      - classpath:db/migration/services
      - classpath:db/migration/appointments
      - classpath:db/migration/finance
      - classpath:db/migration/stock
      - classpath:db/migration/fichas
```

**Feito quando:**
- [ ] `mvn package -pl lash-app -am -DskipTests` gera JAR em `lash-app/target/`
- [ ] `java -jar lash-app/target/lash-app-*.jar` sobe sem erros (com banco rodando)

---

### T11: Migrations — redistribuir V1–V7

**O que**: Dividir as migrations monolíticas existentes em migrations por módulo com numeração correta
**Onde**: Cada `src/main/resources/db/migration/<módulo>/`
**Depende de**: T02–T09

**Mapeamento:**

| Tabelas no SQL atual | Novo arquivo | Módulo |
|---|---|---|
| `users` | `V100__users_schema.sql` | lash-core |
| `clients` | `V200__clients_schema.sql` | lash-clients |
| `services` | `V300__services_schema.sql` | lash-services |
| `appointments` | `V400__appointments_schema.sql` | lash-appointments |
| `financial_entries` | `V500__financial_entries_schema.sql` | lash-finance |
| `inventory_items`, `inventory_movements` | `V600__inventory_schema.sql` | lash-stock |
| `anamneses`, `anamnese_tokens`, `lash_mappings` | `V700__fichas_schema.sql` | lash-fichas |
| Seed admin (V2) | `V101__seed_admin.sql` | lash-core |
| Seed clients/services (V2) | `V201__seed_clients.sql`, `V301__seed_services.sql` | lash-clients, lash-services |
| V3 (financial_entry_id em appointments) | `V401__appointments_financial_entry.sql` | lash-appointments |
| V4 (client_id nullable) | `V402__appointments_client_nullable.sql` | lash-appointments |
| V5 (finance new fields) | `V501__finance_new_fields.sql` | lash-finance |

> ⚠️ Após essa task: banco de dev precisa ser **recriado** (`docker stop lashmanager-db && docker rm lashmanager-db` + novo container) porque o Flyway reconhece as migrations pelo checksum — renomear invalida o histórico.

**Feito quando:**
- [ ] Cada módulo tem seus arquivos SQL em `db/migration/<módulo>/`
- [ ] Os SQL originais V1–V7 em `src/main/resources/db/migration/` são removidos (ou ficam apenas no monólito legado)
- [ ] Nenhuma tabela duplicada entre os arquivos
- [ ] `mvn package -DskipTests` com banco recriado não lança erros Flyway

---

### T12: GlobalExceptionHandler — mover para lash-app

**O que**: Mover o `GlobalExceptionHandler` para `lash-app` onde tem acesso a todas as exceções de todos os módulos
**Onde**: `lash-backend/lash-app/src/main/java/com/lashmanager/app/GlobalExceptionHandler.java`
**Depende de**: T10

**Feito quando:**
- [ ] `GlobalExceptionHandler` em `lash-app`, importando exceções de todos os módulos
- [ ] Todos os `@ExceptionHandler` presentes e com o import correto do novo pacote
- [ ] `mvn compile -pl lash-app -am` sem erros

---

### T13: Testes de Clientes — migrar para lash-clients

**O que**: Mover os 8 arquivos de teste do módulo de clientes para `lash-clients/src/test/`
**Onde**: `lash-backend/lash-clients/src/test/`
**Depende de**: T03

**Arquivos a mover:**
- `AbstractIntegrationTest.java` → já em lash-core (T02); os testes de clientes herdam de lá
- `CreateClientUseCaseImplTest.java` → `lash-clients/src/test/`
- `UpdateClientUseCaseImplTest.java` → `lash-clients/src/test/`
- `DeleteClientUseCaseImplTest.java` → `lash-clients/src/test/`
- `DeactivateClientUseCaseImplTest.java` → `lash-clients/src/test/`
- `GetClientUseCaseImplTest.java` → `lash-clients/src/test/`
- `ListClientsUseCaseImplTest.java` → `lash-clients/src/test/`
- `ClientITest.java` → `lash-clients/src/test/`

**pom.xml de lash-clients**: adicionar `lash-core` como `test` scope para `AbstractIntegrationTest`

**Feito quando:**
- [ ] `mvn test -pl lash-clients` — todos os 28 testes passam
- [ ] Nenhum arquivo de teste restante no monólito antigo

---

### T14: Verificação Final + Limpeza

**O que**: Verificar que o build completo funciona e remover arquivos do monólito original
**Depende de**: T01–T13

**Feito quando:**
- [ ] `mvn package -DskipTests` na raiz (`lash-backend/`) sem erros
- [ ] `mvn test -pl lash-clients` — 28 testes passando
- [ ] `lash-app/target/lash-app-*.jar` executável sobe com banco recriado
- [ ] `src/main/java/com/lashmanager/app/` no monólito original está vazio (todos os arquivos movidos)
- [ ] Nenhuma referência a `com.lashmanager.app` nos módulos filhos (só em lash-app)
- [ ] CLAUDE.md atualizado com os novos comandos por módulo

---

## Resumo de Esforço

| Task | Complexidade | Tipo |
|---|---|---|
| T01: Parent POM | Baixa | Configuração |
| T02: lash-core | Alta | Mover + refatorar pacotes |
| T03: lash-clients | Média | Mover + refatorar pacotes |
| T04: lash-services | Média | Mover + refatorar pacotes |
| T05: lash-appointments | Média | Mover + refatorar pacotes (cross-module deps) |
| T06: lash-finance | Média | Mover + refatorar pacotes |
| T07: lash-stock | Média | Mover + refatorar pacotes |
| T08: lash-fichas | Média | Mover + refatorar pacotes |
| T09: lash-dashboard | Baixa | Mover + refatorar pacotes |
| T10: lash-app | Baixa | Novo módulo boot |
| T11: Migrations | Alta | Dividir SQL + recriar banco |
| T12: ExceptionHandler | Baixa | Mover + ajustar imports |
| T13: Testes | Baixa | Mover + ajustar imports |
| T14: Verificação | Baixa | Validação |
