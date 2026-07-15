# Refatoração Backend → Padrão Pontta — Tasks

**Design**: `.specs/features/refactor-backend/design.md`
**Status**: Fases A, B, C, D, E e F concluídas (código implementado)
**Nota de verificação**: a usuária optou por rodar `mvn compile`/`mvn test`/validação manual só ao final de todo o projeto (todas as fases), não fase a fase. Os checkboxes de "Feito quando" que dependem de gate/execução ficam pendentes até essa validação final — o código de cada task é implementado e revisado estaticamente durante o desenvolvimento.

---

## Plano de Execução

```
Fase A (fundação multi-tenant — sequencial, tudo em lash-core)
T01 (Tenant model+migration) → T02 (TenantContext+Resolver) → T03 (MultiTenantConnectionProvider)
   → T04 (JwtService: claim tenantId) → T05 (TenantResolvingFilter) → T06 (UserEntity: tenant_id)
   → T07 (Hibernate config wiring) → T08 (checkpoint: 1 tenant só, nada quebrou)

Fase B (Command/ApplicationService infra + Registro/Ativação — sequencial, T14 em paralelo)
T09 (AbstractCommand+CommandInterceptor) → T10 (command_audit_log) → T11 (SchemaProvisionerImpl/Liquibase)
   → T12 (Register: Command+AppService+UseCase) → T13 (ResendActivation) → T14 [P] (template e-mail)
   → T15 (ActivateAccount) → T16 (login bloqueado p/ inativo) → T17 (checkpoint: fluxo completo)

Fase C (testes — depende da Fase B completa)
T18 (schema de teste) → T19 (isolamento entre tenants) → T20 (e2e registro→ativação→login)
   → T21 (CommandInterceptor: validação + audit log)

Fase D (operação — depende da Fase B)
T22 (endpoint admin: listar/desativar tenant)

Fase E (migração dos 42 UseCases existentes p/ Command/ApplicationService — todas [P] entre si,
        depende do piloto T12/T13/T15 validado em T17; granularidade grossa proposital, detalhar
        arquivo-por-arquivo só ao iniciar cada módulo)
T23 lash-clients [P] · T24 lash-services [P] · T25 lash-appointments [P] · T26 lash-finance [P]
T27 lash-stock [P] · T28 lash-fichas [P] · T29 lash-dashboard [P]

Fase F (separação de repositórios de leitura — Query — todas [P] entre si, depende de T17;
        pode rodar em paralelo à Fase E já que mexe em arquivos diferentes por módulo)
T30 lash-clients [P] · T31 lash-services [P] · T32 lash-appointments [P] · T33 lash-finance [P]
T34 lash-stock [P] · T35 lash-fichas [P] · T36 lash-dashboard [P]
```

---

## Fase A — Fundação Técnica (Multi-Tenancy) ✅ Concluída

**Status**: Implementada em `lash-core` (T01–T07). T08 (checkpoint de validação manual) fica registrado como pendente — será feito junto da validação final de todo o projeto, por decisão da usuária.

### T01: Modelo `Tenant` + migration `public`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Criar `Tenant` (domain), `TenantEntity`/`TenantJpaRepository`/`TenantMapper`/`TenantRepositoryImpl`, e migration Flyway que cria a tabela `tenants` no schema `public`
**Onde**: `lash-core/src/main/java/com/lashmanager/core/domain/model/Tenant.java`, `infrastructure/persistence/Tenant{Entity,JpaRepository,Mapper,RepositoryImpl}.java`, `lash-core/src/main/resources/db/migration/core/V1xx__create_tenants_table.sql`
**Depende de**: Nenhum
**Reusa**: padrão de `ClientEntity`/`ClientMapper`/`ClientRepositoryImpl`
**Requirement**: RBK-01

**Feito quando**:
- [ ] `Tenant` record/classe com `id (UUID)`, `name`, `schemaName`, `active`, `createdAt`
- [ ] Migration cria `tenants(id, name, schema_name, active, created_at)` com `schema_name` único
- [ ] `TenantRepository` (port/out) com `save`, `findById`, `existsBySchemaName`
- [ ] Gate: `mvn compile -pl lash-core` sem erros

**Tests**: unit (mapper)
**Gate**: quick (`mvn test -pl lash-core`)

---

### T02: `TenantContext` + `CurrentTenantIdentifierResolverImpl`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Contexto de tenant (ThreadLocal) e resolver do Hibernate com default `public`
**Onde**: `lash-core/src/main/java/com/lashmanager/core/infrastructure/multitenancy/TenantContext.java`, `CurrentTenantIdentifierResolverImpl.java`
**Depende de**: T01
**Requirement**: RBK-03, RBK-04

**Feito quando**:
- [ ] `TenantContext.setCurrentTenant(String)`, `getCurrentTenant()`, `clear()`
- [ ] `CurrentTenantIdentifierResolverImpl implements CurrentTenantIdentifierResolver<String>` retorna `"public"` quando `TenantContext` vazio
- [ ] `validateExistingCurrentSessions()` retorna `false`
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit
**Gate**: quick

