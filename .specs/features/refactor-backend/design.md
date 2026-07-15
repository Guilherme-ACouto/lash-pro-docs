# Refatoração Backend → Padrão Pontta — Design

**Spec**: `.specs/features/refactor-backend/spec.md`

---

## Visão Geral da Arquitetura

```
                     ┌─────────────────────────────────────┐
                     │        Schema "public"                │
                     │  ┌───────────┐   ┌──────────────┐   │
                     │  │  tenants  │   │    users      │   │
                     │  └───────────┘   └──────────────┘   │
                     └─────────────────────────────────────┘

                     ┌─────────────────────────────────────┐
                     │   Schema "tenant_<uuid-sem-hifen>"    │
                     │  clients, services, appointments,     │
                     │  financial_entries, inventory_*,      │
                     │  fichas ... (todas as tabelas de       │
                     │  negócio já existentes)               │
                     └─────────────────────────────────────┘
                     (um schema desses por tenant ativo)
```

Login resolve o tenant automaticamente pelo e-mail (schema `public` é o default quando ainda não há tenant no contexto). A partir da autenticação, toda query de negócio roda com `search_path` apontando para o schema do tenant do usuário logado — nenhuma tabela de negócio ganha coluna `tenant_id`; o isolamento é 100% estrutural (schema), replicando a decisão da Pontta.

---

## `lash-core` — Novas Peças

```
lash-core/src/main/java/com/lashmanager/core/
├── domain/
│   ├── model/
│   │   └── Tenant.java                          (id, name, schemaName, active, createdAt)
│   └── port/
│       ├── in/
│       │   ├── RegisterUseCase.java
│       │   ├── ResendActivationUseCase.java
│       │   └── ActivateAccountUseCase.java
│       └── out/
│           ├── TenantRepository.java
│           ├── SchemaProvisionerPort.java
│           └── CommandAuditLogRepository.java
├── application/
│   ├── command/
│   │   ├── RegisterCommand.java
│   │   ├── ResendActivationCommand.java
│   │   └── ActivateAccountCommand.java
│   ├── service/
│   │   ├── RegisterApplicationService.java        (when(RegisterCommand))
│   │   ├── ResendActivationApplicationService.java
│   │   └── ActivateAccountApplicationService.java
│   └── usecase/
│       ├── RegisterUseCaseImpl.java               (regra de negócio pura, chamada pelo ApplicationService)
│       ├── ResendActivationUseCaseImpl.java
│       └── ActivateAccountUseCaseImpl.java
└── infrastructure/
    ├── command/
    │   ├── AbstractCommand.java                    (base: id, timestamp; suporte a @Valid)
    │   └── CommandInterceptor.java                 (@Aspect: valida, loga, audita; ponto de extensão @CommandPermission)
    ├── multitenancy/
    │   ├── TenantContext.java                   (ThreadLocal<String> schemaName)
    │   ├── CurrentTenantIdentifierResolverImpl.java
    │   ├── MultiTenantConnectionProviderImpl.java
    │   └── SchemaProvisionerImpl.java            (CREATE SCHEMA + Liquibase, conexão JDBC própria)
    ├── security/
    │   ├── JwtService.java                       (modificado: claim tenantId)
    │   └── TenantResolvingFilter.java             (novo: lê claim do JWT, popula TenantContext)
    ├── persistence/
    │   ├── TenantEntity.java / TenantJpaRepository.java / TenantMapper.java / TenantRepositoryImpl.java
    │   ├── CommandAuditLogEntity.java / CommandAuditLogJpaRepository.java
    │   └── UserEntity.java                        (modificado: + tenantId, + activationKey, + activationKeyExpiry)
    └── email/
        (e-mail de ativação implementado em EmailPortImpl.sendActivationEmail — texto simples via
         SimpleMailMessage, igual ao "esqueci senha"; o projeto não usa Thymeleaf, então não foi
         introduzido só para isso — ver correção abaixo)
```

---

## Command + ApplicationService + CommandInterceptor — Padrão Piloto

Réplica do fluxo de escrita da Pontta, adaptado ao Lash:

```
Controller → Command (DTO validável) → ApplicationService.when(command) → UseCase (regra de negócio) → Repository
                                              ↑
                                    CommandInterceptor (@Aspect)
                                    intercepta todo when(...) por convenção de assinatura
```

**`AbstractCommand`** (base, sem herança de `DomainEvent` — o Lash não tem multi-tenant carimbado no Command, o tenant já está resolvido na conexão via `TenantContext`/Hibernate; ver seção acima):

```java
public abstract class AbstractCommand {
    private final UUID commandId = UUID.randomUUID();
    private final LocalDateTime issuedAt = LocalDateTime.now();
}
```

**Exemplo — `RegisterCommand`/`RegisterApplicationService`** (piloto desta spec):

```java
@Getter @NoArgsConstructor @AllArgsConstructor
public class RegisterCommand extends AbstractCommand {
    @NotBlank private String name;
    @Email @NotBlank private String email;
    @NotBlank private String password;
}

@Service @RequiredArgsConstructor
public class RegisterApplicationService {
    private final RegisterUseCase registerUseCase;

    public RegisterResult when(@Valid RegisterCommand command) {
        return registerUseCase.execute(command.toDomainCommand());
    }
}
```

`RegisterUseCaseImpl` mantém exatamente a mesma responsabilidade que já tem hoje (regra de negócio) — só passa a ser chamado pelo `ApplicationService`, não direto pelo Controller.

**`CommandInterceptor`** — `@Aspect` que intercepta `execution(* com.lashmanager..*.application.service..*.when(..))`:

1. Roda Bean Validation manual no `Command` recebido (equivalente ao `@Valid` que hoje só existe no Controller — agora cobre qualquer chamador, não só HTTP)
2. Loga início/fim da execução (classe do Command, tempo decorrido)
3. Grava uma linha em `command_audit_log` (schema do tenant atual): classe do Command, payload serializado, `userId` (do `SecurityContext`), timestamp, sucesso/falha
4. **Ponto de extensão preparado, não implementado nesta spec**: leitura de uma futura anotação `@CommandPermission(role)` no Command, checando contra `UserRole` do usuário autenticado — hoje o Lash só tem um papel, então o hook fica pronto mas sem regra ativa (RBK-D04)

```java
@Aspect @Component
public class CommandInterceptor {
    @Around("execution(* com.lashmanager..*.application.service.*ApplicationService.when(..))")
    public Object intercept(ProceedingJoinPoint pjp) throws Throwable {
        AbstractCommand command = (AbstractCommand) pjp.getArgs()[0];
        validate(command);
        Object result;
        try {
            result = pjp.proceed();
            auditLog.record(command, currentUserId(), true);
        } catch (Exception e) {
            auditLog.record(command, currentUserId(), false);
            throw e;
        }
        return result;
    }
}
```

**Migração dos 42 UseCases existentes (RBK-26) e separação de leitura (RBK-27):** granularidade fina fica para depois que este piloto rodar em produção real (mesmo que só com o tenant de desenvolvimento) — a expectativa é que cada módulo vire um conjunto pequeno de tasks (`{Modulo}Command`s + `{Modulo}ApplicationService`s substituindo os UseCases como ponto de entrada, UseCases existentes praticamente inalterados por dentro). Módulos candidatos a ir primeiro: `lash-finance` (já tem separação de leitura parcial) e `lash-clients` (já tem suíte de testes, menor risco de regressão silenciosa).

---

## Resolução de Tenant — Fluxo por Requisição

1. `TenantResolvingFilter` (novo `OncePerRequestFilter`, roda **antes** do filtro de autenticação JWT existente) extrai o `tenantId` do token, se presente, e chama `TenantContext.setCurrentTenant(schemaName)`.
2. Se não houver token ainda (login, registro, ativação), `TenantContext` permanece vazio → `CurrentTenantIdentifierResolverImpl.resolveCurrentTenantIdentifier()` retorna `"public"` (default).
3. Hibernate usa esse identificador para pedir a conexão certa ao `MultiTenantConnectionProviderImpl`.
4. `MultiTenantConnectionProviderImpl.getConnection(tenantIdentifier)`: pega conexão do pool (HikariCP, já existente) e executa `SET search_path TO "<tenantIdentifier>", public` antes de devolvê-la.
5. `TenantContext.clear()` no `finally` do filtro — evita vazamento de tenant entre requisições (pool de threads reaproveitadas).

