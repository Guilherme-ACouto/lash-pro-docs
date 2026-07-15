# Arquitetura — Lash Manager

**Atualizado:** 2026-07-15

## Visão Geral

Monorepo com dois projetos independentes: `lash-backend/` (Java, **multi-módulo Maven**) e `lash-frontend/` (Angular). O backend segue **Arquitetura Hexagonal (Ports & Adapters)** dentro de cada módulo, agora com duas camadas adicionais por cima do hexágono clássico: **multi-tenancy** (isolamento por schema Postgres) e **Command/ApplicationService** (padrão de escrita inspirado na Pontta, piloto em `lash-core` e propagado para os 6 módulos de negócio). O frontend segue **Feature-first com NgRx**.

---

## Backend — Multi-tenancy (schema-per-tenant)

Introduzida no projeto `refactor-backend` (ver `lash-docs/.specs/features/refactor-backend/`). O sistema deixou de assumir um único negócio (salão) e passou a suportar múltiplos tenants (contas de clientes SaaS) isolados por **schema Postgres**, não por coluna `tenant_id` nas tabelas de negócio.

### Como o isolamento funciona

```
Schema "public" (compartilhado, sempre existe)
├── users             ← tenant_id (nullable — null = usuário de plataforma/dev)
├── tenants            ← registro central de cada tenant (id, name, schema_name, active)
└── command_audit_log  ← auditoria de commands executados antes de qualquer tenant existir
                          (registro/ativação) — migration Flyway (V103)

Schema "tenant_<uuid-sem-hifen>" (um por tenant ativo, criado sob demanda)
├── clients, services, appointments, financial_entries,
│   inventory_items/movements, fichas/lash_mappings   ← mesmas tabelas de negócio de sempre
└── command_audit_log  ← auditoria de commands de negócio (Fase E) — changeset Liquibase
```

- **Hibernate multi-tenant `SCHEMA` strategy**: `CurrentTenantIdentifierResolverImpl` resolve qual schema usar (default `"public"` quando não há tenant no contexto); `MultiTenantConnectionProviderImpl` executa `SET search_path TO "<schema>", public` em cada conexão obtida do pool. O `public` sempre entra como fallback no `search_path` — por isso `users`/`tenants` (que só existem lá) continuam sendo encontrados mesmo quando o schema do tenant já está resolvido.
- **`TenantContext`** — `ThreadLocal<String>` com o schema atual da requisição. Populado por `TenantResolvingFilter` (lê o claim `tenantId` do JWT, converte pra nome de schema via `TenantSchemaNaming`), limpo no `finally`.
- **Login continua resolvendo só por e-mail**: no momento do login não há tenant no contexto ainda, então a query roda contra `public` (default) — o mesmo mecanismo que permite achar `users` de qualquer schema.
- **Provisionamento de schema** (`SchemaProvisionerImpl`, em `lash-core/infrastructure/multitenancy/`): `CREATE SCHEMA IF NOT EXISTS` + Liquibase `master.xml`, numa conexão JDBC própria (`DriverManager`, fora do pool/transação gerenciados pelo Spring) — assim uma falha no provisionamento não deixa uma transação JPA "presa" numa conexão que já rodou DDL. Idempotente: retry com o mesmo `tenantId` não duplica nada.
- **`master.xml`** (`lash-core/resources/db/changelog/`) agrega um changelog por módulo de negócio (`clients-changelog.xml`, `services-changelog.xml`, ...), cada um reaproveitando a migration Flyway já existente via `<sqlFile>` (sem reescrever SQL) + o fragmento `command-audit-log.xml`. Os `<include>` dos módulos de negócio usam `errorIfMissing="false"` — necessário porque quando um módulo roda sua própria suíte de teste isolada, só os changelogs dos módulos dos quais ele depende estão no classpath.

### Fluxo de registro → ativação → provisionamento