---

### T03: `MultiTenantConnectionProviderImpl`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Provider que troca `search_path` por conexão obtida do pool
**Onde**: `lash-core/src/main/java/com/lashmanager/core/infrastructure/multitenancy/MultiTenantConnectionProviderImpl.java`
**Depende de**: T02
**Requirement**: RBK-05

**Feito quando**:
- [ ] `getAnyConnection()` retorna conexão do `DataSource` padrão (schema `public`)
- [ ] `getConnection(String tenantIdentifier)` executa `SET search_path TO "<tenantIdentifier>", public`
- [ ] `releaseConnection(...)` reseta `search_path TO public` antes de devolver ao pool
- [ ] Gate: teste de integração conecta, troca schema, confirma via `SHOW search_path`

**Tests**: integration
**Gate**: full (`mvn test -pl lash-core`, requer banco)

---

### T04: `JwtService` — claim `tenantId`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Incluir `tenantId` no payload do JWT (login e refresh)
**Onde**: `lash-core/src/main/java/com/lashmanager/core/infrastructure/security/JwtService.java` (modificar)
**Depende de**: T01
**Requirement**: RBK-07

**Feito quando**:
- [ ] `generateToken(user)` inclui claim `tenantId` (nulo permitido para `admin@lashmanager.com`, sem tenant)
- [ ] `extractTenantId(token)` novo método de leitura do claim
- [ ] Testes unitários de `JwtServiceTest` cobrindo geração e extração do claim
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit
**Gate**: quick

---

### T05: `TenantResolvingFilter`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: `OncePerRequestFilter` que lê o claim `tenantId` do JWT já validado e popula `TenantContext`, limpando no `finally`
**Onde**: `lash-core/src/main/java/com/lashmanager/core/infrastructure/security/TenantResolvingFilter.java`
**Depende de**: T02, T04
**Requirement**: RBK-04

**Feito quando**:
- [ ] Filtro registrado na `SecurityConfig` antes do filtro de autenticação
- [ ] Popula `TenantContext` quando há claim; deixa vazio (default `public`) quando não há
- [ ] `TenantContext.clear()` sempre executado no `finally`
- [ ] Teste de integração: request com JWT de tenant X só enxerga dado do schema X

**Tests**: integration
**Gate**: full

---

### T06: `UserEntity` — colunas `tenant_id`, `activation_key`, `activation_key_expiry`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Migration + entidade atualizadas
**Onde**: `lash-core/src/main/resources/db/migration/core/V1xx__add_tenant_and_activation_to_users.sql`, `UserEntity.java` (modificar)
**Depende de**: T01
**Requirement**: RBK-06, RBK-09, RBK-12

**Feito quando**:
- [ ] Colunas `tenant_id (nullable, FK)`, `activation_key (nullable)`, `activation_key_expiry (nullable)` adicionadas a `users`
- [ ] `UserEntity` com os três campos novos
- [ ] Migration testada em banco local (usuária valida)

**Tests**: none (coberto indiretamente por T12/T15)
**Gate**: build (`mvn compile`)

---

### T07: Wiring do Hibernate multi-tenant

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Configurar `application.yml`/`@Configuration` para `MultiTenancyStrategy.SCHEMA`
**Onde**: `lash-app/src/main/resources/application.yml`, `lash-core/src/main/java/com/lashmanager/core/infrastructure/multitenancy/HibernateMultiTenantConfig.java` (novo)
**Depende de**: T02, T03
**Requirement**: RBK-02

**Feito quando**:
- [ ] `hibernate.multiTenancy=SCHEMA`, provider e resolver configurados
- [ ] Aplicação sobe local sem erro de inicialização do Hibernate
- [ ] Login com `admin@lashmanager.com` (sem tenant) continua funcionando (schema `public` default)

**Tests**: integration
**Gate**: full

---

### T08: Checkpoint — validação manual da Fase A

**Status**: ⏳ Pendente — usuária vai validar manualmente ao final de todas as fases, não fase a fase

**O que**: Validação de que a fundação não quebrou nada, com 1 tenant só (schema `public` default)
**Onde**: N/A
**Depende de**: T01–T07
**Requirement**: RBK-08

**Feito quando**:
- [ ] `mvn test` (reactor completo) passa
- [ ] Login, CRUD de clientes/serviços/agendamentos/financeiro funcionam manualmente (usuária testa)

**Tests**: none (checkpoint)
**Gate**: full (`mvn test`)

---

## Fase B — Command/ApplicationService (piloto) + Registro/Ativação ✅ Concluída

**Status**: Implementada em `lash-core` (T09–T16). T17 (checkpoint de validação manual / go-no-go pra Fase E/F) fica registrado como pendente — validação final de todo o projeto, por decisão da usuária.

