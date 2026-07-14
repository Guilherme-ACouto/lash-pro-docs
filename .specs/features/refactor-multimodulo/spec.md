# Refatoração Multi-Módulo Maven — Spec

**Status**: Draft  
**Data**: 2026-07-13

---

## Contexto

O backend atual é um único módulo Maven (`lash-backend`) com toda a lógica em um único `src/`. À medida que o sistema cresce (financeiro, estoque, fichas, vendas, compras), manter tudo junto dificulta:
- Testes co-localizados com seu módulo
- Compilação isolada (mudar finanças recompila tudo)
- Fronteiras de negócio claras
- Eventual extração de microserviços se necessário

## Objetivo

Reorganizar o backend em múltiplos módulos Maven mantendo a arquitetura hexagonal intacta dentro de cada módulo. Testes ficam dentro do respectivo módulo.

## Módulos Alvo

| Módulo Maven | Domínio | Pacote |
|---|---|---|
| `lash-core` | Infra compartilhada (JWT, Security, Email, exceções base) | `com.lashmanager.core` |
| `lash-clients` | Gestão de clientes | `com.lashmanager.clients` |
| `lash-services` | Catálogo de serviços oferecidos | `com.lashmanager.services` |
| `lash-appointments` | Agendamentos | `com.lashmanager.appointments` |
| `lash-finance` | Financeiro (contas a pagar/receber) | `com.lashmanager.finance` |
| `lash-stock` | Estoque e movimentações | `com.lashmanager.stock` |
| `lash-fichas` | Anamneses e mapping de cílios | `com.lashmanager.fichas` |
| `lash-dashboard` | Dashboard com indicadores | `com.lashmanager.dashboard` |
| `lash-app` | Entry point Spring Boot, wiring final | `com.lashmanager.app` |

## Migrations

Cada módulo gerencia suas próprias migrations dentro de um range de versão exclusivo para evitar colisões no Flyway:

| Módulo | Range de versão | Prefixo na pasta |
|---|---|---|
| lash-core (auth/schema base) | V100–V199 | `db/migration/core/` |
| lash-clients | V200–V299 | `db/migration/clients/` |
| lash-services | V300–V399 | `db/migration/services/` |
| lash-appointments | V400–V499 | `db/migration/appointments/` |
| lash-finance | V500–V599 | `db/migration/finance/` |
| lash-stock | V600–V699 | `db/migration/stock/` |
| lash-fichas | V700–V799 | `db/migration/fichas/` |
| lash-dashboard | V800–V899 | `db/migration/dashboard/` |

O `lash-app` configura Flyway para agregar todos os locations do classpath.

## Escopo

- **Inclui**: reorganização do código Java, POMs, migrations, testes
- **Não inclui**: mudança de comportamento ou lógica de negócio
- **Não inclui**: frontend (Angular permanece em `lash-frontend/` sem alterações)

## Requisitos

| ID | Requisito |
|---|---|
| RMM-01 | Parent POM em `lash-backend/pom.xml` gerencia todos os sub-módulos |
| RMM-02 | `lash-core` contém: `BusinessException`, `DomainException`, `ErrorResponse`, auth domain (User, UserRole), auth use cases (Login, RefreshToken, ForgotPassword), Security (JWT, SecurityConfig, UserDetailsServiceImpl), Email |
| RMM-03 | `AbstractIntegrationTest` e `application-test.yml` ficam em `lash-core` como dependência de escopo `test` |
| RMM-04 | Cada módulo de domínio mantém a estrutura hexagonal interna: `domain/`, `application/`, `infrastructure/`, `adapter/` |
| RMM-05 | Cada módulo de domínio tem seu próprio `pom.xml` declarando somente as dependências que usa |
| RMM-06 | Migrations V1–V7 existentes são redistribuídas para os módulos donos de cada tabela, renomeadas para os ranges corretos |
| RMM-07 | `lash-app` contém apenas `LashManagerApplication.java`, `application.yml` e configura Flyway com todos os locations |
| RMM-08 | `@SpringBootApplication(scanBasePackages = "com.lashmanager")` em `lash-app` garante que todos os beans dos sub-módulos sejam detectados |
| RMM-09 | `mvn package` na raiz gera o JAR executável em `lash-app/target/` |
| RMM-10 | Testes de clientes existentes passam após a migração, localizados em `lash-clients/src/test/` |
