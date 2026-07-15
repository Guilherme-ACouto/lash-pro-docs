# Refatoração Backend → Padrão Pontta (Multi-Tenancy + Command/ApplicationService/Query) — Spec

**Status**: Draft
**Data**: 2026-07-15

---

## Contexto

Hoje o Lash Manager é um sistema single-tenant: um único Postgres, um único schema (`public`), uma `DB_URL` fixa, e um único usuário fixo (`admin@lashmanager.com`) representando o único salão existente (o da usuária). Não existe conceito de "empresa"/"conta"/"tenant" em nenhuma camada — nem no banco, nem na autenticação, nem no domínio.

A usuária pretende transformar o Lash Manager em um produto SaaS, vendido para outros salões. Isso exige isolar os dados de cada cliente (salão) que assinar o produto, sem deixar de oferecer um fluxo de cadastro self-service.

Como referência de mercado, foi levantado (via consulta a uma IA com acesso ao codebase da Pontta, empresa onde a usuária trabalha) como um sistema real já resolve esse problema: **schema-per-tenant** no mesmo Postgres, com Hibernate `MultiTenancyStrategy.SCHEMA`, resolução de tenant via e-mail no login, e provisionamento automático (schema criado + migrado) durante a ativação de conta por e-mail.

Além disso, a usuária trabalha na Pontta e identificou que o padrão arquitetural de lá (`Command` → `ApplicationService` → `UseCase` para escrita, `Specification`/`Dao`+`RowMapper` para leitura, `CommandInterceptor` central) resolve um problema real que o Lash já tem hoje: os UseCases atuais são fáceis de testar (granularidade fina, um arquivo por operação), mas isso gerou 42 arquivos espalhados em 7 módulos sem nenhuma camada de orquestração, validação centralizada ou auditoria. Replicar esse padrão serve tanto ao produto quanto ao aprendizado da usuária.

## Objetivo

Reformar o backend do Lash Manager para ficar estruturalmente alinhado com o padrão da Pontta, em duas frentes que se apoiam uma na outra:

1. **Multi-tenancy** (schema-per-tenant) com fluxo de autocadastro (registro + ativação por e-mail) que provisiona um novo tenant automaticamente — reaproveitando ao máximo a infraestrutura já existente (SMTP, JWT, Flyway → substituído por Liquibase apenas para os schemas de tenant, ver RBK-02).
2. **Command + ApplicationService + Query** — introduzir a camada de orquestração (`ApplicationService`) e o objeto `Command` formal entre Controller e UseCase, um `CommandInterceptor` central (validação + auditoria, com gancho para permissão), e formalizar a separação leitura/escrita nos repositórios (`Specification` para leitura simples, `Dao`+`RowMapper` para leitura composta) — generalizando o que hoje só existe parcialmente em `FinancialSummaryRepositoryImpl`/Dashboard.

A frente 2 é **piloto primeiro, propagação depois**: os UseCases novos desta mesma spec (Register/ResendActivation/ActivateAccount, em `lash-core`) já nascem no padrão novo — validando a abordagem em código real antes de migrar os 42 UseCases existentes nos outros 7 módulos.

## Decisões Registradas (Fase 0 — já validadas com a usuária)

| Decisão | Razão |
|---|---|
| Liquibase para migrar os schemas de tenant (não Flyway) | Alinhar com o padrão já validado na Pontta; Liquibase rastreia changesets aplicados por schema, o que torna o provisionamento idempotente (`CREATE SCHEMA IF NOT EXISTS` + reexecução segura) |
| `admin@lashmanager.com` permanece como usuário de desenvolvimento/plataforma | Não vira o "tenant 1"; é usado só para acesso interno/dev, fora do fluxo de autocadastro |
| E-mail de ativação reaproveita a infra SMTP existente (usada hoje só no "esqueci senha") | Evita construir mecanismo de envio de e-mail do zero |
| `activationKey` **tem** expiração (diferente da Pontta) | Gap identificado na Pontta (lá não expira nunca) — decidido corrigir no Lash desde o início |
| Existe endpoint de reenvio de e-mail de ativação, condicionado a `!user.isActive()` | Gap identificado na Pontta (lá, recadastro com e-mail pendente sempre falha com "e-mail já em uso", sem saída) — decidido corrigir no Lash |
| Provisionamento de schema roda em conexão JDBC separada da transação Spring (JPA) | Espelha o design da Pontta: evita estado inconsistente sem exigir saga/compensação complexa — `CREATE SCHEMA IF NOT EXISTS` + Liquibase idempotente tornam o retry seguro |