### T09: `AbstractCommand` + `CommandInterceptor`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Base de todo Command e o `@Aspect` que intercepta `when(...)` de qualquer `ApplicationService`: valida (Bean Validation), loga início/fim, prepara o hook de auditoria (gravação real depende de T10)
**Onde**: `lash-core/src/main/java/com/lashmanager/core/infrastructure/command/AbstractCommand.java`, `CommandInterceptor.java`
**Depende de**: Nenhum (pode rodar em paralelo à Fase A) `[P]`
**Requirement**: RBK-20, RBK-22

**Feito quando**:
- [ ] `AbstractCommand` com `commandId (UUID)`, `issuedAt (LocalDateTime)`
- [ ] `CommandInterceptor` intercepta `execution(* com.lashmanager..*.application.service.*ApplicationService.when(..))`
- [ ] Bean Validation roda sobre o `Command` recebido antes do `proceed()`; `ConstraintViolationException` propagada se inválido
- [ ] Teste com um `ApplicationService`/`Command` de exemplo (test-only) confirmando: valida, loga, intercepta exceção
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit
**Gate**: quick

---

### T10: `command_audit_log`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Tabela de auditoria (schema de tenant) + entidade/repositório, e `CommandInterceptor` (T09) passa a gravar de verdade
**Onde**: `lash-core/src/main/resources/db/changelog/fragments/command-audit-log.xml` (changeset Liquibase, incluído no `master.xml` de T11), `infrastructure/persistence/CommandAuditLogEntity.java`, `CommandAuditLogJpaRepository.java`, `domain/port/out/CommandAuditLogRepository.java`
**Depende de**: T09
**Requirement**: RBK-23

**Feito quando**:
- [ ] Changeset cria `command_audit_log(id, command_class, payload_json, user_id, executed_at, success)`
- [ ] `CommandInterceptor` grava uma linha por execução (sucesso ou falha)
- [ ] Teste de integração: executar um Command de exemplo grava a linha esperada
- [ ] Gate: `mvn test -pl lash-core` (requer banco)

**Tests**: integration
**Gate**: full

---

### T11: `SchemaProvisionerImpl` (Liquibase)

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Provisionamento de schema — `CREATE SCHEMA IF NOT EXISTS` + Liquibase update (incluindo o changeset de `command_audit_log` de T10), em conexão JDBC própria (fora da transação Spring)
**Onde**: `lash-core/src/main/java/com/lashmanager/core/infrastructure/multitenancy/SchemaProvisionerImpl.java`, `lash-core/src/main/resources/db/changelog/master.xml`
**Depende de**: T07, T10
**Requirement**: RBK-02, RBK-15

**Feito quando**:
- [ ] `provision(tenantId)`: `DriverManager` (conexão própria, fora do pool), `CREATE SCHEMA IF NOT EXISTS "tenant_<id>"`, Liquibase `update()` com `master.xml`
- [ ] Idempotente: rodar duas vezes seguidas não falha nem duplica
- [ ] Migrations Flyway existentes (V200+ clients, V300+ services etc.) convertidas para changesets Liquibase equivalentes nos módulos de domínio
- [ ] Teste de integração: provisiona schema novo, valida tabelas criadas (incluindo `command_audit_log`), roda de novo (idempotência)

**Tests**: integration
**Gate**: full

---

### T12: `Register` — Command + ApplicationService + UseCase

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: `POST /api/register`, já no padrão novo desde o início
**Onde**: `lash-core/application/command/RegisterCommand.java`, `application/service/RegisterApplicationService.java`, `domain/port/in/RegisterUseCase.java`, `application/usecase/RegisterUseCaseImpl.java`, `adapter/web/controller/RegisterController.java`
**Depende de**: T06, T08, T09
**Reusa**: padrão `CreateClientUseCaseImpl` para a regra de negócio interna
**Requirement**: RBK-09, RBK-10 (parcial), RBK-21, RBK-24

**Feito quando**:
- [ ] `RegisterCommand` com `@NotBlank`/`@Email` nos campos
- [ ] `RegisterApplicationService.when(RegisterCommand)` delega pro `RegisterUseCaseImpl`
- [ ] E-mail novo → cria usuário inativo, `activationKey`/`activationKeyExpiry`
- [ ] E-mail existente e ativo → 409; e-mail existente e inativo → delega pra `ResendActivationUseCase` (T13)
- [ ] Testes unitários dos três casos + teste do `ApplicationService` (delega corretamente)
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit
**Gate**: quick

---

### T13: `ResendActivation` — Command + ApplicationService + UseCase

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: `POST /api/register/resend`, mesmo padrão de T12
**Onde**: `lash-core/application/command/ResendActivationCommand.java`, `application/service/ResendActivationApplicationService.java`, `domain/port/in/ResendActivationUseCase.java`, `application/usecase/ResendActivationUseCaseImpl.java`, endpoint em `RegisterController`
**Depende de**: T12
**Requirement**: RBK-13, RBK-10, RBK-21, RBK-24

