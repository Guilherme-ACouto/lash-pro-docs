# Integrações Externas — Lash Manager

**Atualizado:** 2026-07-15

---

## Autenticação — JWT (JJWT 0.12.6)

**Propósito:** Tokens stateless de autenticação e renovação de sessão, agora carregando o tenant do usuário
**Implementação:** `lash-core/infrastructure/security/JwtService.java` (implementa `TokenPort`)
**Configuração:** `lash-app/src/main/resources/application.yml` — `app.jwt.*` (única config de runtime do sistema — os demais módulos só têm `application-test.yml`)

### Dois tipos de token

| Tipo | Claim `"type"` | Expiração default | Uso |
|---|---|---|---|
| Access Token | `"ACCESS"` | 24h (`86400000ms`) | Authorization header em cada request. Carrega também o claim `"tenantId"` (nulo pro usuário de plataforma/dev, sem tenant) |
| Refresh Token | `"REFRESH"` | 7d (`604800000ms`) | Endpoint `/api/auth/refresh` |

### Fluxo

```
POST /api/auth/login → { accessToken, refreshToken }
POST /api/auth/refresh { refreshToken } → { accessToken, refreshToken }
```

Cada requisição autenticada passa primeiro pelo `TenantResolvingFilter` (extrai `tenantId` do token, popula `TenantContext`) e só depois pelo `JwtAuthFilter` (valida token, popula `SecurityContextHolder`) — ver ARCHITECTURE.md pro fluxo completo.

### Configuração de variáveis

```yaml
app:
  jwt:
    secret: ${JWT_SECRET:404E635266...}  # ⚠️ default hardcoded — mudar em produção
    expiration: ${JWT_EXPIRATION:86400000}
    refresh-expiration: ${JWT_REFRESH_EXPIRATION:604800000}
  tenant:
    schema-prefix: ${TENANT_SCHEMA_PREFIX:tenant_}
  activation:
    key-expiration-hours: ${ACTIVATION_KEY_HOURS:48}
```

---

## Banco de Dados — PostgreSQL + Spring Data JPA (multi-tenant)

**Propósito:** Persistência principal de todos os dados do sistema, agora isolado por schema Postgres por tenant
**Driver:** `org.postgresql.Driver` (PostgreSQL 42.x)
**ORM:** Hibernate 6.5.3 via Spring Data JPA — configurado em **`MultiTenancyStrategy.SCHEMA`**
**Config:** `lash-app/src/main/resources/application.yml` — `spring.datasource.*` e `spring.jpa.*`

### Conexão

```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5433/lashmanager}
    username: ${DB_USER:postgres}
    password: ${DB_PASS:postgres}
  jpa:
    hibernate:
      ddl-auto: validate       # schema validado contra Flyway (schema public) — no boot, com TenantContext
                                # vazio, valida contra "public" (default do resolver)
    show-sql: false
```

### Wiring do multi-tenant (Hibernate)

`HibernateMultiTenantConfig` (`lash-core/infrastructure/multitenancy/`) registra, via `HibernatePropertiesCustomizer`:
- `AvailableSettings.MULTI_TENANT_CONNECTION_PROVIDER` → `MultiTenantConnectionProviderImpl` (troca `search_path` por conexão obtida do pool)
- `AvailableSettings.MULTI_TENANT_IDENTIFIER_RESOLVER` → `CurrentTenantIdentifierResolverImpl` (lê `TenantContext`, default `"public"`)

**Nota de compatibilidade:** essas constantes ficam em `org.hibernate.cfg.MultiTenancySettings` no Hibernate 6.5.3 — a classe `MultiTenancyStrategy`/constante `AvailableSettings.MULTI_TENANT` (usada em versões antigas do Hibernate, vista em exemplos de outros projetos) **não existe** nessa versão. Confirmado decompilando o jar antes de escrever o código.

### Ambiente de desenvolvimento