## Escopo

- **Inclui**: fundação técnica de multi-tenancy (Hibernate SCHEMA strategy, resolução de tenant via JWT), migração da tabela `users` para o schema `public`, fluxo completo de registro + ativação + provisionamento automático de schema, ajustes de teste de integração para rodar contra schema isolado.
- **Inclui**: padrão `Command`/`ApplicationService`/`CommandInterceptor` piloto em `lash-core` (UseCases desta spec), documentado em `CONVENTIONS.md`.
- **Inclui**: plano de migração dos 42 UseCases existentes (7 módulos) para o padrão novo e da separação de repositórios de leitura — quebrado em tasks por módulo, mas com granularidade fina definida **depois** que o piloto em `lash-core` validar a abordagem (ver Fase E/F em `tasks.md`).
- **Não inclui**: billing/cobrança, planos pagos, portal administrativo completo de gestão de tenants (endpoint básico de listar/desativar tenant está no escopo mínimo, gestão avançada não).
- **Não inclui**: frontend completo do fluxo de cadastro (telas de registro/ativação) — este spec cobre o backend; o frontend será uma spec própria depois que o backend estiver validado.
- **Não inclui**: migração de dado real do ambiente de produção da usuária para o novo modelo multi-tenant (fica registrado como pendência operacional, não como tarefa de código).
- **Não inclui**: `@CommandPermission` com controle de acesso granular por operação (o gancho existe no `CommandInterceptor`, mas hoje o Lash só tem um `UserRole`; regras de permissão por role múltiplo ficam para quando existir mais de um papel de usuário).

## Requisitos

### Fundação técnica

| ID | Requisito |
|---|---|
| RBK-01 | Tabela `tenants` criada no schema `public`, com `id (UUID)`, `name`, `schema_name`, `active`, `created_at` |
| RBK-02 | Hibernate configurado com `MultiTenancyStrategy.SCHEMA`; schemas de tenant migrados via Liquibase (não Flyway); `lash-core`/`lash-app` mantêm Flyway apenas para as migrations do schema `public` (tabela `users`, `tenants`) |
| RBK-03 | `CurrentTenantIdentifierResolver` implementado, com default `public` quando não há tenant resolvido no contexto (necessário para o login localizar o usuário) |
| RBK-04 | Contexto de tenant (`TenantContext`, ThreadLocal ou equivalente) populado a partir do claim `tenantId` do JWT, antes de qualquer query de negócio rodar |
| RBK-05 | `DataSource` decorado que executa `SET search_path` por conexão obtida do pool, de acordo com o tenant do contexto atual |
| RBK-06 | `UserEntity`/tabela `users` migrada para o schema `public`; ganha coluna `tenant_id (UUID, FK → tenants.id)` |
| RBK-07 | `JwtService` passa a incluir `tenantId` no payload do token, tanto no login quanto no refresh |
| RBK-08 | Isolamento validado: usuário do tenant A não consegue, por nenhum caminho de API, ler dado do tenant B |

### Registro e ativação