**Feito quando**:
- [ ] `!user.isActive()` → renova `activationKey`/`activationKeyExpiry`, reenvia e-mail
- [ ] E-mail não encontrado ou já ativo → resposta genérica idêntica à de sucesso
- [ ] Testes unitários dos três casos
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit
**Gate**: quick

---

### T14: Template de e-mail `activation.html`

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Template Thymeleaf reaproveitando o fragmento de rodapé do "esqueci senha"
**Onde**: `lash-core/src/main/resources/templates/mails/activation.html`
**Depende de**: Nenhum `[P]`
**Requirement**: RBK-11

**Feito quando**:
- [ ] Assunto + saudação com nome + botão "Confirmar cadastro" com link `{baseUrl}/activation?key={key}`
- [ ] Reaproveita fragmento de rodapé compartilhado
- [ ] Envio real testado manualmente (usuária confirma recebimento)

**Tests**: none (visual)
**Gate**: build

---

### T15: `ActivateAccount` — Command + ApplicationService + UseCase

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: `GET/POST /api/activation`, orquestra: valida key/expiração, cria Tenant + ativa usuário (transação Spring), chama `SchemaProvisionerImpl` (fora da transação)
**Onde**: `lash-core/application/command/ActivateAccountCommand.java`, `application/service/ActivateAccountApplicationService.java`, `domain/port/in/ActivateAccountUseCase.java`, `application/usecase/ActivateAccountUseCaseImpl.java`, `adapter/web/controller/ActivationController.java`
**Depende de**: T01, T11, T12
**Requirement**: RBK-14, RBK-15, RBK-12, RBK-21, RBK-24

**Feito quando**:
- [ ] Key não encontrada → 404 "link inválido"; key expirada → 410 "link expirado" (mensagem distinta)
- [ ] Key válida → cria `Tenant`, ativa usuário, então chama provisionamento; falha no provisionamento reverte a transação Spring (rollback) e propaga erro tratável
- [ ] Retry com a mesma key (ainda válida) após falha completa o fluxo com sucesso (idempotência ponta a ponta)
- [ ] Testes unitários (mock de `SchemaProvisionerPort`) + 1 teste de integração com fluxo real
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit + integration
**Gate**: full

---

### T16: Login bloqueado para usuário inativo

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Ajustar `UserDetailsServiceImpl`/checker para lançar erro claro quando `!active`
**Onde**: `lash-core/src/main/java/com/lashmanager/core/infrastructure/security/UserDetailsServiceImpl.java` (modificar)
**Depende de**: T06
**Requirement**: RBK-16

**Feito quando**:
- [ ] Login de usuário `active=false` retorna erro com mensagem "confirme seu cadastro"
- [ ] Teste unitário cobrindo o caso
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit
**Gate**: quick

---

### T17: Checkpoint — Fase B completa (piloto validado)

**Status**: ⏳ Pendente — usuária vai validar manualmente ao final de todas as fases, não fase a fase

**O que**: Validação manual do fluxo ponta a ponta — este é o ponto de decisão para destravar as Fases E/F
**Onde**: N/A
**Depende de**: T09–T16
**Requirement**: RBK-09 a RBK-27

**Feito quando**:
- [ ] Registro → e-mail recebido → clique no link → tenant criado → schema provisionado (com `command_audit_log` presente) → login funciona (usuária testa manualmente)
- [ ] `command_audit_log` tem entradas para cada Command executado no fluxo
- [ ] `mvn test` (reactor completo) passa
- [ ] Usuária valida que o padrão Command/ApplicationService está ergonômico o suficiente para replicar nos outros módulos (go/no-go informal para Fase E/F)

**Tests**: none (checkpoint)
**Gate**: full

---

## Fase C — Testes ✅ Concluída

**Status**: Implementada em `lash-core` (T18, T20, T21) e `lash-clients` (T18). Gate de execução (`mvn test`) adiado para a validação final do projeto.

**Revisão de estratégia (2026-07-15, alinhando com a Pontta):** depois de consultar como a Pontta testa multi-tenancy, mudamos de "cada teste roda contra um banco/schema efêmero" para o padrão deles: **banco de teste separado** (`lashmanager_test`, distinto do banco de dev), um **schema de tenant fixo** para toda a suíte (já era assim, via `TEST_TENANT_ID`), e **provisionamento de schema mockado** nos testes que envolvem ativação de conta — evita criar schemas Postgres reais a cada execução de teste. T19 (isolamento real entre dois schemas) foi **descartado**, mesma decisão da Pontta: eles também não têm esse teste, aceitando que a integração real do provisionamento só é verificada em ambiente real (dev/staging).

### T18: `AbstractIntegrationTest` — schema de teste

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Setup que provisiona um schema de tenant fixo (`TEST_TENANT_ID`) antes de cada teste, num banco de teste separado do de dev (`lashmanager_test`)
**Onde**: `lash-core/src/test/java/com/lashmanager/core/AbstractIntegrationTest.java`, `lash-clients/.../AbstractIntegrationTest.java`, `application-test.yml` de ambos os módulos (datasource → `lashmanager_test`)
**Depende de**: T11
**Requirement**: RBK-17

