# Integrações Externas — Lash Manager

**Atualizado:** 2026-07-14

---

## Autenticação — JWT (JJWT 0.12.6)

**Propósito:** Tokens stateless de autenticação e renovação de sessão  
**Implementação:** `lash-core/infrastructure/security/JwtService.java` (implementa `TokenPort`)  
**Configuração:** `lash-app/src/main/resources/application.yml` — `app.jwt.*` (única config de runtime do sistema — os demais módulos só têm `application-test.yml`)

### Dois tipos de token

| Tipo | Claim `"type"` | Expiração default | Uso |
|---|---|---|---|
| Access Token | `"ACCESS"` | 24h (`86400000ms`) | Authorization header em cada request |
| Refresh Token | `"REFRESH"` | 7d (`604800000ms`) | Endpoint `/api/auth/refresh` |

### Fluxo

```
POST /api/auth/login → { accessToken, refreshToken }
POST /api/auth/refresh { refreshToken } → { accessToken, refreshToken }
```

### Configuração de variáveis

```yaml
app:
  jwt:
    secret: ${JWT_SECRET:404E635266...}  # ⚠️ default hardcoded — mudar em produção
    expiration: ${JWT_EXPIRATION:86400000}
    refresh-expiration: ${JWT_REFRESH_EXPIRATION:604800000}
```

---

## Banco de Dados — PostgreSQL + Spring Data JPA

**Propósito:** Persistência principal de todos os dados do sistema  
**Driver:** `org.postgresql.Driver` (PostgreSQL 42.x)  
**ORM:** Hibernate 6.x via Spring Data JPA  
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
      ddl-auto: validate       # schema validado contra Flyway, nunca gerado automaticamente
    show-sql: false
```

### Ambiente de desenvolvimento

- Container Docker: `docker start lashmanager-db`
- Porta: 5433 (não padrão 5432 — evitar conflito com PostgreSQL local)
- Banco: `lashmanager`
- Usuário/senha: `postgres`/`postgres`

---

## Migrations — Flyway (uma migration por módulo)

**Propósito:** Versionamento do schema do banco de dados  
**Localização:** `{modulo}/src/main/resources/db/migration/` — **cada módulo Maven tem sua própria pasta de migrations**, não existe mais um `db/migration/` central  
**Convenção de nomes:** `V{n}__{descricao_snake_case}.sql`  
**Convenção de numeração:** cada módulo recebe um range de 100 (`V100`–`V199` para lash-core, `V200`–`V299` para lash-clients, etc.) — uma migration incremental futura em um módulo usa o próximo número livre dentro do seu próprio range (ex.: `V201__add_column_x.sql` para uma alteração em lash-clients)

### Migrations existentes

| Módulo | Arquivo | Conteúdo |
|---|---|---|
| lash-core | `V100__create_users_table.sql` | Tabela `users` |
| lash-clients | `V200__create_clients_table.sql` | Tabela `clients` |
| lash-services | `V300__create_services_table.sql` | Tabela `services` |
| lash-appointments | `V400__create_appointments_table.sql` | Tabela `appointments` (FK clients, services) + constraint CHECK de status |
| lash-finance | `V500__create_financial_entries_table.sql` | Tabela `financial_entries` (FK appointments) |
| lash-stock | `V600__create_inventory_tables.sql` | Tabelas `inventory_items`, `inventory_movements` |
| lash-fichas | `V700__create_fichas_tables.sql` | Tabelas `fichas`, `lash_mappings` (FK clients) |
| lash-app | `V800__seed_data.sql` | Insere 1 admin (`admin@lashmanager.com`) e serviços padrão |

Todas as migrations dos 9 módulos ficam no classpath do `lash-app` (que depende de todos eles) e o Flyway as aplica em ordem de número de versão — não pela ordem de módulos no POM. `spring.flyway.locations: classpath:db/migration` (configurado só em `lash-app/application.yml`) resolve automaticamente para o `db/migration/` de todos os JARs no classpath.

### Regra crítica

**Nunca alterar uma migration já aplicada.** Sempre criar uma nova versão incremental, no range do módulo dono da tabela (ex.: `V201__...` para uma mudança futura em `clients`, não `V801__...`).

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
```

---

## SMTP — JavaMailSender

**Propósito:** Envio de e-mails (recuperação de senha)  
**Implementação:** Spring Mail (`spring-boot-starter-mail`)  
**Status:** Configurado — não utilizado ativamente (endpoints de password reset existem mas email não é disparado)

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

**Todos os demais endpoints** requerem `Authorization: Bearer <accessToken>`.

**Filtro JWT:** `JwtAuthFilter` (estende `OncePerRequestFilter`) valida o token antes de cada request autenticada.

---

## Sem integrações externas de terceiros

Atualmente o sistema não usa:
- Nenhum serviço de pagamento
- Nenhuma API de mensagens (WhatsApp, SMS) — previsto para Fase 3
- Nenhum storage externo (S3, Cloudinary)
- Nenhum serviço de monitoramento (Sentry, Datadog)
- Nenhum CDN