```
POST /api/register
  → RegisterUseCaseImpl: cria User (inativo) + Tenant (inativo, MESMO tenantId) — a linha em
    `tenants` precisa existir já aqui por causa da FK users.tenant_id → tenants.id
  → e-mail de ativação (texto simples via EmailPortImpl, mesmo padrão do "esqueci senha")

GET/POST /api/activation?key=
  → ActivateAccountUseCaseImpl (@Transactional):
      1. valida activationKey (existe? expirou?)
      2. ativa o Tenant existente (active=true) e o User (active=true, limpa activationKey)
      3. schemaProvisionerPort.provision(tenantId) — fora da transação (conexão própria)
      4. se o provisionamento falhar: exceção propaga, passos 2 são revertidos (rollback JPA);
         o schema físico (se chegou a ser criado) não é revertido, mas o retry com a mesma
         key é seguro (idempotente) porque reusa o mesmo tenantId

POST /api/register/resend
  → reenvia o e-mail de ativação pra um cadastro pendente (!active), sem revelar se o
    e-mail existe (resposta genérica)
```

### Endpoint administrativo

`TenantAdminController` (`/api/admin/tenants`) — listar (paginado) e desativar tenants. Restrito a "usuário de plataforma" (`tenantId == null`), não a um `UserRole` novo — `UserRole` (`OWNER`/`ASSISTANT`) é escopado por tenant, então não serve pra distinguir quem administra tenants de *outros* clientes. Desativar um tenant também bloqueia login de todos os usuários dele (`LoginUseCaseImpl` checa `tenant.isActive()`).

---

## Backend — Command + ApplicationService + Query

Segunda camada nova, também do projeto `refactor-backend`, inspirada no padrão real usado na Pontta (levantado via consulta a uma IA com acesso ao codebase de lá). Piloto em `lash-core`, propagado pros 6 módulos de negócio (todos exceto `lash-dashboard`, que é 100% leitura).

```
Controller → Command (DTO com Bean Validation) → ApplicationService.when(command) → UseCase (domínio) → Repository
                                                          ↑
                                                CommandInterceptor (@Aspect)
                                                intercepta qualquer *ApplicationService.when(..)
```

- **`Command`** (`application/command/`) — objeto por operação de escrita, estende `AbstractCommand` (marca `commandId`/`issuedAt`, ignorados no JSON de auditoria). Carrega as mesmas anotações de Bean Validation que os DTOs de Request já tinham.
- **`ApplicationService`** (`application/service/`) — só orquestra: `when(Command)` chama o `UseCase` de domínio, que mantém toda a regra de negócio (nenhuma mudança de responsabilidade). Operações do mesmo agregado podem compartilhar um `ApplicationService` com múltiplos `when(...)` sobrecarregados (ex.: `DeactivateClientApplicationService.when(DeactivateClientCommand)` e `.when(ReactivateClientCommand)`).
- **`CommandInterceptor`** (`lash-core/infrastructure/command/`) — `@Aspect` cujo pointcut casa qualquer `*ApplicationService.when(..)`, em qualquer módulo. Faz três coisas: valida o `Command` (Bean Validation, cobre qualquer chamador — não só HTTP), loga início/fim, e grava uma linha em `command_audit_log` (sucesso ou falha). Ponto de extensão para `@CommandPermission` está preparado mas não implementado (só existe um `UserRole` hoje).
- **Somente escrita passa por Command/ApplicationService** — leitura (`Get`/`List`) continua sendo chamada direto do Controller pelo `UseCase`, sem Command.
- **`QueryRepository`** (`domain/port/out/`) — porta de leitura separada da porta de escrita (`Repository`), reaproveitando o mesmo `JpaRepository`/`Mapper` por baixo (Nível 2: separação lógica, não física de schema). `findById` costuma existir **nas duas portas** quando um use case de escrita precisa carregar o agregado antes de mutar — não é duplicação acidental.
- **Agregações via `EntityManager`/JPQL** (ex.: `FinancialSummaryRepositoryImpl`) continuam com o sufixo `*RepositoryImpl` de sempre — decisão documentada em CONVENTIONS.md de não adotar o sufixo `*Dao` da Pontta, pra não ter duas convenções de adapter de persistência no mesmo código.