**Feito quando**:
- [ ] `application-test.yml` aponta para `lashmanager_test` (banco separado do de dev), não `lashmanager`
- [ ] `@BeforeEach` provisiona um schema de tenant fixo (`TEST_TENANT_ID`) — idempotente, mesmo schema reaproveitado por toda a suíte
- [ ] `TenantContext.clear()` no `@AfterEach`
- [ ] Os 28 testes existentes de `lash-clients` continuam passando

**Tests**: integration
**Gate**: full (`mvn test -pl lash-clients`)

---

### T19: ~~Teste de isolamento entre tenants~~ — descartado

**Status**: ❌ Descartado (decisão da usuária, 2026-07-15) — replicando a escolha da Pontta

**O que**: A Pontta não tem um teste equivalente — evita provar isolamento real entre dois schemas criando-os de fato, pelo mesmo motivo que nos levou a criar esse teste (acumular schema no banco). Optamos por não manter essa cobertura automatizada; o isolamento estrutural (mecanismo do Hibernate/`search_path`) já foi implementado na Fase A e pode ser conferido manualmente em ambiente real
**Onde**: N/A — `TenantIsolationITest.java` removido
**Depende de**: T18
**Requirement**: RBK-18 (parcialmente não coberto por teste automatizado — risco aceito)

---

### T20: Teste e2e — registro → ativação → login

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Teste de integração cobrindo a orquestração completa registro → ativação → login. `SchemaProvisionerPort` é mockado (`@MockBean`) — testa que a orquestração chama o provisionamento com o `tenantId` certo, sem criar schema Postgres real, mesma escolha da Pontta (`SignatureCreatorTest`/`SignatureInitializerTest` lá também mockam essa camada)
**Onde**: `lash-core/src/test/java/com/lashmanager/core/RegistrationFlowITest.java`
**Depende de**: T17
**Requirement**: RBK-18

**Feito quando**:
- [ ] Registro cria usuário inativo com `activationKey`/`tenantId`
- [ ] Ativação (com key válida) ativa o usuário, cria o `Tenant`, e chama `schemaProvisionerPort.provision(tenantId)` — verificado via `Mockito.verify`, não por schema real
- [ ] Login funciona após ativação
- [ ] Ativação com key inválida lança `ActivationKeyInvalidException`
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: integration
**Gate**: full

---

### T21: Teste do `CommandInterceptor` — validação + auditoria

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Prova de que o interceptor valida e audita corretamente em cenário de sucesso e de falha
**Onde**: `lash-core/src/test/java/com/lashmanager/core/CommandInterceptorITest.java` (novo)
**Depende de**: T17
**Requirement**: RBK-22, RBK-23

**Feito quando**:
- [ ] Command inválido → `ConstraintViolationException`, nenhuma linha de sucesso em `command_audit_log`
- [ ] Command válido cuja execução lança exceção → linha de falha gravada
- [ ] Command válido com sucesso → linha de sucesso gravada com payload correto
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: integration
**Gate**: full

---

## Fase D — Operação ✅ Concluída

**Status**: Implementada em `lash-core` (T22). Gate de execução adiado para a validação final do projeto.


### T22: Endpoint administrativo mínimo (tenants)

**Status**: ✅ Implementado (código) — verificação (gate) adiada para a validação final do projeto

**O que**: Listar e desativar tenants
**Onde**: `lash-core/domain/port/in/{List,Deactivate}TenantUseCase.java`, `application/usecase/*Impl.java`, `adapter/web/controller/TenantAdminController.java`
**Depende de**: T17
**Reusa**: padrão `ListClientsUseCase`/`DeactivateClientUseCase`
**Requirement**: RBK-19

**Feito quando**:
- [ ] `GET /api/admin/tenants` lista tenants (paginado)
- [ ] `PATCH /api/admin/tenants/{id}/deactivate` desativa (usuários do tenant não conseguem mais logar)
- [ ] Restrito a papel administrativo (reaproveitar `UserRole` existente)
- [ ] Testes unitários dos use cases
- [ ] Gate: `mvn test -pl lash-core`

**Tests**: unit
**Gate**: quick

---

## Fase E — Migração dos UseCases Existentes para Command/ApplicationService ✅ Concluída

**Status**: Implementada em lash-clients, lash-services, lash-appointments, lash-finance, lash-stock, lash-fichas (T23–T28) — cada operação de escrita ganhou Command+ApplicationService, Controllers atualizados para chamar o ApplicationService (endpoints de leitura continuam chamando o UseCase direto). **T29** (lash-dashboard): confirmado que não há nada a migrar — módulo 100% leitura.


**Pré-requisito de todas as tasks abaixo**: T17 (piloto validado). Cada task é **intencionalmente grossa** — a granularidade fina (um Command/ApplicationService por operação) só é definida ao iniciar o módulo, espelhando exatamente o padrão de T12/T13/T15. Todas rodam em paralelo entre si (módulos diferentes, sem dependência de código entre eles).

