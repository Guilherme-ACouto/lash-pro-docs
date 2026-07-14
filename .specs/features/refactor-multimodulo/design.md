# Refatoração Multi-Módulo Maven — Design

---

## Estrutura de Diretórios Alvo

```
lash-backend/                          ← parent POM (packaging=pom)
├── pom.xml
├── lash-core/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/core/
│       │   ├── domain/
│       │   │   ├── model/             (User, UserRole)
│       │   │   ├── exception/         (BusinessException, DomainException, InvalidCredentialsException, TokenExpiredException, UserNotFoundException)
│       │   │   └── port/
│       │   │       ├── in/            (LoginUseCase, RefreshTokenUseCase, ForgotPasswordUseCase)
│       │   │       └── out/           (UserRepository, TokenPort, EmailPort)
│       │   ├── application/usecase/   (LoginUseCaseImpl, RefreshTokenUseCaseImpl, ForgotPasswordUseCaseImpl)
│       │   └── infrastructure/
│       │       ├── email/             (EmailPortImpl)
│       │       ├── persistence/       (UserEntity, UserJpaRepository, UserMapper, UserRepositoryImpl)
│       │       └── security/          (JwtService, JwtAuthFilter, SecurityConfig, UserDetailsServiceImpl)
│       ├── main/resources/
│       │   └── db/migration/core/     (V100__create_users_schema.sql, V101__seed_admin.sql)
│       └── test/java/com/lashmanager/core/
│           ├── AbstractIntegrationTest.java
│           └── resources/application-test.yml
│
├── lash-clients/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/clients/
│       │   ├── domain/model/          (Client)
│       │   ├── domain/exception/      (ClientNotFoundException, ClientAlreadyExistsException, HasFutureAppointmentsException, AppointmentSummary)
│       │   ├── domain/port/in/        (CreateClientUseCase, UpdateClientUseCase, ...)
│       │   ├── domain/port/out/       (ClientRepository)
│       │   ├── application/usecase/   (*UseCaseImpl)
│       │   ├── infrastructure/        (ClientEntity, ClientJpaRepository, ClientMapper, ClientRepositoryImpl)
│       │   └── adapter/web/           (ClientController, ClientRequest/Response DTOs)
│       ├── main/resources/db/migration/clients/
│       │   └── V200__clients_schema.sql
│       └── test/                      (testes de clientes migrados aqui)
│
├── lash-services/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/services/
│       │   └── (mesma estrutura hexagonal)
│       └── main/resources/db/migration/services/
│           └── V300__services_schema.sql
│
├── lash-appointments/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/appointments/
│       │   └── (mesma estrutura hexagonal)
│       └── main/resources/db/migration/appointments/
│           └── V400__appointments_schema.sql
│           └── V401__appointments_client_nullable.sql
│           └── V402__appointments_financial_entry.sql
│
├── lash-finance/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/finance/
│       │   └── (mesma estrutura hexagonal)
│       └── main/resources/db/migration/finance/
│           └── V500__finance_schema.sql
│           └── V501__finance_new_fields.sql
│
├── lash-stock/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/stock/
│       │   └── (mesma estrutura hexagonal)
│       └── main/resources/db/migration/stock/
│           └── V600__stock_schema.sql
│
├── lash-fichas/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/lashmanager/fichas/
│       │   └── (mesma estrutura hexagonal)
│       └── main/resources/db/migration/fichas/
│           └── V700__fichas_schema.sql
│
├── lash-dashboard/
│   ├── pom.xml
│   └── src/
│       └── main/java/com/lashmanager/dashboard/
│           └── (domain + use cases + infra + controller)
│
└── lash-app/
    ├── pom.xml
    └── src/
        └── main/
            ├── java/com/lashmanager/app/
            │   └── LashManagerApplication.java  (@SpringBootApplication)
            └── resources/
                └── application.yml
```

---

## Grafo de Dependências Entre Módulos

```
lash-core
   ↑
lash-clients ──────────────────────────────────┐
lash-services ─────────────────────────────────┤
lash-appointments ← lash-clients, lash-services │
lash-finance ← lash-appointments               │
lash-stock ← lash-finance                      │
lash-fichas ← lash-clients                     │
lash-dashboard ← (todos os módulos acima)      │
lash-app ← (todos os módulos acima) ───────────┘
```

> `lash-appointments` depende de `lash-clients` e `lash-services` porque `AppointmentEntity` tem `@ManyToOne` read-only para `ClientEntity` e `ServiceEntity` (necessário para joins JPA).