Ver CONVENTIONS.md para a tabela completa de nomenclatura desse padrão.

---

## Backend — Multi-módulo Maven + Arquitetura Hexagonal

O backend está dividido em **9 módulos Maven independentes**, um por domínio de negócio (+ um módulo executável). Cada módulo é internamente hexagonal — a divisão em módulos é uma fronteira física adicional por cima da fronteira lógica de camadas.

### Módulos e dependências

```
lash-core            ← base: User/Tenant/Auth, Command/ApplicationService/Interceptor,
                          multi-tenancy, exceções raiz, infra compartilhada (security, email)
   ▲
   ├── lash-clients        ← depende de: lash-core
   ├── lash-services       ← depende de: lash-core
   │      ▲
   │      └── lash-appointments  ← depende de: lash-core, lash-clients, lash-services
   │             ▲
   │             └── lash-finance     ← depende de: lash-core, lash-appointments
   │                    ▲
   │                    └── lash-stock     ← depende de: lash-core, lash-finance
   ├── lash-fichas         ← depende de: lash-core, lash-clients
   └── lash-dashboard      ← depende de: lash-core, lash-clients, lash-appointments,
                               lash-finance, lash-stock (agrega leituras de todos)

lash-app              ← módulo executável (@SpringBootApplication), depende de TODOS os módulos acima
```

**Regra de dependência:** um módulo só pode depender de módulos "abaixo" dele nessa árvore — nunca há dependência circular entre módulos de domínio. `lash-core` não depende de nenhum outro módulo interno.

**Restrição conhecida (C05/C10, ver CONCERNS.md):** `lash-services` não pode depender de `lash-appointments` (inverteria a direção e criaria ciclo) — ainda sem solução aplicada, avaliar módulo `*-api` (padrão visto na Pontta) como possível saída.

**Comunicação entre módulos:** quando um módulo precisa de dado de outro (ex.: `lash-clients` precisa saber se um cliente tem agendamentos futuros antes de desativar), a comunicação é feita via **port/out implementado no módulo consumidor e satisfeito por um adapter no módulo que tem o dado** — nunca acessando repositório/entidade de outro módulo diretamente (exceção pragmática: alguns `QueryRepositoryImpl`/`UseCaseImpl` de leitura, como `AppointmentQueryRepositoryImpl` e `GetDashboardSummaryUseCaseImpl`, leem `JpaRepository`s de outros módulos diretamente pra montar nome/preço de cliente e serviço — aceito porque é leitura pura, sem mutação cross-módulo).

### Camadas e fluxo de uma requisição de escrita (dentro de cada módulo)

```
HTTP Request
    │
    ▼
[Controller]              {modulo}/adapter/web/controller/
    │  constrói Command, chama
    ▼
[ApplicationService]      {modulo}/application/service/    ← intercept. por CommandInterceptor
    │  chama
    ▼
[port/in interface]       {modulo}/domain/port/in/
    │  implementado por
    ▼
[UseCase Impl]            {modulo}/application/usecase/
    │  usa
    ▼
[port/out interface]      {modulo}/domain/port/out/  (Repository — escrita)
    │  implementado por
    ▼
[RepositoryImpl]          {modulo}/infrastructure/persistence/repository/
    │  usa
    ▼
[JpaRepository]           {modulo}/infrastructure/persistence/repository/ (Spring Data)
    │
    ▼
PostgreSQL (schema resolvido pelo TenantContext)
```

Fluxo de **leitura** é mais curto — sem Command/ApplicationService:

```
[Controller] → [UseCase Impl] → [QueryRepository port/out] → [QueryRepositoryImpl] → [JpaRepository] → PostgreSQL
```