### T23: `lash-clients` → Command/ApplicationService `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Migrar os 6 UseCases existentes (Create/Update/Get/List/Delete/DeactivateClient) para o padrão Command+ApplicationService; UseCases internos praticamente inalterados
**Onde**: `lash-clients/src/main/java/com/lashmanager/clients/application/{command,service}/`
**Depende de**: T17
**Reusa**: padrão validado em T12/T13/T15
**Requirement**: RBK-26

**Feito quando**:
- [ ] Um `Command` + `ApplicationService` por operação, Controller passa a chamar o `ApplicationService`
- [ ] Os 28 testes existentes continuam passando (ajustados para o novo ponto de entrada)
- [ ] Gate: `mvn test -pl lash-clients`

**Tests**: unit (+ integration nos ITests existentes)
**Gate**: full

---

### T24: `lash-services` → Command/ApplicationService `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Migrar os 6 UseCases existentes
**Onde**: `lash-services/src/main/java/com/lashmanager/services/application/{command,service}/`
**Depende de**: T17
**Requirement**: RBK-26

**Feito quando**:
- [ ] Mesmo padrão de T23 aplicado
- [ ] Gate: `mvn test -pl lash-services`

**Tests**: unit
**Gate**: quick

---

### T25: `lash-appointments` → Command/ApplicationService `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Migrar os 5 UseCases existentes
**Onde**: `lash-appointments/src/main/java/com/lashmanager/appointments/application/{command,service}/`
**Depende de**: T17
**Requirement**: RBK-26

**Feito quando**:
- [ ] Mesmo padrão de T23 aplicado
- [ ] Gate: `mvn test -pl lash-appointments`

**Tests**: unit
**Gate**: quick

---

### T26: `lash-finance` → Command/ApplicationService `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Migrar os 6 UseCases existentes
**Onde**: `lash-finance/src/main/java/com/lashmanager/finance/application/{command,service}/`
**Depende de**: T17
**Requirement**: RBK-26

**Feito quando**:
- [ ] Mesmo padrão de T23 aplicado
- [ ] Gate: `mvn test -pl lash-finance`

**Tests**: unit
**Gate**: quick

---

### T27: `lash-stock` → Command/ApplicationService `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Migrar os 9 UseCases existentes
**Onde**: `lash-stock/src/main/java/com/lashmanager/stock/application/{command,service}/`
**Depende de**: T17
**Requirement**: RBK-26

**Feito quando**:
- [ ] Mesmo padrão de T23 aplicado
- [ ] Gate: `mvn test -pl lash-stock`

**Tests**: unit
**Gate**: quick

---

### T28: `lash-fichas` → Command/ApplicationService `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Migrar os 8 UseCases existentes
**Onde**: `lash-fichas/src/main/java/com/lashmanager/fichas/application/{command,service}/`
**Depende de**: T17
**Requirement**: RBK-26

**Feito quando**:
- [ ] Mesmo padrão de T23 aplicado
- [ ] Gate: `mvn test -pl lash-fichas`

**Tests**: unit
**Gate**: quick

---

### T29: `lash-dashboard` → Command/ApplicationService `[P]`

**Status**: ✅ Confirmado sem código a migrar — os 2 UseCases (`GetDashboardSummaryUseCase`, `GetTodayScheduleUseCase`) são 100% leitura, sem nenhuma operação de escrita; não há Command a criar (mesma conclusão do T36 na Fase F)

**O que**: ~~Migrar os 2 UseCases existentes~~ — confirmado que o módulo inteiro é leitura, nada a fazer
**Onde**: N/A
**Depende de**: T17
**Requirement**: RBK-26

**Feito quando**:
- [x] Confirmado que não há operação de escrita em `lash-dashboard` para migrar
- [ ] Gate: `mvn test -pl lash-dashboard`

**Tests**: unit
**Gate**: quick

---

## Fase F — Separação de Repositórios de Leitura (Query) ✅ Concluída

**Status**: Implementada em lash-clients, lash-services, lash-appointments, lash-finance, lash-stock, lash-fichas (T30–T35) + documentação (TD01). **T36 revisado**: `DashboardRepositoryImpl` não existe no código atual — os dois use cases de `lash-dashboard` já compõem leitura direto dos `JpaRepository`s de outros módulos, sem porta de escrita própria da qual seria preciso separar; não havia nada pra renomear. Ver nota em STATE.md.


**Pré-requisito**: T17. Generaliza o padrão já usado em `FinancialSummaryRepositoryImpl`/Dashboard para os módulos que ainda misturam leitura e escrita no mesmo `Repository`. Pode rodar em paralelo à Fase E (arquivos diferentes).

### T30: `lash-clients` — Query `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Separar `ListClientsUseCase`/`GetClientUseCase` para usar um `ClientQueryRepository` (Specification sobre JPA, já que a listagem hoje é filtro simples — não precisa de SQL nativo)
**Onde**: `lash-clients/domain/port/out/ClientQueryRepository.java`, `infrastructure/persistence/ClientQueryRepositoryImpl.java`
**Depende de**: T17
**Requirement**: RBK-27

