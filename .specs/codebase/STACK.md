# Stack Tecnológica — Lash Manager

**Atualizado:** 2026-07-15

## Backend

| Tecnologia | Versão | Uso |
|---|---|---|
| Java | 21 (Zulu) | Linguagem principal |
| Spring Boot | 3.3.5 | Framework principal |
| Spring Security | 6.3.x | Autenticação e autorização |
| JJWT | 0.12.6 | Tokens JWT stateless (api + impl + jackson), agora com claim `tenantId` |
| Spring Data JPA | 3.3.x | Acesso a dados |
| Hibernate | 6.5.3 | ORM — configurado em **multi-tenant SCHEMA strategy** (ver ARCHITECTURE.md) |
| PostgreSQL Driver | 42.x | Driver JDBC |
| Flyway | 10.x | Migrations do schema `public` (`flyway-core` + `flyway-database-postgresql`), uma migration por módulo. Também dependência `test`-scope em `lash-core`/`lash-clients` (necessário pro banco de teste isolado rodar as próprias migrations) |
| **Liquibase** | 4.27.0 | Migrations dos **schemas de tenant** (provisionados sob demanda) — usado só programaticamente via API Java (`SchemaProvisionerImpl`), **não** via autoconfiguração do Spring Boot (`spring.liquibase.enabled: false`) |
| Spring AOP (`spring-boot-starter-aop`) | 3.3.5 | `CommandInterceptor` — `@Aspect` que valida/audita toda operação de escrita (`*ApplicationService.when(...)`) |
| Lombok | 1.18.x | Redução de boilerplate |
| Spring Validation | 3.3.x | Jakarta Bean Validation — usada tanto nos DTOs de Request quanto nos `Command` (ver CONVENTIONS.md) |
| Spring Mail | 3.3.x | JavaMailSender para SMTP — usado no "esqueci senha" e no e-mail de ativação de conta |
| Maven | 3.x | Build **multi-módulo** (Maven Reactor) — 1 POM pai + 9 módulos |

### Arquitetura de build — Maven multi-módulo

O backend é um **reactor Maven com 9 módulos**, coordenados pelo `pom.xml` raiz (`packaging=pom`):

```
lash-backend/ (pom pai)
├── lash-core            ← User, Tenant, Auth, JWT, Command/ApplicationService/Interceptor,
│                           multi-tenancy, exceções base, infra compartilhada
├── lash-clients         ← depende de: lash-core
├── lash-services        ← depende de: lash-core
├── lash-appointments     ← depende de: lash-core, lash-clients, lash-services
├── lash-finance          ← depende de: lash-core, lash-appointments
├── lash-stock            ← depende de: lash-core, lash-finance
├── lash-fichas           ← depende de: lash-core, lash-clients
├── lash-dashboard        ← depende de: lash-core, lash-clients, lash-appointments, lash-finance, lash-stock
└── lash-app              ← módulo executável — depende de todos os demais
```

Cada módulo é um artefato Maven independente (`com.lashmanager:lash-{modulo}`), versionado junto (`${project.version}`) e gerenciado via `<dependencyManagement>` no POM pai. Só `lash-app` tem `spring-boot-maven-plugin` e classe `main` — é o único artefato executável (fat JAR); os demais empacotam como JAR comum e são consumidos como dependência.

**Comandos de build relevantes:**

```bash
mvn install -DskipTests    # instala todos os módulos no .m2 local (necessário após pull com mudança de módulo)
mvn compile                # compila o reactor inteiro
mvn test                   # roda testes de todos os módulos
mvn test -pl lash-clients  # roda testes de um módulo específico
mvn spring-boot:run -pl lash-app  # sobe a aplicação (ou `cd lash-app && mvn spring-boot:run`)
```

## Frontend

| Tecnologia | Versão | Uso |
|---|---|---|
| Angular | 18.2.0 | Framework principal |
| TypeScript | 5.4.0 | Linguagem (strict mode ativo) |
| Angular Material | 18.2.0 | Componentes UI (Material Design 3) |
| Angular CDK | 18.2.0 | Utilitários de componente |
| NgRx Store | 18.0.2 | Estado global centralizado |
| NgRx Effects | 18.0.2 | Side effects assíncronos |
| NgRx Entity | 18.0.2 | Coleções no store |
| NgRx DevTools | 18.0.2 | Debug no browser |
| RxJS | 7.8.x | Programação reativa |
| Zone.js | 0.14.x | Change detection |
| Karma + Jasmine | 6.4 / 5.2 | Testes unitários (configurado, sem uso atual) |