### Estrutura de pacotes (repetida em cada módulo, prefixo `com.lashmanager.{modulo}`)

```
com.lashmanager.{modulo}
├── domain/                      ← zero dependência de framework
│   ├── model/                   ← entidades puras (Client, Appointment, FinancialEntry, ...)
│   ├── exception/                ← *Exception estendendo DomainException/BusinessException (de lash-core)
│   └── port/
│       ├── in/                  ← interfaces de use case (CreateClientUseCase, ...)
│       └── out/                 ← *Repository (escrita), *QueryRepository (leitura),
│                                    portas de integração cross-módulo (ex.: ClientAppointmentPort)
│
├── application/
│   ├── command/                 ← *Command (escrita), estende AbstractCommand (lash-core)
│   ├── service/                 ← *ApplicationService — when(Command), intercept. por CommandInterceptor
│   └── usecase/                 ← *UseCaseImpl (regra de negócio) + *UseCaseMapper
│
├── infrastructure/
│   ├── persistence/
│   │   ├── entity/              ← @Entity JPA (ClientEntity, AppointmentEntity, ...)
│   │   ├── mapper/               ← converte Entity ↔ Domain Model
│   │   └── repository/          ← JpaRepository + RepositoryImpl (escrita) + QueryRepositoryImpl (leitura)
│   ├── adapter/                  ← implementações de port/out que cruzam módulo (ex.: ClientAppointmentPortImpl)
│   └── config/                   ← @Configuration específica do módulo, quando existe
│
└── adapter/web/
    ├── controller/               ← @RestController — injeta *ApplicationService (escrita) + *UseCase (leitura)
    └── dto/                      ← records de Request e Response
```

`lash-core` adicionalmente concentra: `Tenant`/`CommandAuditLog` (domain), `AbstractCommand`/`CommandInterceptor` (infra/command), multi-tenancy (`infrastructure/multitenancy/`: `TenantContext`, `CurrentTenantIdentifierResolverImpl`, `MultiTenantConnectionProviderImpl`, `TenantSchemaNaming`, `SchemaProvisionerImpl`, `HibernateMultiTenantConfig`), `DomainException`/`BusinessException` (exceções raiz que todos os módulos estendem), `security/` (JwtService, SecurityConfig, JwtAuthFilter, `TenantResolvingFilter`, UserDetailsServiceImpl) e `infrastructure/email/`.

`lash-app` concentra apenas: classe `LashManagerApplication` (`@SpringBootApplication` com `scanBasePackages`, `@EnableJpaRepositories` e `@EntityScan` em `com.lashmanager`, para varrer todos os módulos) e `GlobalExceptionHandler` (único `@RestControllerAdvice` do sistema — centraliza handlers de exceções de **todos** os módulos, incluindo as novas de multi-tenancy: `EmailAlreadyInUseException`, `ActivationKeyInvalidException`, `ActivationKeyExpiredException`, `AccountNotActivatedException`, `TenantInactiveException`, `PlatformAdminRequiredException`, `TenantNotFoundException`, `SchemaProvisioningException`, `ConstraintViolationException` do `CommandInterceptor`).

### Regras invioláveis

- `domain/` não tem nenhuma dependência de framework (sem `@Entity`, sem `@Service`, sem Spring) — vale em todos os módulos
- Use cases dependem apenas de interfaces (`port/out`), nunca de implementações
- Controllers dependem apenas de `port/in`/`ApplicationService`, nunca de repositórios diretamente
- Injeção sempre via construtor — `@RequiredArgsConstructor`; nunca `@Autowired` em campo
- Módulos nunca acessam entidade JPA ou repositório de outro módulo pra **escrita** — a fronteira é sempre um `port/out` + adapter (leitura pura cross-módulo é aceita pragmaticamente, ver acima)

### Wiring dos use cases: `@Service` direto, sem classe de configuração

