# Arquitetura — Lash Manager

**Atualizado:** 2026-05-22

## Visão Geral

Monorepo com dois projetos independentes: `lash-backend/` (Java) e `lash-frontend/` (Angular). O backend segue **Arquitetura Hexagonal (Ports & Adapters)**; o frontend segue **Feature-first com NgRx**.

---

## Backend — Arquitetura Hexagonal

### Camadas e fluxo de uma requisição

```
HTTP Request
    │
    ▼
[Controller]          adapter/web/controller/
    │  usa
    ▼
[port/in interface]   domain/port/in/
    │  implementado por
    ▼
[UseCase Impl]        application/usecase/
    │  usa
    ▼
[port/out interface]  domain/port/out/
    │  implementado por
    ▼
[RepositoryImpl]      infrastructure/persistence/repository/
    │  usa
    ▼
[JpaRepository]       infrastructure/persistence/repository/ (Spring Data)
    │
    ▼
PostgreSQL
```

### Estrutura de pacotes

```
com.lashmanager.app
├── domain/                      ← zero dependência de framework
│   ├── model/                   ← entidades puras (Client, Service, User)
│   ├── exception/               ← DomainException e subclasses
│   └── port/
│       ├── in/                  ← interfaces de use case (CreateClientUseCase, ...)
│       └── out/                 ← interfaces de repositório (ClientRepository, ...)
│
├── application/usecase/         ← implementações dos use cases
│   ├── CreateClientUseCaseImpl
│   ├── ClientUseCaseMapper      ← converte domain model → result record
│   └── [demais *UseCaseImpl]
│
├── infrastructure/
│   ├── persistence/
│   │   ├── entity/              ← @Entity JPA (ClientEntity, ServiceEntity, ...)
│   │   ├── mapper/              ← converte Entity ↔ Domain Model
│   │   └── repository/         ← JpaRepository + RepositoryImpl
│   └── security/               ← JwtService, SecurityConfig, JwtAuthFilter
│
└── adapter/web/
    ├── controller/              ← @RestController + GlobalExceptionHandler
    └── dto/                     ← records de Request e Response
```

### Regras invioláveis

- `domain/` não tem nenhuma dependência de framework (sem `@Entity`, sem `@Service`, sem Spring)
- Use cases dependem apenas de interfaces (`port/out`), nunca de implementações
- Controllers dependem apenas de `port/in`, nunca de repositórios diretamente
- Injeção sempre via construtor — `@RequiredArgsConstructor`; nunca `@Autowired` em campo

### Nota: conflito de nome `Service`

`domain.model.Service` compartilha nome com `@org.springframework.stereotype.Service`. Nas implementações de use case, a anotação Spring é usada com nome qualificado completo:

```java
@org.springframework.stereotype.Service
@RequiredArgsConstructor
public class CreateServiceUseCaseImpl implements CreateServiceUseCase { ... }
```

---

## Backend — Módulos Implementados

| Módulo | Domain Model | Use Cases | Endpoint base |
|---|---|---|---|
| Auth | `User` | Login, Refresh, ForgotPassword | `/api/auth/**` |
| Clients | `Client` | Create, Update, Get, List, Deactivate | `/api/clients` |
| Services | `Service` | Create, Update, Get, List, Deactivate | `/api/services` |
| Appointments | *(schema criado)* | *(não implementado)* | — |
| Financial | *(schema criado)* | *(não implementado)* | — |
| Inventory | *(schema criado)* | *(não implementado)* | — |

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
| Appointments | `/appointments` | *(pendente)* | Placeholder |
| Financial | `/financial` | *(pendente)* | Placeholder |
| Inventory | `/inventory` | *(pendente)* | Placeholder |
| Dashboard | `/dashboard` | *(pendente)* | Placeholder |

---

## Banco de Dados — Schema

8 tabelas definidas em `V1__create_schema.sql`:

```
users ──────────────────────────────────────────────────────────
clients ────────────────────────────────────────────────────────
services ───────────────────────────────────────────────────────
appointments ──── FK → clients.id ──── FK → services.id ────────
financial_entries ──────────────────────────────────────────────
inventory_items ─────────────────────────────────────────────────
inventory_movements ─── FK → inventory_items.id ─────────────────
```

Status do appointment: `CHECK (status IN ('SCHEDULED','CONFIRMED','COMPLETED','CANCELLED'))`

---

## Segurança — Fluxo JWT

```
POST /api/auth/login
    │
    ▼
LoginUseCaseImpl
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
JwtAuthFilter
    ├── JwtService.extractEmail(token)
    ├── JwtService.isTokenValid(token, userDetails)
    └── SecurityContextHolder.setAuthentication(...)
```

Token types discriminados via claim `"type"`: `"ACCESS"` vs `"REFRESH"`.
