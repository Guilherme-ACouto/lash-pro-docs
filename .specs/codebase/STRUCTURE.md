# Estrutura do Projeto — Lash Manager

**Atualizado:** 2026-07-15

## Raiz do monorepo

```
/Users/Apple/Documents/Lash/
├── lash-backend/          ← Java 21 + Spring Boot 3, multi-módulo Maven (9 módulos)
├── lash-frontend/         ← Angular 18
├── lash-docs/             ← Documentação e specs
│   └── .specs/
│       ├── project/       ← PROJECT.md, ROADMAP.md, STATE.md
│       ├── codebase/      ← Este conjunto de 7 docs
│       └── features/      ← Spec/Design/Tasks por feature
│           ├── clientes/
│           ├── servicos/
│           ├── refactor-multimodulo/
│           └── refactor-backend/    ← multi-tenancy + Command/ApplicationService/Query
└── CLAUDE.md              ← Instruções para o Claude Code
```

---

## Backend — `lash-backend/` (multi-módulo Maven)

```
lash-backend/
├── pom.xml                       ← POM pai (packaging=pom), declara os 9 <modules> e <dependencyManagement>
│
├── lash-core/                    ← Auth, User, Tenant, multi-tenancy, Command/ApplicationService/
│   │                                Interceptor, exceções raiz, security, email
│   ├── pom.xml                   ← + spring-boot-starter-aop, liquibase-core, flyway-core (test)
│   └── src/
│       ├── main/java/com/lashmanager/core/
│       │   ├── domain/
│       │   │   ├── model/User.java, Tenant.java, CommandAuditLog.java
│       │   │   ├── exception/DomainException, BusinessException, InvalidCredentialsException,
│       │   │   │   TokenExpiredException, UserNotFoundException, EmailAlreadyInUseException,
│       │   │   │   ActivationKeyInvalidException, ActivationKeyExpiredException,
│       │   │   │   AccountNotActivatedException, TenantInactiveException,
│       │   │   │   PlatformAdminRequiredException, TenantNotFoundException,
│       │   │   │   SchemaProvisioningException
│       │   │   └── port/
│       │   │       ├── in/ (LoginUseCase, RefreshTokenUseCase, ForgotPasswordUseCase,
│       │   │       │         RegisterUseCase, ResendActivationUseCase, ActivateAccountUseCase,
│       │   │       │         ListTenantsUseCase, DeactivateTenantUseCase)
│       │   │       └── out/ (UserRepository, TokenPort, EmailPort, TenantRepository,
│       │   │                 CommandAuditLogRepository, SchemaProvisionerPort)
│       │   ├── application/
│       │   │   ├── command/ (RegisterCommand, ResendActivationCommand, ActivateAccountCommand)
│       │   │   ├── service/ (RegisterApplicationService, ResendActivationApplicationService,
│       │   │   │              ActivateAccountApplicationService)
│       │   │   └── usecase/ (LoginUseCaseImpl, RegisterUseCaseImpl, ActivateAccountUseCaseImpl,
│       │   │                  ListTenantsUseCaseImpl, DeactivateTenantUseCaseImpl,
│       │   │                  PlatformAdminChecker, ...)
│       │   ├── infrastructure/
│       │   │   ├── command/ (AbstractCommand, CommandInterceptor)
│       │   │   ├── multitenancy/ (TenantContext, CurrentTenantIdentifierResolverImpl,
│       │   │   │                   MultiTenantConnectionProviderImpl, TenantSchemaNaming,
│       │   │   │                   SchemaProvisionerImpl, HibernateMultiTenantConfig)
│       │   │   ├── persistence/entity|mapper|repository/ (UserEntity, TenantEntity,
│       │   │   │     CommandAuditLogEntity, ...)
│       │   │   ├── security/ (JwtService, SecurityConfig, JwtAuthFilter,
│       │   │   │               TenantResolvingFilter, UserDetailsServiceImpl)
│       │   │   └── email/ (EmailPortImpl — sendPasswordResetEmail + sendActivationEmail)
│       │   └── adapter/web/controller/ (AuthController, RegisterController,
│       │         ActivationController, TenantAdminController) · dto/
│       ├── main/resources/
│       │   ├── db/migration/ (V100 users, V101 tenants, V102 tenant_id+activation em users,
│       │   │                   V103 command_audit_log)
│       │   └── db/changelog/ (master.xml, fragments/command-audit-log.xml)
│       └── test/java/com/lashmanager/core/
│             (AbstractIntegrationTest, CoreTestApplication, RegistrationFlowITest,
│              CommandInterceptorITest)
│
├── lash-clients/                 ← depende de lash-core
│   └── src/main/java/com/lashmanager/clients/
│       ├── domain/port/out/ (ClientRepository — escrita, ClientQueryRepository — leitura)
│       ├── application/command/ (CreateClientCommand, UpdateClientCommand, DeleteClientCommand,
│       │                          DeactivateClientCommand, ReactivateClientCommand)
│       ├── application/service/ (CreateClientApplicationService, UpdateClientApplicationService,
│       │                          DeleteClientApplicationService, DeactivateClientApplicationService)
│       ├── infrastructure/persistence/repository/ (ClientRepositoryImpl — escrita,
│       │                                             ClientQueryRepositoryImpl — leitura)
│       ├── resources/db/migration/V200__create_clients_table.sql
│       ├── resources/db/changelog/clients-changelog.xml  ← Liquibase, reaproveita o V200 via <sqlFile>
│       └── test/java/com/lashmanager/clients/
│             ├── AbstractIntegrationTest.java (schema de tenant fixo, banco lashmanager_test)
│             ├── ClientsTestApplication.java
│             └── application/usecase/ (CreateClientUseCaseImplTest, ..., ClientITest)
│
├── lash-services/                ← depende de lash-core — mesma estrutura de lash-clients
│   └── ...ServiceQueryRepository, Create/Update/Delete/DeactivateServiceCommand+ApplicationService...
│       resources/db/migration/V300__create_services_table.sql
│       resources/db/changelog/services-changelog.xml
│
├── lash-appointments/            ← depende de lash-core, lash-clients, lash-services
│   └── src/main/java/com/lashmanager/appointments/
│       ├── domain/port/out/ (AppointmentRepository — escrita, AppointmentQueryRepository — leitura)
│       ├── application/command/ (CreateAppointmentCommand, UpdateAppointmentCommand,
│       │     ConfirmAppointmentCommand, CompleteAppointmentCommand, CancelAppointmentCommand,
│       │     NoShowAppointmentCommand)
│       ├── application/service/ (CreateAppointmentApplicationService,
│       │     UpdateAppointmentApplicationService, ChangeAppointmentStatusApplicationService
│       │     — 1 serviço com 4 when() sobrecarregados pros comandos de status)
│       ├── resources/db/migration/V400__create_appointments_table.sql
│       └── resources/db/changelog/appointments-changelog.xml
│
├── lash-finance/                 ← depende de lash-core, lash-appointments
│   └── src/main/java/com/lashmanager/finance/
│       ├── domain/port/out/ (FinancialEntryRepository — escrita,
│       │     FinancialEntryQueryRepository — leitura, FinancialSummaryRepository — agregação)
│       ├── application/command/ (Create/Update/DeleteFinancialEntryCommand,
│       │     ToggleFinancialEntryPaidCommand)
│       ├── infrastructure/persistence/repository/FinancialSummaryRepositoryImpl.java
│       │     ← referência do padrão de leitura por agregação via EntityManager (ver CONVENTIONS.md)
│       ├── resources/db/migration/V500__create_financial_entries_table.sql
│       └── resources/db/changelog/finance-changelog.xml
│
├── lash-stock/                   ← depende de lash-core, lash-finance
│   └── src/main/java/com/lashmanager/stock/
│       ├── domain/port/out/ (InventoryItemRepository/QueryRepository,
│       │     InventoryMovementRepository/QueryRepository)
│       ├── application/command/ (Create/Update/DeleteInventoryItemCommand,
│       │     SetInventoryItemActiveCommand, RegisterPurchaseCommand, RegisterManualExitCommand)
│       ├── resources/db/migration/V600__create_inventory_tables.sql
│       └── resources/db/changelog/stock-changelog.xml
│
├── lash-fichas/                  ← depende de lash-core, lash-clients
│   └── src/main/java/com/lashmanager/fichas/
│       ├── domain/port/out/ (FichaRepository/QueryRepository, LashMappingRepository/QueryRepository)
│       ├── application/command/ (Create/UpdateFichaCommand, Create/Update/DeleteLashMappingCommand)
│       ├── application/service/ (..., LashMappingApplicationService — quando()s de
│       │     Create/Update/DeleteLashMapping compartilhados)
│       ├── resources/db/migration/V700__create_fichas_tables.sql
│       └── resources/db/changelog/fichas-changelog.xml
│
├── lash-dashboard/                ← depende de lash-core, lash-clients, lash-appointments, lash-finance, lash-stock
│   └── src/main/java/com/lashmanager/dashboard/
│       ├── application/usecase/ (2 use cases de agregação — 100% leitura, sem Command/ApplicationService)
│       └── adapter/web/controller/DashboardController.java
│
└── lash-app/                     ← módulo executável, depende de todos os outros 8
    └── src/main/
        ├── java/com/lashmanager/app/
        │   ├── LashManagerApplication.java   ← @SpringBootApplication(scanBasePackages="com.lashmanager")
        │   │                                    + @EnableJpaRepositories + @EntityScan (mesmo basePackages)
        │   └── adapter/web/GlobalExceptionHandler.java  ← único @RestControllerAdvice do sistema
        └── resources/
            ├── application.yml               ← única config de runtime — + spring.flyway.out-of-order,
            │                                    spring.liquibase.enabled=false, app.tenant.*, app.activation.*
            └── db/migration/V800__seed_data.sql
```

