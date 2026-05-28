# Integrações Externas — Lash Manager

**Atualizado:** 2026-05-22

---

## Autenticação — JWT (JJWT 0.12.6)

**Propósito:** Tokens stateless de autenticação e renovação de sessão  
**Implementação:** `infrastructure/security/JwtService.java` (implementa `TokenPort`)  
**Configuração:** `application.yml` — `app.jwt.*`

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
**Config:** `application.yml` — `spring.datasource.*` e `spring.jpa.*`

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

## Migrations — Flyway

**Propósito:** Versionamento do schema do banco de dados  
**Localização:** `src/main/resources/db/migration/`  
**Convenção de nomes:** `V{n}__{descricao_snake_case}.sql`

### Migrations existentes

| Arquivo | Conteúdo |
|---|---|
| `V1__create_schema.sql` | Cria 8 tabelas: `users`, `clients`, `services`, `appointments`, `financial_entries`, `inventory_items`, `inventory_movements` + índices + constraints CHECK |
| `V2__seed_data.sql` | Insere 1 admin (`admin@lashmanager.com`) e 4 serviços padrão |

### Regra crítica

**Nunca alterar uma migration já aplicada.** Sempre criar uma nova versão incremental (`V3__...`).

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
**Implementação:** `SecurityConfig.java`  
**Configuração:** `app.cors.allowed-origins` em `application.yml`

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