**Feito quando**:
- [ ] Leitura usa `ClientQueryRepository`; escrita continua em `ClientRepository`
- [ ] Testes existentes continuam passando
- [ ] Gate: `mvn test -pl lash-clients`

**Tests**: unit
**Gate**: quick

---

### T31: `lash-services` — Query `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**Onde**: `lash-services/domain/port/out/ServiceQueryRepository.java` + impl
**Depende de**: T17
**Requirement**: RBK-27

**Feito quando**:
- [ ] Mesmo padrão de T30
- [ ] Gate: `mvn test -pl lash-services`

**Tests**: unit
**Gate**: quick

---

### T32: `lash-appointments` — Query `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Separar leitura, avaliando se algumas queries (ex. grade semanal) merecem `Dao`+`RowMapper` em vez de `Specification`, dado o volume de `LEFT JOIN FETCH` já existente
**Onde**: `lash-appointments/domain/port/out/AppointmentQueryRepository.java` + impl (`Dao`/`RowMapper` onde aplicável)
**Depende de**: T17
**Requirement**: RBK-27

**Feito quando**:
- [ ] Queries de listagem/grade usam o repositório de leitura separado
- [ ] Gate: `mvn test -pl lash-appointments`

**Tests**: unit
**Gate**: quick

---

### T33: `lash-finance` — Query `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Formalizar `FinancialSummaryRepositoryImpl` já existente como o `Dao`/`RowMapper` de referência, documentando a convenção
**Onde**: `lash-finance/infrastructure/persistence/repository/FinancialSummaryRepositoryImpl.java` (já existe — menor esforço, principalmente documentação/nomenclatura)
**Depende de**: T17
**Requirement**: RBK-27, RBK-25

**Feito quando**:
- [ ] Nomenclatura alinhada com `CONVENTIONS.md` atualizado (T-doc, ver nota abaixo)
- [ ] Gate: `mvn test -pl lash-finance`

**Tests**: none (refatoração de nomenclatura sobre código já testado)
**Gate**: quick

---

### T34: `lash-stock` — Query `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**Onde**: `lash-stock/domain/port/out/InventoryQueryRepository.java` + impl
**Depende de**: T17
**Requirement**: RBK-27

**Feito quando**:
- [ ] Mesmo padrão de T30
- [ ] Gate: `mvn test -pl lash-stock`

**Tests**: unit
**Gate**: quick

---

### T35: `lash-fichas` — Query `[P]`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**Onde**: `lash-fichas/domain/port/out/FichaQueryRepository.java` + impl
**Depende de**: T17
**Requirement**: RBK-27

**Feito quando**:
- [ ] Mesmo padrão de T30
- [ ] Gate: `mvn test -pl lash-fichas`

**Tests**: unit
**Gate**: quick

---

### T36: `lash-dashboard` — Query (formalização)

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: ~~Dashboard já usa `EntityManager`/JPQL ad-hoc só de leitura — formalizar como `Dao`/`RowMapper`~~ — **revisado**: não existe `DashboardRepositoryImpl` no código atual (suposição desatualizada de uma versão anterior do projeto, antes da refatoração multi-módulo). `GetDashboardSummaryUseCaseImpl`/`GetTodayScheduleUseCaseImpl` já compõem leitura direto dos `JpaRepository`s de `lash-clients`/`lash-appointments`/`lash-finance`/`lash-stock`/`lash-services`, sem nenhuma porta de escrita própria do módulo — não há "leitura" para separar de "escrita" porque o módulo inteiro já é só leitura
**Onde**: N/A (nada a criar/mover)
**Depende de**: T17
**Requirement**: RBK-27, RBK-25

**Feito quando**:
- [x] Verificado que o módulo já satisfaz RBK-27 por construção (100% leitura, sem repositório de escrita)
- [ ] Gate: `mvn test -pl lash-dashboard`

**Tests**: none (refatoração de nomenclatura)
**Gate**: quick

---

## Documentação (transversal, sem dependência de código)

### TD01: Atualizar `CONVENTIONS.md`

**Status**: ✅ Implementado — verificação (gate) adiada para a validação final do projeto

**O que**: Documentar oficialmente os sufixos `*Command`, `*ApplicationService`, `*UseCase`, `*Dao`, `*RowMapper`, `*QueryResource` e quando usar cada um
**Onde**: `lash-docs/.specs/codebase/CONVENTIONS.md`
**Depende de**: T17 (padrão validado no piloto)
**Requirement**: RBK-25

**Feito quando**:
- [ ] Seção nova em `CONVENTIONS.md` com exemplos de cada sufixo, espelhando o piloto de `lash-core`

**Tests**: none
**Gate**: none (documentação)

---

## Parallel Execution Map

