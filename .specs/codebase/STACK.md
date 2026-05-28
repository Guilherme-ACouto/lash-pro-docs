# Stack Tecnológica — Lash Manager

**Atualizado:** 2026-05-22

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
| Flyway | 10.x | Migrations de banco (`flyway-core` + `flyway-database-postgresql`) |
| Lombok | 1.18.x | Redução de boilerplate |
| Spring Validation | 3.3.x | Jakarta Bean Validation |
| Spring Mail | 3.3.x | JavaMailSender para SMTP |
| Maven | 3.x | Build e dependências |

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
| Backend Spring Boot | 8080 |
| PostgreSQL | 5433 |

## Variáveis de Ambiente (Backend)

Todas com defaults em `application.yml` para desenvolvimento local:

| Variável | Default dev | Descrição |
|---|---|---|
| `DB_URL` | `jdbc:postgresql://localhost:5433/lashmanager` | URL JDBC |
| `DB_USER` / `DB_PASS` | `postgres` / `postgres` | Credenciais do banco |
| `JWT_SECRET` | valor hex hardcoded (64 bytes) | Chave de assinatura JWT — **mudar em produção** |
| `JWT_EXPIRATION` | `86400000` | Expiração access token (24h em ms) |
| `JWT_REFRESH_EXPIRATION` | `604800000` | Expiração refresh token (7d em ms) |
| `MAIL_HOST` / `MAIL_PORT` | `smtp.gmail.com` / `587` | SMTP (configurado, não usado ativamente) |
| `CORS_ORIGINS` | `http://localhost:4200` | Origens CORS permitidas |
