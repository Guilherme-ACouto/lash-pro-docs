# Estrutura do Projeto — Lash Manager

**Atualizado:** 2026-05-22

## Raiz do monorepo

```
/Users/Apple/Documents/Lash/
├── lash-backend/          ← Java 21 + Spring Boot 3
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

## Backend — `lash-backend/`

```
lash-backend/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/lashmanager/app/
    │   │   ├── LashBackendApplication.java
    │   │   ├── domain/
    │   │   │   ├── model/
    │   │   │   │   ├── Client.java
    │   │   │   │   ├── Service.java
    │   │   │   │   └── User.java
    │   │   │   ├── exception/
    │   │   │   │   ├── DomainException.java
    │   │   │   │   ├── ClientNotFoundException.java
    │   │   │   │   ├── ClientAlreadyExistsException.java
    │   │   │   │   ├── ServiceNotFoundException.java
    │   │   │   │   ├── InvalidCredentialsException.java
    │   │   │   │   ├── TokenExpiredException.java
    │   │   │   │   └── UserNotFoundException.java
    │   │   │   └── port/
    │   │   │       ├── in/
    │   │   │       │   ├── LoginUseCase.java
    │   │   │       │   ├── CreateClientUseCase.java
    │   │   │       │   ├── UpdateClientUseCase.java
    │   │   │       │   ├── GetClientUseCase.java
    │   │   │       │   ├── ListClientsUseCase.java
    │   │   │       │   ├── DeactivateClientUseCase.java
    │   │   │       │   ├── CreateServiceUseCase.java
    │   │   │       │   ├── UpdateServiceUseCase.java
    │   │   │       │   ├── GetServiceUseCase.java
    │   │   │       │   ├── ListServicesUseCase.java
    │   │   │       │   └── DeactivateServiceUseCase.java
    │   │   │       └── out/
    │   │   │           ├── ClientRepository.java
    │   │   │           ├── ServiceRepository.java
    │   │   │           ├── UserRepository.java
    │   │   │           └── TokenPort.java
    │   │   ├── application/usecase/
    │   │   │   ├── LoginUseCaseImpl.java
    │   │   │   ├── ClientUseCaseMapper.java
    │   │   │   ├── CreateClientUseCaseImpl.java
    │   │   │   ├── UpdateClientUseCaseImpl.java
    │   │   │   ├── GetClientUseCaseImpl.java
    │   │   │   ├── ListClientsUseCaseImpl.java
    │   │   │   ├── DeactivateClientUseCaseImpl.java
    │   │   │   ├── ServiceUseCaseMapper.java
    │   │   │   ├── CreateServiceUseCaseImpl.java
    │   │   │   ├── UpdateServiceUseCaseImpl.java
    │   │   │   ├── GetServiceUseCaseImpl.java
    │   │   │   ├── ListServicesUseCaseImpl.java
    │   │   │   └── DeactivateServiceUseCaseImpl.java
    │   │   ├── infrastructure/
    │   │   │   ├── persistence/
    │   │   │   │   ├── entity/
    │   │   │   │   │   ├── ClientEntity.java
    │   │   │   │   │   └── ServiceEntity.java
    │   │   │   │   ├── mapper/
    │   │   │   │   │   ├── ClientMapper.java
    │   │   │   │   │   └── ServiceMapper.java
    │   │   │   │   └── repository/
    │   │   │   │       ├── ClientJpaRepository.java
    │   │   │   │       ├── ClientRepositoryImpl.java
    │   │   │   │       ├── ServiceJpaRepository.java
    │   │   │   │       ├── ServiceRepositoryImpl.java
    │   │   │   │       ├── UserJpaRepository.java
    │   │   │   │       └── UserRepositoryImpl.java
    │   │   │   └── security/
    │   │   │       ├── JwtService.java
    │   │   │       ├── SecurityConfig.java
    │   │   │       ├── JwtAuthFilter.java
    │   │   │       └── UserDetailsServiceImpl.java
    │   │   └── adapter/web/
    │   │       ├── controller/
    │   │       │   ├── AuthController.java
    │   │       │   ├── ClientController.java
    │   │       │   ├── ServiceController.java
    │   │       │   └── GlobalExceptionHandler.java
    │   │       └── dto/
    │   │           ├── LoginRequest.java / LoginResponse.java
    │   │           ├── CreateClientRequest.java / UpdateClientRequest.java / ClientResponse.java
    │   │           ├── CreateServiceRequest.java / UpdateServiceRequest.java / ServiceResponse.java
    │   │           └── ErrorResponse.java
    │   └── resources/
    │       ├── application.yml
    │       └── db/migration/
    │           ├── V1__create_schema.sql
    │           └── V2__seed_data.sql
    └── test/java/                ← configurado, sem testes implementados
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
        │   ├── appointments/       ← placeholder (shell component apenas)
        │   ├── financial/          ← placeholder
        │   ├── inventory/          ← placeholder
        │   └── dashboard/          ← placeholder
        ├── layout/
        │   ├── shell/shell.component.ts
        │   ├── sidebar/sidebar.component.ts     ← desktop
        │   └── bottom-nav/bottom-nav.component.ts ← mobile
        └── shared/
            └── pipes/
                ├── currency-br.pipe.ts
                ├── date-ptbr.pipe.ts
                └── duration.pipe.ts              ← novo (formata minutos)
```

---

## Banco de dados — `db/migration/`

| Arquivo | Conteúdo |
|---|---|
| `V1__create_schema.sql` | 8 tabelas: users, clients, services, appointments, financial_entries, inventory_items, inventory_movements + índices + constraints |
| `V2__seed_data.sql` | 1 usuário admin + 4 serviços padrão |

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