> O frontend **ainda não tem** telas para o fluxo de registro/ativação/multi-tenancy nem para o endpoint admin de tenants — só o backend foi implementado até agora (ver ROADMAP.md/STATE.md).

## Banco de Dados

| Tecnologia | Versão | Uso |
|---|---|---|
| PostgreSQL | 15 | Banco de dados principal |
| Docker | - | Container local (`lashmanager-db`, porta 5433) |

**Dois bancos lógicos na mesma instância Postgres:**
- `lashmanager` — banco de desenvolvimento/produção
- `lashmanager_test` — banco separado só para os testes de integração (introduzido junto da multi-tenancy, replicando o padrão de banco de teste isolado de referência — ver TESTING.md)

**Schema físico por tenant:** dentro de qualquer um dos dois bancos, o schema `public` guarda as tabelas compartilhadas (`users`, `tenants`, `command_audit_log`) e cada tenant ativo ganha seu próprio schema Postgres (`tenant_<uuid-sem-hifen>`) com as tabelas de negócio replicadas (clientes, serviços, agendamentos, financeiro, estoque, fichas) — provisionado sob demanda na ativação de conta, não mais um schema único compartilhado por todos. Ver ARCHITECTURE.md para o fluxo completo.

## Ambiente de Desenvolvimento

| Ferramenta | Uso |
|---|---|
| SDKMAN | Gerenciamento de versões Java |
| NVM | Gerenciamento de versões Node |
| Docker Desktop | Container local do PostgreSQL |

## Portas Locais

| Serviço | Porta |
|---|---|
| Frontend Angular | 4200 |
| Backend Spring Boot (lash-app) | 8080 |
| PostgreSQL | 5433 |

## Variáveis de Ambiente (Backend)

Todas com defaults em `lash-app/src/main/resources/application.yml` para desenvolvimento local (único módulo com `application.yml` de runtime — os demais só têm `application-test.yml` para testes de integração):

| Variável | Default dev | Descrição |
|---|---|---|
| `DB_URL` | `jdbc:postgresql://localhost:5433/lashmanager` | URL JDBC |
| `DB_USER` / `DB_PASS` | `postgres` / `postgres` | Credenciais do banco |
| `JWT_SECRET` | valor hex hardcoded (64 bytes) | Chave de assinatura JWT — **mudar em produção** |
| `JWT_EXPIRATION` | `86400000` | Expiração access token (24h em ms) |
| `JWT_REFRESH_EXPIRATION` | `604800000` | Expiração refresh token (7d em ms) |
| `MAIL_HOST` / `MAIL_PORT` | `smtp.gmail.com` / `587` | SMTP — usado no "esqueci senha" e no e-mail de ativação de conta |
| `CORS_ORIGINS` | `http://localhost:4200` | Origens CORS permitidas |
| `TENANT_SCHEMA_PREFIX` | `tenant_` | Prefixo do nome do schema Postgres derivado do `tenantId` de cada tenant |
| `ACTIVATION_KEY_HOURS` | `48` | Validade (em horas) do link de ativação de conta enviado por e-mail |

## Configuração de Flyway/Liquibase (`lash-app/application.yml`)

Duas flags importantes, adicionadas junto da multi-tenancy — sem elas o boot falha (ver CONCERNS.md/histórico de bugs em STATE.md):

```yaml
spring:
  flyway:
    out-of-order: true    # migrations novas do core (V101-103) têm número menor que
                           # migrations de domínio já aplicadas (V200+) — ranges reservados
                           # por módulo desde a refatoração multi-módulo (ver ARCHITECTURE.md)
  liquibase:
    enabled: false         # Liquibase só é usado programaticamente (SchemaProvisionerImpl);
                           # sem essa flag o Spring Boot tenta rodar sua própria autoconfiguração
                           # e falha procurando um changelog padrão que não existe
```