Em todos os 9 módulos, `*UseCaseImpl` e `*ApplicationService` são anotados diretamente com `@Service` (`import org.springframework.stereotype.Service;`) e `@RequiredArgsConstructor` — viram bean automaticamente via component scan (`@SpringBootApplication(scanBasePackages = "com.lashmanager")` em `lash-app` cobre todos os módulos). Não existe (e não deve ser criada) nenhuma classe `{Modulo}Config`/`@Bean` para registrar use cases manualmente.

---

## Backend — Módulos Implementados

| Módulo | Domain Model | Endpoint base | Padrão de escrita |
|---|---|---|---|
| lash-core | `User`, `Tenant`, `CommandAuditLog` | `/api/auth/**`, `/api/register/**`, `/api/activation`, `/api/admin/tenants` | Command/ApplicationService (piloto) |
| lash-clients | `Client` | `/api/clients` | Command/ApplicationService |
| lash-services | `ServiceOffering` | `/api/services` | Command/ApplicationService |
| lash-appointments | `Appointment`, `AppointmentStatus` | `/api/appointments` | Command/ApplicationService |
| lash-finance | `FinancialEntry`, `MonthlyFinancialStat` | `/api/financial` | Command/ApplicationService |
| lash-stock | `InventoryItem`, `InventoryMovement` | `/api/inventory` | Command/ApplicationService |
| lash-fichas | `Ficha`, `LashMapping` | `/api/fichas` | Command/ApplicationService |
| lash-dashboard | *(agrega leituras — sem model próprio)* | `/api/dashboard` | N/A — 100% leitura, sem operação de escrita |
| lash-app | — (só bootstrap + GlobalExceptionHandler) | — | — |

Todos os 9 módulos estão implementados no backend (domain → use case → persistência → controller), agora com Command/ApplicationService nas operações de escrita e QueryRepository na leitura (exceto `lash-dashboard`, puramente leitura). A cobertura de testes automatizados, porém, hoje só existe em `lash-core` e `lash-clients` — ver TESTING.md.

---

## Frontend — Feature-first com NgRx

### Ciclo de dados NgRx

```
Component
    │ dispatch(Action)
    ▼
NgRx Effects          ← chama HTTP service
    │ map → Action
    ▼
NgRx Reducer          ← atualiza o state imutavelmente
    │
    ▼
NgRx Store (state)
    │ select(Selector)
    ▼
Component (async pipe no template)
```

### Estrutura de uma feature

```
features/nome/
├── components/
│   ├── nome-list/nome-list.component.ts
│   ├── nome-form/nome-form.component.ts
│   └── nome-detail/nome-detail.component.ts
├── services/nome.service.ts          ← HTTP client
├── store/
│   ├── nome.actions.ts               ← createAction com props<>()
│   ├── nome.reducer.ts               ← createReducer com on()
│   ├── nome.effects.ts               ← createEffect, switchMap, catchError
│   └── nome.selectors.ts             ← createFeatureSelector + createSelector
└── nome.routes.ts                    ← Routes lazy-loaded
```

### Regras Angular

- Todos os componentes são `standalone: true` — sem NgModules
- `inject()` no lugar de injeção por construtor em componentes e serviços
- `async pipe` nos templates — zero `.subscribe()` em componentes
- Reactive Forms em todos os formulários
- Control flow Angular 17+: `@if`, `@for`, `@else` (não `*ngIf`, `*ngFor`)
- Rotas lazy-loaded: `loadChildren: () => import(...)` no `app.routes.ts`

### Features implementadas

| Feature | Rota | Store slice | Componentes |
|---|---|---|---|
| Auth | `/login` | `auth` | LoginComponent |
| Layout | (shell) | — | SidebarComponent, BottomNavComponent |
| Clients | `/clients` | `clients` | List, Form, Detail |
| Services | `/services` | `services` | List, Form, Detail |
| Appointments | `/appointments` | *(pendente)* | Placeholder — backend já implementado |
| Financial | `/financial` | *(pendente)* | Placeholder — backend já implementado |
| Inventory | `/inventory` | *(pendente)* | Placeholder — backend já implementado |
| Dashboard | `/dashboard` | *(pendente)* | Placeholder — backend já implementado |