**Por que filtro e não Spring Security `AuthenticationSuccessHandler`:** o tenant precisa estar resolvido para **qualquer** request autenticado, não só no momento do login — o filtro roda em toda requisição, lendo o claim do JWT já validado.

---

## Login — Sem Mudança de Contrato HTTP

`UserDetailsServiceImpl.loadUserByUsername(email)` continua buscando só por e-mail — funciona porque, nesse momento, `search_path` ainda é `public` (nenhum JWT foi processado ainda nessa requisição). Ao autenticar com sucesso, `JwtService.generateToken(user)` passa a incluir `tenantId` (via `user.getTenantId()`) no payload — é esse claim que o `TenantResolvingFilter` vai ler em requisições futuras.

---

## Fluxo de Registro → Ativação → Provisionamento

```
POST /api/register
  → RegisterUseCaseImpl
      → valida e-mail (existsByEmail)
          → se existe e active=true  → 409 email já em uso
          → se existe e active=false → delega para ResendActivationUseCaseImpl (RBK-10)
          → se não existe            → cria UserEntity (schema public), active=false,
                                        activationKey=UUID aleatório,
                                        activationKeyExpiry=now + ACTIVATION_KEY_HOURS
      → envia e-mail (template activation.html) com link {baseUrl}/activation?key={key}

GET/POST /api/activation?key=xxx
  → ActivateAccountUseCaseImpl
      1. busca user por activationKey
          → não encontrado                        → 404 "link inválido"
          → encontrado mas activationKeyExpiry < now → 410 "link expirado" (usuário deve pedir reenvio)
      2. dentro de @Transactional (Spring/JPA):
          a. cria Tenant (id determinístico = novo UUID gerado para o tenant)
          b. ativa o usuário (active=true, limpa activationKey/Expiry, seta tenantId)
      3. FORA da transação Spring — conexão JDBC própria via SchemaProvisionerImpl:
          a. CREATE SCHEMA IF NOT EXISTS "tenant_<id-sem-hifen>"
          b. Liquibase.update() aplicando o changelog completo nesse schema
      4. se o passo 3 falhar: exceção sobe, passo 2 sofre rollback (Tenant e ativação desfeitos);
         schema já criado/parcialmente migrado permanece (idempotente — próxima tentativa com a
         mesma key, se ainda não expirada, refaz o processo e completa, sem duplicar nada)

POST /api/register/resend
  → ResendActivationUseCaseImpl
      → busca user por email; se !active, gera novo activationKey/Expiry, reenvia e-mail
      → se active=true ou não encontrado → resposta genérica (evita enumeração de e-mails cadastrados)
```

**Nota de segurança:** a resposta de `/register/resend` para e-mail inexistente ou já ativo deve ser idêntica à de sucesso (mensagem genérica "se o e-mail existir e estiver pendente, um novo link foi enviado") — evita que o endpoint seja usado para descobrir quais e-mails já têm conta.

---

## Migrations

- **Schema `public`**: continua via Flyway, em `lash-core/src/main/resources/db/migration/` — nova migration para `tenants` e alteração de `users` (colunas `tenant_id`, `activation_key`, `activation_key_expiry`).
- **Schema de tenant**: Liquibase, changelog único em `lash-core/src/main/resources/db/changelog/` contendo — na primeira versão — a consolidação de todas as migrations Flyway hoje existentes por módulo (V200+ de clients, V300+ de services, etc.), convertidas para cada respectivo módulo continuar dono do seu changelog, agregados via um `master.xml` que o `SchemaProvisionerImpl` executa contra o schema novo.
- Módulos de domínio (`lash-clients`, `lash-services`, etc.) passam a ter changelogs Liquibase em vez de migrations Flyway — mudança de ferramenta, não de dono/conteúdo das tabelas.