- Container Docker: `docker start lashmanager-db`
- Porta: 5433 (não padrão 5432 — evitar conflito com PostgreSQL local)
- **Dois bancos lógicos na mesma instância:**
  - `lashmanager` — dev/produção
  - `lashmanager_test` — só para testes de integração (criar manualmente uma vez: `docker exec lashmanager-db psql -U postgres -c "CREATE DATABASE lashmanager_test;"`)
- Usuário/senha: `postgres`/`postgres`

---

## Migrations — Flyway (schema `public`) + Liquibase (schema de tenant)

Duas ferramentas de migração convivem no projeto, cada uma dona de um schema diferente:

### Flyway — schema `public` (uma migration por módulo, como antes)

**Localização:** `{modulo}/src/main/resources/db/migration/`
**Convenção de nomes:** `V{n}__{descricao_snake_case}.sql`
**Convenção de numeração:** cada módulo recebe um range de 100 (`V100`–`V199` para lash-core, `V200`–`V299` para lash-clients, etc.)

| Módulo | Arquivo | Conteúdo |
|---|---|---|
| lash-core | `V100__create_users_table.sql` | Tabela `users` |
| lash-core | `V101__create_tenants_table.sql` | Tabela `tenants` |
| lash-core | `V102__add_tenant_and_activation_to_users.sql` | `tenant_id`, `activation_key`, `activation_key_expiry` em `users` |
| lash-core | `V103__create_command_audit_log_table.sql` | Tabela `command_audit_log` (também existe via Liquibase nos schemas de tenant) |
| lash-clients | `V200__create_clients_table.sql` | Tabela `clients` |
| lash-services | `V300__create_services_table.sql` | Tabela `services` |
| lash-appointments | `V400__create_appointments_table.sql` | Tabela `appointments` (FK clients, services) + constraint CHECK de status |
| lash-finance | `V500__create_financial_entries_table.sql` | Tabela `financial_entries` (FK appointments) |
| lash-stock | `V600__create_inventory_tables.sql` | Tabelas `inventory_items`, `inventory_movements` |
| lash-fichas | `V700__create_fichas_tables.sql` | Tabelas `fichas`, `lash_mappings` (FK clients) |
| lash-app | `V800__seed_data.sql` | Insere 1 admin (`admin@lashmanager.com`, sem tenant) e serviços padrão |

**Flag necessária** (`lash-app/application.yml` e `application-test.yml`): `spring.flyway.out-of-order: true` — `V101`–`V103` têm número menor que migrations de domínio já aplicadas (`V200`+), o que o Flyway rejeita por padrão sem essa flag.

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    out-of-order: true
```

### Liquibase — schema de cada tenant (novo, provisionado sob demanda)

**Propósito:** aplicar as tabelas de negócio (clients, services, appointments, financial_entries, inventory, fichas) num schema Postgres novo, criado quando uma conta é ativada
**Implementação:** `SchemaProvisionerImpl` (`lash-core/infrastructure/multitenancy/`) — API Java do Liquibase (`liquibase.Liquibase`), **não** a autoconfiguração do Spring Boot
**Changelog raiz:** `lash-core/src/main/resources/db/changelog/master.xml`, agregando um changelog por módulo (cada um reaproveitando a migration Flyway já existente via `<sqlFile>`) + o fragmento `command-audit-log.xml`

```yaml
spring:
  liquibase:
    enabled: false   # essencial — sem isso o Spring Boot autoconfigura seu próprio SpringLiquibase
                      # e falha procurando um changelog padrão (classpath:/db/changelog/db.changelog-master.yaml)
                      # que não existe; o Liquibase real só roda via SchemaProvisionerImpl
