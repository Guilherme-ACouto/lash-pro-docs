# Arquitetura — Lash Manager

**Atualizado:** 2026-07-14

## Visão Geral

Monorepo com dois projetos independentes: `lash-backend/` (Java, **multi-módulo Maven**) e `lash-frontend/` (Angular). O backend segue **Arquitetura Hexagonal (Ports & Adapters)** dentro de cada módulo; o frontend segue **Feature-first com NgRx**.

---

## Backend — Multi-módulo Maven + Arquitetura Hexagonal

O backend foi dividido em **9 módulos Maven independentes**, um por domínio de negócio (+ um módulo executável). Cada módulo é internamente hexagonal — a divisão em módulos é uma fronteira física adicional por cima da fronteira lógica de camadas.

### Módulos e dependências

```
lash-core            ← base: User/Auth, exceções raiz, infra compartilhada (security, email)
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

**Comunicação entre módulos:** quando um módulo precisa de dado de outro (ex.: `lash-clients` precisa saber se um cliente tem agendamentos futuros antes de desativar), a comunicação é feita via **port/out implementado no módulo consumidor e satisfeito por um adapter no módulo que tem o dado** — nunca acessando repositório/entidade de outro módulo diretamente. Exemplo real: `ClientAppointmentPort` (definida em `lash-clients/domain/port/out`) é implementada em `lash-appointments/infrastructure/adapter`, e injetada em `lash-clients` via Spring (o bean vem do classpath de `lash-appointments`, presente porque `lash-app` traz ambos).

### Camadas e fluxo de uma requisição (dentro de cada módulo)

```
HTTP Request
    │
    ▼
[Controller]          {modulo}/adapter/web/controller/
    │  usa
    ▼
[port/in interface]   {modulo}/domain/port/in/
    │  implementado por
    ▼
[UseCase Impl]        {modulo}/application/usecase/
    │  usa
    ▼
[port/out interface]  {modulo}/domain/port/out/
    │  implementado por
    ▼
[RepositoryImpl]      {modulo}/infrastructure/persistence/repository/
    │  usa
    ▼
[JpaRepository]       {modulo}/infrastructure/persistence/repository/ (Spring Data)
    │
    ▼
PostgreSQL
```

### Estrutura de pacotes (repetida em cada módulo, prefixo `com.lashmanager.{modulo}`)

```
com.lashmanager.{modulo}
├── domain/                      ← zero dependência de framework
│   ├── model/                   ← entidades puras (Client, Appointment, FinancialEntry, ...)
│   ├── exception/                ← *Exception estendendo DomainException/BusinessException (de lash-core)
│   └── port/
│       ├── in/                  ← interfaces de use case (CreateClientUseCase, ...)
│       └── out/                 ← interfaces de repositório/integração (ClientRepository, ClientAppointmentPort, ...)
│
├── application/usecase/         ← implementações dos use cases
│   ├── CreateClientUseCaseImpl
│   ├── ClientUseCaseMapper      ← converte domain model → result record
│   └── [demais *UseCaseImpl]
│
├── infrastructure/
│   ├── persistence/
│   │   ├── entity/              ← @Entity JPA (ClientEntity, AppointmentEntity, ...)
│   │   ├── mapper/               ← converte Entity ↔ Domain Model
│   │   └── repository/          ← JpaRepository + RepositoryImpl
│   ├── adapter/                  ← implementações de port/out que cruzam módulo (ex.: ClientAppointmentPortImpl)
│   └── config/                   ← @Configuration específica do módulo, quando existe
│
└── adapter/web/
    ├── controller/               ← @RestController
    └── dto/                      ← records de Request e Response