| ID | Requisito |
|---|---|
| RBK-09 | Endpoint `POST /api/register` cria usuário inativo no schema `public`, com `activationKey` e `activationKeyExpiry` |
| RBK-10 | Cadastro com e-mail já existente e ativo retorna erro (`email já em uso`); cadastro com e-mail existente **mas inativo** reenvia o e-mail de ativação (renovando `activationKey`) em vez de dar erro |
| RBK-11 | E-mail de ativação enviado via infra SMTP existente, template simples com nome do usuário + botão de confirmação, reaproveitando o fragmento de rodapé já usado no e-mail de "esqueci senha" |
| RBK-12 | `activationKey` expira após um prazo configurável (`ACTIVATION_KEY_HOURS`, default a definir no design); link expirado leva a uma resposta clara de "link expirado", distinta de "link inválido" |
| RBK-13 | Endpoint `POST /api/register/resend` reenvia o e-mail de ativação para um e-mail com cadastro pendente (`!user.isActive()`) |
| RBK-14 | Endpoint `GET/POST /api/activation?key=` ativa a conta: cria o `Tenant`, provisiona o schema (`CREATE SCHEMA IF NOT EXISTS` + Liquibase), ativa o usuário — tudo dentro do fluxo de ativação |
| RBK-15 | Provisionamento de schema roda em conexão JDBC própria (fora da transação Spring/JPA que ativa o usuário) — falha no schema não deixa o usuário "ativado sem tenant"; falha e retry são seguros por idempotência (`IF NOT EXISTS` + Liquibase changelog) |
| RBK-16 | Login de usuário não ativado é bloqueado com mensagem explícita ("confirme seu cadastro"), sem período de graça |

### Command + ApplicationService + Query (piloto em `lash-core`)

| ID | Requisito |
|---|---|
| RBK-20 | `AbstractCommand` — classe base para todo Command, com suporte a Bean Validation (`@Valid`) |
| RBK-21 | `ApplicationService` — camada de orquestração com método `when(Command)`, entre Controller e UseCase; UseCase mantém a regra de negócio pura (sem mudança de responsabilidade, só de quem o chama) |
| RBK-22 | `CommandInterceptor` (`@Aspect`) intercepta todo método `when(...)` de qualquer `ApplicationService`: roda Bean Validation do Command, loga início/fim da execução, grava entrada de auditoria (`RBK-23`); ponto de extensão para verificação de permissão (`@CommandPermission`) preparado mas não aplicado nesta spec (ver Escopo) |
| RBK-23 | `command_audit_log` — tabela (no schema do tenant) registrando: classe do Command, payload (JSON), usuário executor, timestamp, sucesso/falha |
| RBK-24 | `RegisterUseCase`, `ResendActivationUseCase`, `ActivateAccountUseCase` (desta mesma spec) implementados no padrão novo: `Command` próprio + `ApplicationService.when()` + `UseCase` de domínio |
| RBK-25 | `CONVENTIONS.md` documenta oficialmente os sufixos `*Command`, `*ApplicationService`, `*UseCase`, `*Dao`, `*RowMapper`, `*QueryResource` e quando usar cada um — corrigindo o gap observado na Pontta (lá, parte da convenção só existe "no código", não documentada) |
| RBK-26 | Plano de migração dos 42 UseCases existentes (`lash-clients`, `lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`, `lash-fichas`, `lash-dashboard`) para `Command`/`ApplicationService`, um módulo por vez, após o piloto de `lash-core` validado |
| RBK-27 | Plano de separação de repositórios de leitura por módulo (`Specification` para listagem simples, `Dao`+`RowMapper` para agregação/relatório), generalizando o padrão já usado em `FinancialSummaryRepositoryImpl`/Dashboard |

### Testes e operação

| ID | Requisito |
|---|---|
| RBK-17 | Testes de integração existentes (`AbstractIntegrationTest`, módulo `lash-clients`) continuam passando, agora rodando contra um schema de tenant de teste isolado |
| RBK-18 | Testes cobrindo isolamento entre tenants (tenant A não vê dado do tenant B) e o fluxo completo de registro → ativação → schema provisionado → login |
| RBK-19 | Endpoint administrativo mínimo: listar tenants, desativar tenant (sem UI de gestão avançada) |

## Fora de escopo / Deferred

| ID | Descrição | Quando resolver |
|---|---|---|
| RBK-D01 | Job de limpeza de schemas órfãos (tenant nunca ativado após N dias) | Depois do MVP multi-tenant — gap que a própria Pontta também não resolve |
| RBK-D02 | Migração do dado real da usuária para o novo modelo | Operacional, decidir junto com a usuária antes do deploy |
| RBK-D03 | Frontend de registro/ativação | Spec própria, depois do backend validado |
| RBK-D04 | `@CommandPermission` com múltiplos papéis de usuário | Quando existir mais de um `UserRole` além do atual |