```

**Conexão usada pelo Liquibase:** própria (`DriverManager`, não o pool/transação gerenciados pelo Spring) — permite que uma falha no provisionamento não deixe uma transação JPA presa numa conexão que já rodou DDL. Idempotente (`CREATE SCHEMA IF NOT EXISTS` + changesets já aplicados não rodam de novo).

**Dependência Maven:** `liquibase-core` (`lash-core/pom.xml`, sem versão explícita — herda do BOM do `spring-boot-starter-parent`, resolvido em 4.27.0). Pacote real do `JdbcConnection` nessa versão: `liquibase.database.jvm.JdbcConnection` (não `liquibase.database.jdbc`, que só existe em versões mais novas — confirmado decompilando o jar).

### Regra crítica (vale pras duas ferramentas)

**Nunca alterar uma migration/changeset já aplicado.** Sempre criar uma nova versão incremental, no range do módulo dono da tabela.

---

## SMTP — JavaMailSender

**Propósito:** Envio de e-mails — recuperação de senha **e agora também ativação de conta**
**Implementação:** Spring Mail (`spring-boot-starter-mail`), `EmailPortImpl` em `lash-core/infrastructure/email/`
**Status:** Configurado, texto simples (`SimpleMailMessage`) — **não** usa Thymeleaf (decisão deliberada: o projeto nunca teve esse motor de template, não valia introduzir um só pro e-mail de ativação)

### Dois métodos no `EmailPort`

| Método | Uso | Link gerado |
|---|---|---|
| `sendPasswordResetEmail` | Fluxo de "esqueci senha" | `{frontendUrl}/auth/reset-password?token=...` |
| `sendActivationEmail` | Fluxo de registro/ativação de conta | `{frontendUrl}/auth/activation?key=...` |

Falha de envio (ex.: sem servidor SMTP configurado em dev/teste) é capturada e logada como `WARN` — nunca derruba o fluxo (o link também é logado em `INFO`, útil pra testar manualmente sem SMTP real).

### Configuração

```yaml
spring:
  mail:
    host: ${MAIL_HOST:smtp.gmail.com}
    port: ${MAIL_PORT:587}
    username: ${MAIL_USER:}
    password: ${MAIL_PASS:}
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
```

Para produção: configurar variáveis `MAIL_USER` e `MAIL_PASS` com conta Gmail ou serviço transacional (SendGrid, Resend, etc.).

---

## Proxy Angular → Backend

**Propósito:** Redirecionar requests `/api/*` do frontend (porta 4200) para o backend (porta 8080)
**Arquivo:** `lash-frontend/proxy.conf.json`

```json
{
  "/api": {
    "target": "http://localhost:8080",
    "secure": false
  }
}
```

**Como funciona:** Em desenvolvimento, o Angular DevServer intercepta qualquer request para `/api/*` e faz proxy para `http://localhost:8080/api/*`. Em produção, um reverse proxy (nginx, etc.) faz o mesmo papel.

O `angular.json` referencia o proxy config no target `serve`:
```json
"proxyConfig": "proxy.conf.json"
```

---

## CORS — Spring Security

**Propósito:** Permitir que o frontend (porta 4200) acesse o backend (porta 8080)
**Implementação:** `lash-core/infrastructure/security/SecurityConfig.java`
**Configuração:** `app.cors.allowed-origins` em `lash-app/src/main/resources/application.yml`

```yaml
app:
  cors:
    allowed-origins: ${CORS_ORIGINS:http://localhost:4200}
```

Em produção, definir `CORS_ORIGINS` com o domínio real do frontend.

---

## Spring Security — Configuração

**Endpoints públicos (sem autenticação):**
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/forgot-password`
- `POST /api/auth/reset-password`
- `POST /api/register`
- `POST /api/register/resend`
- `GET`/`POST /api/activation`

**Todos os demais endpoints** requerem `Authorization: Bearer <accessToken>`, incluindo `/api/admin/tenants` (que também exige que o usuário autenticado não tenha `tenantId` — checagem feita em `PlatformAdminChecker`, na camada de aplicação, não no `SecurityConfig`).

**Filtros, em ordem:** `TenantResolvingFilter` (resolve tenant do JWT) → `JwtAuthFilter` (valida token, autentica) → `UsernamePasswordAuthenticationFilter`.

---

## Sem integrações externas de terceiros

Atualmente o sistema não usa:
- Nenhum serviço de pagamento
- Nenhuma API de mensagens (WhatsApp, SMS) — previsto para Fase 3
- Nenhum storage externo (S3, Cloudinary)
- Nenhum serviço de monitoramento (Sentry, Datadog)
- Nenhum CDN