---

## Parent POM — Decisões

- `packaging=pom` no parent
- `spring-boot-starter-parent` como parent do parent POM
- Versões de todas as dependências gerenciadas no `<dependencyManagement>` do parent
- `spring-boot-maven-plugin` com `skip=true` em todos os módulos exceto `lash-app`
- `lash-app` usa `<repackage>` para gerar o JAR executável

---

## lash-core — Conteúdo Detalhado

| Origem atual | Destino |
|---|---|
| `domain/model/User.java`, `UserRole.java` | `lash-core` domain/model |
| `domain/exception/BusinessException.java`, `DomainException.java`, `InvalidCredentialsException.java`, `TokenExpiredException.java`, `UserNotFoundException.java` | `lash-core` domain/exception |
| `domain/port/in/LoginUseCase.java`, `RefreshTokenUseCase.java`, `ForgotPasswordUseCase.java` | `lash-core` domain/port/in |
| `domain/port/out/UserRepository.java`, `TokenPort.java`, `EmailPort.java` | `lash-core` domain/port/out |
| `application/usecase/LoginUseCaseImpl.java`, `RefreshTokenUseCaseImpl.java`, `ForgotPasswordUseCaseImpl.java` | `lash-core` application/usecase |
| `infrastructure/security/` (todos) | `lash-core` infrastructure/security |
| `infrastructure/email/EmailPortImpl.java` | `lash-core` infrastructure/email |
| `infrastructure/persistence/entity/UserEntity.java` | `lash-core` infrastructure/persistence |
| `infrastructure/persistence/repository/UserJpaRepository.java`, `UserRepositoryImpl.java` | `lash-core` infrastructure/persistence |
| `infrastructure/persistence/mapper/UserMapper.java` | `lash-core` infrastructure/persistence |
| `adapter/web/controller/AuthController.java` | `lash-core` adapter/web |
| `adapter/web/dto/LoginRequest.java`, `LoginResponse.java`, `RefreshTokenRequest.java`, `ForgotPasswordRequest.java`, `ErrorResponse.java` | `lash-core` adapter/web/dto |
| `src/test/java/.../AbstractIntegrationTest.java` | `lash-core` src/test |
| `src/test/resources/application-test.yml` | `lash-core` src/test/resources |

---

## Estratégia de Migrations

As migrations atuais V1–V7 são redistribuídas:

| Migration atual | Novo nome | Módulo dono |
|---|---|---|
| V1 (users, schema completo) | → V100 (só tabela `users`) | lash-core |
| V1 (clients) | → V200 | lash-clients |
| V1 (services) | → V300 | lash-services |
| V1 (appointments) | → V400 | lash-appointments |
| V2 (seed admin) | → V101 | lash-core |
| V2 (seed clients/services) | → V201/V301 | lash-clients / lash-services |
| V3 (financial_entry_id em appointments) | → V401 | lash-appointments |
| V4 (client_id nullable) | → V402 | lash-appointments |
| V5 (finance new fields) | → V500 + V501 | lash-finance |
| V6 (inventory schema) | → V600 | lash-stock |
| V7 (fichas schema) | → V700 | lash-fichas |

**Configuração Flyway em `lash-app/application.yml`:**
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
      - classpath:db/migration/dashboard
```

> ⚠️ **Atenção**: o banco de desenvolvimento já tem as migrations V1–V7 aplicadas. Será necessário fazer um `flyway repair` ou recriar o banco após a refatoração (em dev isso é seguro).

---

## GlobalExceptionHandler

O `GlobalExceptionHandler` atual trata exceções de todos os módulos. Duas opções:

**Opção A (escolhida)**: manter um único `GlobalExceptionHandler` em `lash-app` que importa e trata todas as exceções de todos os módulos. Simples e centralizado.

**Opção B**: cada módulo tem seu próprio `@ControllerAdvice` — mais isolado mas requer cuidado com precedência.

→ Seguir opção A por simplicidade.

---

## Componentes Compartilhados Entre Módulos

- `HasFutureAppointmentsException` e `AppointmentSummary` — ficam em `lash-clients` pois são lançadas pelo use case de clientes (que consultam agendamentos via UUID, sem import de lash-appointments)
- `AppointmentEntity` com `@ManyToOne ClientEntity` — `lash-appointments` depende de `lash-clients`
- `FinancialEntryLinkedToAppointmentException` — fica em `lash-finance`
- `MonthlyFinancialStat` — fica em `lash-finance`