### Padrão interno de cada módulo de domínio

```
{modulo}/src/main/java/com/lashmanager/{modulo}/
├── domain/
│   ├── model/            ← entidades puras
│   ├── exception/         ← estende DomainException/BusinessException de lash-core
│   └── port/
│       ├── in/            ← *UseCase
│       └── out/           ← *Repository (escrita), *QueryRepository (leitura),
│                             portas de integração cross-módulo (ex.: ClientAppointmentPort)
├── application/
│   ├── command/           ← *Command (escrita, estende AbstractCommand de lash-core)
│   ├── service/            ← *ApplicationService (when(Command), interceptado por CommandInterceptor)
│   └── usecase/            ← *UseCaseImpl (anotado @Service) + *UseCaseMapper
├── infrastructure/
│   ├── persistence/entity|mapper|repository/  (*RepositoryImpl + *QueryRepositoryImpl)
│   └── adapter/           ← implementações de port/out que cruzam módulo (quando este módulo é o "dono" do dado)
└── adapter/web/
    ├── controller/         ← injeta *ApplicationService (escrita) + *UseCase (leitura)
    └── dto/
```

---

## Frontend — `lash-frontend/`

```
lash-frontend/
├── package.json
├── angular.json
├── tsconfig.json
├── proxy.conf.json               ← /api → http://localhost:8080
└── src/
    ├── main.ts
    ├── styles.scss                ← tema Angular Material M3
    └── app/
        ├── app.config.ts          ← provideStore, provideEffects, provideRouter
        ├── app.routes.ts          ← rotas lazy-loaded por feature
        ├── core/
        │   ├── auth/
        │   │   ├── auth.guard.ts
        │   │   ├── auth.service.ts
        │   │   └── jwt.interceptor.ts
        │   ├── http/
        │   │   ├── error.interceptor.ts
        │   │   ├── loading.interceptor.ts
        │   │   └── loading.service.ts
        │   └── models/
        │       ├── api.model.ts     ← PageResponse<T>, ApiError
        │       ├── auth.model.ts
        │       ├── client.model.ts  ← Client, CreateClientRequest, ClientState
        │       └── service.model.ts ← Service, CreateServiceRequest, ServiceState
        ├── features/
        │   ├── auth/
        │   │   ├── components/login/login.component.ts
        │   │   └── store/ (auth.actions/reducer/effects/selectors)
        │   ├── clients/
        │   │   ├── clients.routes.ts
        │   │   ├── services/client.service.ts
        │   │   ├── store/ (client.actions/reducer/effects/selectors)
        │   │   └── components/
        │   │       ├── client-list/
        │   │       ├── client-form/
        │   │       └── client-detail/
        │   ├── services/
        │   │   ├── services.routes.ts
        │   │   ├── services/service.service.ts
        │   │   ├── store/ (service.actions/reducer/effects/selectors)
        │   │   └── components/
        │   │       ├── service-list/
        │   │       ├── service-form/
        │   │       └── service-detail/
        │   ├── appointments/       ← placeholder (shell component apenas) — backend já implementado
        │   ├── financial/          ← placeholder — backend já implementado
        │   ├── inventory/          ← placeholder — backend já implementado
        │   └── dashboard/          ← placeholder — backend já implementado
        ├── layout/
        │   ├── shell/shell.component.ts
        │   ├── sidebar/sidebar.component.ts     ← desktop
        │   └── bottom-nav/bottom-nav.component.ts ← mobile
        └── shared/
            └── pipes/
                ├── currency-br.pipe.ts
                ├── date-ptbr.pipe.ts
                └── duration.pipe.ts
```