---

## Testes de Integração

`AbstractIntegrationTest` (hoje aponta pro schema único de teste) passa a:
1. Antes da suíte: criar um schema de teste (`test_tenant`), rodar o changelog Liquibase nele.
2. Popular `TenantContext` com esse schema antes de cada teste (substitui a necessidade de passar por login real).
3. Depois da suíte: `DROP SCHEMA test_tenant CASCADE` (limpeza).

Testes novos (`RBK-18`) cobrem: dois tenants distintos, mesma tabela lógica, dado de um não aparece na query do outro; e o fluxo HTTP completo registro → ativação → login no schema recém-criado.

---

## Configuração (`application.yml`)

```yaml
app:
  activation:
    key-expiration-hours: ${ACTIVATION_KEY_HOURS:48}
  tenant:
    schema-prefix: ${TENANT_SCHEMA_PREFIX:tenant_}
```

---

## Decisões de Design

| Decisão | Razão |
|---|---|
| Filtro dedicado (`TenantResolvingFilter`) separado do filtro de autenticação JWT existente | Mantém responsabilidade única; o filtro de auth continua só validando token/carregando `UserDetails` |
| Tenant `id` vira o nome do schema (prefixado, sem hífen) | Evita colisão de nome e mantém determinismo/idempotência igual ao design da Pontta |
| `SchemaProvisionerImpl` usa conexão JDBC própria (`DriverManager`, fora do pool gerenciado pelo Spring) | Isola o `CREATE SCHEMA`/Liquibase do controle transacional do Hibernate, replicando o design validado da Pontta (ver RBK-15) |
| `activationKeyExpiry` e endpoint de reenvio desde o início | Corrige os dois gaps identificados na Pontta antes de replicar o padrão |
| `AbstractCommand` não herda de `DomainEvent`/não carrega `tenantId` | O Lash já resolve o tenant na camada de conexão (Hibernate `search_path`), diferente da Pontta que usa o Command pra carimbar isso — evita duplicar o mecanismo de resolução de tenant |
| `command_audit_log` implementado desde o piloto (não adiado) | Baixo custo, alto valor para um SaaS com múltiplos usuários por conta — auditoria "quem fez o quê" é exatamente o que falta hoje no Lash |
| `@CommandPermission` preparado mas não aplicado | O Lash só tem um `UserRole` hoje; implementar controle de permissão sem ter mais de um papel seria over-engineering (RBK-D04) |
| Piloto do padrão Command/ApplicationService em `lash-core` (Fase B), migração dos outros 7 módulos em fase própria depois de validado | Evita retrabalho (construir os UseCases novos já certo) sem bloquear a fundação multi-tenant esperando a migração completa de 42 arquivos |
| **Correção (implementação):** e-mail de ativação usa `SimpleMailMessage` (texto simples), não Thymeleaf | O projeto não tinha Thymeleaf como dependência — o "esqueci senha" já existente também é texto simples; reaproveitar o padrão real do código bateu mais com RBK-11 ("reaproveitar infra existente") do que introduzir um motor de template novo só para este e-mail |
| **Correção (implementação):** `tenantId` é gerado e persistido no `User` já no **registro** (`RegisterUseCaseImpl`), não na ativação | Necessário para idempotência real do retry: como `ActivateAccountUseCaseImpl.execute()` é `@Transactional` e o provisionamento de schema roda dentro dela (ver RBK-15), se o provisionamento falhar o rollback desfaz qualquer `tenantId` atribuído *durante* a ativação — gerando um `tenantId` novo a cada retry e um schema órfão por tentativa. Atribuir no registro (fora dessa transação) garante que o retry sempre resolva para o mesmo `tenantId`/schema |
| `liquibase.database.jvm.JdbcConnection` (não `liquibase.database.jdbc.JdbcConnection`) | Verificado decompilando o jar real do Liquibase 4.27.0 (gerenciado pelo BOM do Spring Boot 3.3.5) antes de escrever o código — o pacote `jdbc` só existe em versões mais recentes que a instalada; `jvm` é o correto para 4.27.0 |