> O frontend está atrasado em relação ao backend: os módulos de Agendamentos, Financeiro, Estoque e Dashboard já têm API completa, mas ainda não têm UI/NgRx. O fluxo de registro/ativação/multi-tenancy e o endpoint admin de tenants (backend pronto) também não têm UI ainda. Próxima fase do roadmap.

---

## Banco de Dados — Schema

**Schema `public`** (compartilhado, sempre existe):

```
users ──────────────────────────────────────────────────────────  V100 (lash-core)
  + tenant_id (FK → tenants.id, nullable), activation_key,
    activation_key_expiry ─────────────────────────────────────── V102 (lash-core)
tenants (id, name, schema_name, active, created_at) ─────────────  V101 (lash-core)
command_audit_log (id, command_class, payload_json, user_id,
  executed_at, success) ────────────────────────────────────────── V103 (lash-core)
```

**Schema de cada tenant** (`tenant_<uuid-sem-hifen>`, provisionado sob demanda via Liquibase — `master.xml`):

```
clients ────────────────────────────────────────────────────────  (Flyway V200, reaproveitado via <sqlFile>)
services ───────────────────────────────────────────────────────  (Flyway V300)
appointments ──── FK → clients.id ──── FK → services.id ────────  (Flyway V400)
financial_entries ──── FK → appointments.id ─────────────────────  (Flyway V500)
inventory_items / inventory_movements ─── FK entre si ───────────  (Flyway V600)
fichas / lash_mappings ─── FK → clients.id ──────────────────────  (Flyway V700)
command_audit_log ────────────────────────────────────────────────  (Liquibase, fragmento próprio)
```

Status do appointment: `CHECK (status IN ('SCHEDULED','CONFIRMED','COMPLETED','CANCELLED'))`

**Importante:** `V101`/`V102`/`V103` têm número **menor** que migrations de domínio já aplicadas (`V200`+) — o Flyway precisa de `out-of-order: true` pra aceitar isso (ver STACK.md). Ranges de versão continuam reservados por módulo (`V100`–`V199` para `lash-core`, etc.) desde a refatoração multi-módulo.

---

## Segurança — Fluxo JWT (com tenant)

```
POST /api/auth/login
    │
    ▼
LoginUseCaseImpl (lash-core)
    ├── findByEmail → UserRepository (schema public, default)
    ├── passwordEncoder.matches(password, hash)
    ├── checa user.isActive() → AccountNotActivatedException se não
    ├── checa tenant.isActive() (se houver tenant) → TenantInactiveException se não
    └── TokenPort.generateAccessToken(email, role, tenantId) + generateRefreshToken
    │
    ▼
Response: { accessToken, refreshToken, name, email, role }
    (accessToken carrega claim "tenantId" — nulo pro usuário de plataforma/dev)

Requisições autenticadas:
    Authorization: Bearer <accessToken>
    │
    ▼
TenantResolvingFilter (lash-core)          ← roda ANTES do JwtAuthFilter
    ├── extrai claim tenantId do token (se válido)
    └── TenantContext.setCurrentTenant(schemaName) — limpo no finally
    │
    ▼
JwtAuthFilter (lash-core)
    ├── JwtService.extractEmail(token)
    ├── JwtService.isTokenValid(token)
    └── SecurityContextHolder.setAuthentication(...)
```

Token types discriminados via claim `"type"`: `"ACCESS"` vs `"REFRESH"`. `JwtAuthFilter`, `TenantResolvingFilter` e `SecurityConfig` vivem em `lash-core` e são reaproveitados por todos os módulos porque `lash-app` os traz no classpath e o `@ComponentScan` (via `scanBasePackages = "com.lashmanager"`) os registra globalmente.