```
Fase A: sequencial (T01→T02→T03→T04→T05→T06→T07→T08)
Fase B: T09/T14 [P] independentes; resto sequencial (T09→T10→T11→T12→T13→T15→T16→T17)
Fase C: sequencial (T18→T19→T20→T21, T20/T21 podem rodar em paralelo entre si após T18/T17)
Fase D: T22 após T17, independente de Fase C
Fase E: T23–T29 todas [P] entre si, após T17
Fase F: T30–T36 todas [P] entre si, após T17 (T36 também depende de T29)
TD01: após T17
```

---

## Diagram-Definition Cross-Check

| Task | Depends On (corpo) | Diagrama mostra | Status |
|---|---|---|---|
| T01–T08 | (ver Fase A) | Sequência A | ✅ |
| T09 | Nenhum `[P]` | Paralelo à Fase A | ✅ |
| T10 | T09 | T09→T10 | ✅ |
| T11 | T07, T10 | Converge de T07 e T10 | ✅ |
| T12 | T06, T08, T09 | T08→T12, T09→T12 | ✅ |
| T13 | T12 | T12→T13 | ✅ |
| T14 | Nenhum `[P]` | Paralelo à Fase B | ✅ |
| T15 | T01, T11, T12 | T11→T15, T12→T15 | ✅ |
| T16 | T06 | Independente, antes do checkpoint | ✅ |
| T17 | T09–T16 | Checkpoint após Fase B | ✅ |
| T18 | T11 | Início da Fase C | ✅ |
| T19 | T18 | T18→T19 | ✅ |
| T20 | T17 | Depende do fluxo completo | ✅ |
| T21 | T17 | Depende do interceptor+audit completos | ✅ |
| T22 | T17 | Fase D após T17 | ✅ |
| T23–T29 | T17 (cada uma) | Todas partem de T17, paralelas entre si | ✅ |
| T30–T36 | T17 (cada uma); T36 também T29 | Partem de T17; T36 mostra dependência extra de T29 | ✅ |
| TD01 | T17 | Após T17 | ✅ |

---

## Test Co-location Validation

| Task | Camada criada/modificada | Tipo exigido | Task declara | Status |
|---|---|---|---|---|
| T01 | Mapper/Repository simples | unit | unit | ✅ |
| T02 | Resolver (lógica pura) | unit | unit | ✅ |
| T03 | Connection provider (JDBC real) | integration | integration | ✅ |
| T04 | JwtService (lógica pura) | unit | unit | ✅ |
| T05 | Filtro (contexto Spring + JWT real) | integration | integration | ✅ |
| T06 | Migration + entidade | none | none | ✅ |
| T07 | Wiring (só valida boot) | integration | integration | ✅ |
| T08 | Checkpoint | none | none | ✅ |
| T09 | Aspect (lógica + validação) | unit | unit | ✅ |
| T10 | Persistência + aspect (banco real) | integration | integration | ✅ |
| T11 | Provisionador (JDBC + Liquibase real) | integration | integration | ✅ |
| T12 | Command+AppService+UseCase (regra pura, repo mockável) | unit | unit | ✅ |
| T13 | Idem | unit | unit | ✅ |
| T14 | Template (visual) | none | none | ✅ |
| T15 | Orquestrador (mock parcial + fluxo real) | unit + integration | unit + integration | ✅ |
| T16 | UserDetailsService (lógica pura) | unit | unit | ✅ |
| T17 | Checkpoint | none | none | ✅ |
| T18 | Infra de teste | integration | integration | ✅ |
| T19 | Teste de isolamento (banco real) | integration | integration | ✅ |
| T20 | Teste e2e (banco real) | integration | integration | ✅ |
| T21 | Teste de aspect (banco real, audit log) | integration | integration | ✅ |
| T22 | UseCase (regra simples) | unit | unit | ✅ |
| T23–T29 | UseCase existente + Command/AppService novo (repo mockável) | unit (+ integration onde já existir ITest) | unit (+ integration) | ✅ |
| T30–T32, T34–T35 | Repositório de leitura novo (repo mockável nos UseCases que o consomem) | unit | unit | ✅ |
| T33, T36 | Refatoração de nomenclatura sobre código já testado | none | none | ✅ |
| TD01 | Documentação | none | none | ✅ |

---

## Task Granularity Check

| Grupo | Escopo | Status |
|---|---|---|
| T01–T22 | 1 arquivo/conceito coeso por task | ✅ Granular |
| T23–T36 | Intencionalmente grossa (1 módulo por task) — detalhamento fino ocorre ao iniciar cada módulo, replicando o padrão do piloto | ⚠️ Aceito — ver nota da Fase E/F |

---

## Ferramentas por Task — a confirmar com a usuária

Nenhum MCP externo identificado como necessário (tudo é código Java local + `mvn`/banco Docker já configurados). Skill `tlc-spec-driven` conduz a execução; skill `verify` recomendada ao final de cada fase (T08, T17, T20/T21, e ao final de cada task de Fase E/F) para rodar a aplicação real antes de marcar a task como concluída.
