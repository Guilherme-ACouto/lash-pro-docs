# Stack Tecnológica — Lash Manager

**Atualizado:** 2026-07-14

## Backend

| Tecnologia | Versão | Uso |
|---|---|---|
| Java | 21 (Zulu) | Linguagem principal |
| Spring Boot | 3.3.5 | Framework principal |
| Spring Security | 6.3.x | Autenticação e autorização |
| JJWT | 0.12.6 | Tokens JWT stateless (api + impl + jackson) |
| Spring Data JPA | 3.3.x | Acesso a dados |
| Hibernate | 6.x | ORM |
| PostgreSQL Driver | 42.x | Driver JDBC |
| Flyway | 10.x | Migrations de banco (`flyway-core` + `flyway-database-postgresql`), uma migration por módulo |
| Lombok | 1.18.x | Redução de boilerplate |
| Spring Validation | 3.3.x | Jakarta Bean Validation |
| Spring Mail | 3.3.x | JavaMailSender para SMTP |
| Maven | 3.x | Build **multi-módulo** (Maven Reactor) — 1 POM pai + 9 módulos |

### Arquitetura de build — Maven multi-módulo

O backend deixou de ser um módulo único e passou a ser um **reactor Maven com 9 módulos**, coordenados pelo `pom.xml` raiz (`packaging=pom`):

```
lash-backend/ (pom pai)
├── lash-core            ← User, Auth, JWT, exceções base, infra compartilhada
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

## Banco de Dados

| Tecnologia | Versão | Uso |
|---|---|---|
| PostgreSQL | 15 | Banco de dados principal |
| Docker | - | Container local (`lashmanager-db`, porta 5433) |

Um único banco físico compartilhado por todos os módulos — não há separação de schema por módulo, apenas separação lógica via ranges de versão do Flyway (ver ARCHITECTURE.md).

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
| `MAIL_HOST` / `MAIL_PORT` | `smtp.gmail.com` / `587` | SMTP (configurado, não usado ativamente) |
| `CORS_ORIGINS` | `http://localhost:4200` | Origens CORS permitidas |