> Ainda não existe UI para registro/ativação de conta nem para o endpoint admin de tenants — só o backend está pronto.

---

## Banco de dados — schema `public` vs schema de tenant

**`public`** (Flyway, `{modulo}/src/main/resources/db/migration/`):

| Módulo | Arquivo | Conteúdo |
|---|---|---|
| lash-core | `V100__create_users_table.sql` | Tabela `users` |
| lash-core | `V101__create_tenants_table.sql` | Tabela `tenants` |
| lash-core | `V102__add_tenant_and_activation_to_users.sql` | `tenant_id`, `activation_key`, `activation_key_expiry` em `users` |
| lash-core | `V103__create_command_audit_log_table.sql` | Tabela `command_audit_log` |
| lash-app | `V800__seed_data.sql` | 1 usuário admin (sem tenant) + serviços padrão |

**Schema de cada tenant** (Liquibase, `{modulo}/src/main/resources/db/changelog/`, aplicado por `SchemaProvisionerImpl` via `lash-core/db/changelog/master.xml`):

| Módulo | Arquivo | Conteúdo |
|---|---|---|
| lash-clients | `clients-changelog.xml` | Tabela `clients` (reaproveita `V200` via `<sqlFile>`) |
| lash-services | `services-changelog.xml` | Tabela `services` (reaproveita `V300`) |
| lash-appointments | `appointments-changelog.xml` | Tabela `appointments` (reaproveita `V400`) |
| lash-finance | `finance-changelog.xml` | Tabela `financial_entries` (reaproveita `V500`) |
| lash-stock | `stock-changelog.xml` | Tabelas `inventory_items`/`inventory_movements` (reaproveita `V600`) |
| lash-fichas | `fichas-changelog.xml` | Tabelas `fichas`/`lash_mappings` (reaproveita `V700`) |
| lash-core | `fragments/command-audit-log.xml` | Tabela `command_audit_log` (changeset nativo, não reaproveita SQL) |

Os `<include>` de módulos de negócio no `master.xml` usam `errorIfMissing="false"` — necessário porque um módulo rodando sua própria suíte de teste isolada não tem os changelogs dos outros módulos no classpath.

Ranges de versão Flyway (`V100`–`V800`) continuam reservados por módulo, com espaço para migrations incrementais futuras dentro de cada faixa.

---

## Specs — `lash-docs/.specs/`

```
.specs/
├── project/
│   ├── PROJECT.md
│   ├── ROADMAP.md
│   └── STATE.md
├── codebase/              ← este conjunto de 7 docs
└── features/
    ├── clientes/          ← spec/design/tasks — CLI-01 a CLI-13, concluído
    ├── servicos/           ← spec/design/tasks — SVC-01 a SVC-13, concluído
    ├── refactor-multimodulo/  ← migração pra multi-módulo Maven, concluído
    └── refactor-backend/      ← multi-tenancy + Command/ApplicationService/Query
        ├── spec.md         ← RBK-01 a RBK-27
        ├── design.md
        └── tasks.md        ← 36 tasks + TD01, todas as fases (A-F) concluídas
```