```

`lash-core` adicionalmente concentra: `DomainException`/`BusinessException` (exceções raiz que todos os módulos estendem), `security/` (JwtService, SecurityConfig, JwtAuthFilter, UserDetailsServiceImpl) e `infrastructure/email/`.

`lash-app` concentra apenas: classe `LashManagerApplication` (`@SpringBootApplication` com `scanBasePackages`, `@EnableJpaRepositories` e `@EntityScan` em `com.lashmanager`, para varrer todos os módulos) e `GlobalExceptionHandler` (único `@RestControllerAdvice` do sistema — centraliza handlers de exceções de **todos** os módulos).

### Regras invioláveis

- `domain/` não tem nenhuma dependência de framework (sem `@Entity`, sem `@Service`, sem Spring) — vale em todos os módulos
- Use cases dependem apenas de interfaces (`port/out`), nunca de implementações
- Controllers dependem apenas de `port/in`, nunca de repositórios diretamente
- Injeção sempre via construtor — `@RequiredArgsConstructor`; nunca `@Autowired` em campo
- Módulos nunca acessam entidade JPA ou repositório de outro módulo — a fronteira é sempre um `port/out` + adapter

### Wiring dos use cases: `@Service` direto, sem classe de configuração

Em todos os 9 módulos, `*UseCaseImpl` é anotado diretamente com `@Service` (`import org.springframework.stereotype.Service;`) e `@RequiredArgsConstructor` — vira bean automaticamente via component scan (`@SpringBootApplication(scanBasePackages = "com.lashmanager")` em `lash-app` cobre todos os módulos). Não existe (e não deve ser criada) nenhuma classe `{Modulo}Config`/`@Bean` para registrar use cases manualmente — isso foi tentado brevemente durante o refactor multi-módulo, mas foi revertido a pedido da usuária por não ter sido solicitado.

**Nota histórica — conflito de nome `Service` (resolvido):** até 2026-07-14, `lash-services` tinha um `domain.model.Service` que colidia de nome com `@org.springframework.stereotype.Service`, exigindo anotação qualificada (`@org.springframework.stereotype.Service`, sem `import`) nos use cases desse módulo. O domain model foi renomeado para `ServiceOffering` — o conflito não existe mais, e `lash-services` usa `@Service` normal como qualquer outro módulo.

---

## Backend — Módulos Implementados

| Módulo | Domain Model | Use Cases | Endpoint base |
|---|---|---|---|
| lash-core | `User` | Login, Refresh, ForgotPassword (3 use cases) | `/api/auth/**` |
| lash-clients | `Client` | Create, Update, Get, List, Deactivate/Reactivate, Delete (7 use cases) | `/api/clients` |
| lash-services | `ServiceOffering` | Create, Update, Get, List, Deactivate (7 use cases) | `/api/services` |
| lash-appointments | `Appointment`, `AppointmentStatus` | Create, Update, Get, List, Cancel, ... (6 use cases) | `/api/appointments` |
| lash-finance | `FinancialEntry`, `MonthlyFinancialStat`, tipos/status | Create, Update, Get, List, ... (7 use cases) | `/api/financial` |
| lash-stock | `InventoryItem`, `InventoryMovement`, tipos de movimento | Create, Update, Get, List, registrar movimentação, ... (10 use cases) | `/api/inventory` |
| lash-fichas | `Ficha`, `LashMapping` | Create, Update, Get, List, mapeamento de lashes, ... (9 use cases) | `/api/fichas` |
| lash-dashboard | *(agrega leituras — sem model próprio)* | 2 use cases de agregação | `/api/dashboard` |
| lash-app | — (só bootstrap + GlobalExceptionHandler) | — | — |

Todos os 9 módulos estão implementados no backend (domain → use case → persistência → controller). A cobertura de testes automatizados, porém, hoje só existe em `lash-core` e `lash-clients` — ver TESTING.md.

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

> O frontend está atrasado em relação ao backend: os módulos de Agendamentos, Financeiro, Estoque e Dashboard já têm API completa (ver tabela acima), mas ainda não têm UI/NgRx. Próxima fase do roadmap.

---

## Banco de Dados — Schema

Mesmo schema físico de 8 tabelas, agora criado por **migrations separadas por módulo** em vez de um único `V1__create_schema.sql` (ver ARCHITECTURE das migrations em INTEGRATIONS.md e STACK.md):

```
users ──────────────────────────────────────────────────────────  V100 (lash-core)
clients ────────────────────────────────────────────────────────  V200 (lash-clients)
services ───────────────────────────────────────────────────────  V300 (lash-services)
appointments ──── FK → clients.id ──── FK → services.id ────────  V400 (lash-appointments)
financial_entries ──── FK → appointments.id ─────────────────────  V500 (lash-finance)
inventory_items / inventory_movements ─── FK entre si ───────────  V600 (lash-stock)
fichas / lash_mappings ─── FK → clients.id ──────────────────────  V700 (lash-fichas)
                                                                    V800 (lash-app: seed data)
```

Status do appointment: `CHECK (status IN ('SCHEDULED','CONFIRMED','COMPLETED','CANCELLED'))`

---

## Segurança — Fluxo JWT

```
POST /api/auth/login
    │
    ▼
LoginUseCaseImpl (lash-core)
    ├── findByEmail → UserRepository
    ├── passwordEncoder.matches(password, hash)
    └── TokenPort.generateAccessToken + generateRefreshToken
    │
    ▼
Response: { accessToken, refreshToken, name, email, role }

Requisições autenticadas:
    Authorization: Bearer <accessToken>
    │
    ▼
JwtAuthFilter (lash-core)
    ├── JwtService.extractEmail(token)
    ├── JwtService.isTokenValid(token, userDetails)
    └── SecurityContextHolder.setAuthentication(...)
```

Token types discriminados via claim `"type"`: `"ACCESS"` vs `"REFRESH"`. `JwtAuthFilter` e `SecurityConfig` vivem em `lash-core` e são reaproveitados por todos os módulos porque `lash-app` os traz no classpath e o `@ComponentScan` (via `scanBasePackages = "com.lashmanager"`) os registra globalmente.
