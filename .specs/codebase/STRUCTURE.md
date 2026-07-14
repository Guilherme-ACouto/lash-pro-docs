# Estrutura do Projeto — Lash Manager

**Atualizado:** 2026-07-14

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
│           └── servicos/
└── CLAUDE.md              ← Instruções para o Claude Code
```

---

## Backend — `lash-backend/` (multi-módulo Maven)

```
lash-backend/
├── pom.xml                       ← POM pai (packaging=pom), declara os 9 <modules> e <dependencyManagement>
│
├── lash-core/                    ← Auth, User, exceções raiz, security, email
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/core/
│       │   ├── domain/
│       │   │   ├── model/User.java
│       │   │   ├── exception/DomainException.java, BusinessException.java, InvalidCredentialsException.java, TokenExpiredException.java, UserNotFoundException.java
│       │   │   └── port/in/ (LoginUseCase, ...) · port/out/ (UserRepository, TokenPort, ...)
│       │   ├── application/usecase/ (LoginUseCaseImpl, ...)
│       │   ├── infrastructure/
│       │   │   ├── persistence/entity|mapper|repository/ (UserEntity, ...)
│       │   │   ├── security/ (JwtService, SecurityConfig, JwtAuthFilter, UserDetailsServiceImpl)
│       │   │   └── email/
│       │   └── adapter/web/controller/AuthController.java · dto/
│       ├── main/resources/db/migration/core/V100__create_users_table.sql
│       └── test/java/com/lashmanager/core/AbstractIntegrationTest.java
│
├── lash-clients/                 ← depende de lash-core
│   └── src/main/java/com/lashmanager/clients/{domain,application,infrastructure,adapter}/...
│       resources/db/migration/clients/V200__create_clients_table.sql
│       test/java/com/lashmanager/clients/
│           ├── AbstractIntegrationTest.java · ClientsTestApplication.java
│           └── application/usecase/ (CreateClientUseCaseImplTest, ..., ClientITest)
│
├── lash-services/                ← depende de lash-core
│   └── src/main/java/com/lashmanager/services/... · resources/db/migration/V300__create_services_table.sql
│
├── lash-appointments/            ← depende de lash-core, lash-clients, lash-services
│   └── src/main/java/com/lashmanager/appointments/...
│       ├── domain/model/Appointment.java, AppointmentStatus.java
│       └── resources/db/migration/V400__create_appointments_table.sql
│
├── lash-finance/                 ← depende de lash-core, lash-appointments
│   └── src/main/java/com/lashmanager/finance/...
│       ├── domain/model/FinancialEntry.java, MonthlyFinancialStat.java, FinancialEntryType.java, ...
│       └── resources/db/migration/V500__create_financial_entries_table.sql
│
├── lash-stock/                   ← depende de lash-core, lash-finance
│   └── src/main/java/com/lashmanager/stock/...
│       ├── domain/model/InventoryItem.java, InventoryMovement.java, MovementType.java, ...
│       └── resources/db/migration/V600__create_inventory_tables.sql
│
├── lash-fichas/                  ← depende de lash-core, lash-clients
│   └── src/main/java/com/lashmanager/fichas/...
│       ├── domain/model/Ficha.java, LashMapping.java
│       └── resources/db/migration/V700__create_fichas_tables.sql
│
├── lash-dashboard/                ← depende de lash-core, lash-clients, lash-appointments, lash-finance, lash-stock
│   └── src/main/java/com/lashmanager/dashboard/
│       ├── application/usecase/ (2 use cases de agregação)
│       └── adapter/web/controller/DashboardController.java
│
└── lash-app/                     ← módulo executável, depende de todos os outros 8
    └── src/main/
        ├── java/com/lashmanager/app/
        │   ├── LashManagerApplication.java   ← @SpringBootApplication(scanBasePackages="com.lashmanager")
        │   │                                    + @EnableJpaRepositories + @EntityScan (mesmo basePackages)
        │   └── adapter/web/GlobalExceptionHandler.java  ← único @RestControllerAdvice do sistema
        └── resources/
            ├── application.yml               ← única config de runtime do sistema
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
│       └── out/           ← *Repository, portas de integração cross-módulo (ex.: ClientAppointmentPort)
├── application/usecase/   ← *UseCaseImpl + *UseCaseMapper
├── infrastructure/
│   ├── persistence/entity|mapper|repository/
│   ├── adapter/           ← implementações de port/out que cruzam módulo (quando este módulo é o "dono" do dado)
│   └── config/            ← @Configuration específica (quando existe)
└── adapter/web/
    ├── controller/
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

---

## Banco de dados — `db/migration/` (uma migration por módulo)

| Módulo | Arquivo | Conteúdo |
|---|---|---|
| lash-core | `V100__create_users_table.sql` | Tabela `users` |
| lash-clients | `V200__create_clients_table.sql` | Tabela `clients` |
| lash-services | `V300__create_services_table.sql` | Tabela `services` |
| lash-appointments | `V400__create_appointments_table.sql` | Tabela `appointments` (FK clients, services) |
| lash-finance | `V500__create_financial_entries_table.sql` | Tabela `financial_entries` (FK appointments) |
| lash-stock | `V600__create_inventory_tables.sql` | Tabelas `inventory_items`, `inventory_movements` |
| lash-fichas | `V700__create_fichas_tables.sql` | Tabelas `fichas`, `lash_mappings` (FK clients) |
| lash-app | `V800__seed_data.sql` | 1 usuário admin + serviços padrão |

Ranges reservados por módulo (V100–V800), com espaço para migrations incrementais futuras dentro de cada faixa (ex.: `V101__...` para uma alteração futura em `lash-core`). Flyway varre `classpath:db/migration` em todos os módulos do classpath do `lash-app` — a ordem de aplicação é pela numeração `V{n}`, não pela ordem dos módulos no POM.

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
    ├── clientes/
    │   ├── spec.md        ← CLI-01 a CLI-13
    │   ├── design.md
    │   └── tasks.md       ← 12/12 concluídas
    └── servicos/
        ├── spec.md        ← SVC-01 a SVC-13
        ├── design.md
        └── tasks.md       ← 12/12 concluídas
```
