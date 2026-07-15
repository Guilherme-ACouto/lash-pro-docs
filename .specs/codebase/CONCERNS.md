# Preocupações — Lash Manager

**Atualizado:** 2026-07-15

Preocupações identificadas no código com evidências concretas. Ordenadas por risco.

---

## 🔴 CRÍTICO

### C01 — 7 de 9 módulos backend sem nenhum teste automatizado (agravado pela Fase E)

**Evidência:** apenas `lash-core` (4 arquivos) e `lash-clients` (8 arquivos) têm `src/test/` com conteúdo. `lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`, `lash-fichas`, `lash-dashboard` e `lash-app` têm `src/test/` vazio ou inexistente. A refatoração `refactor-backend` adicionou uma quantidade grande de código novo (`*Command`, `*ApplicationService`, `*QueryRepository`) exatamente nesses 6 módulos sem teste — o volume de código não coberto cresceu, não diminuiu
**Risco:** Qualquer refatoração ou nova feature nesses 7 módulos pode quebrar comportamentos existentes (incluindo o padrão Command/ApplicationService recém-implementado) sem ser detectada
**Impacto:** lash-services, lash-appointments, lash-finance, lash-stock, lash-fichas, lash-dashboard, lash-app

**Correção recomendada:** usar `lash-clients` como modelo de referência (ver TESTING.md) — replicar por módulo, na ordem de dependência do reactor, cobrindo tanto os `*UseCaseImpl` quanto os `*Command`/`*ApplicationService` novos:
1. Prioridade 1: `lash-services` e `lash-appointments` — use case tests com Mockito + `@DisplayName` em português
2. Prioridade 2: `lash-finance`, `lash-stock`, `lash-fichas` — mesmo padrão
3. Prioridade 3: `lash-dashboard` (agregação, poucos use cases) e `@WebMvcTest` para controllers de todos os módulos
4. Prioridade 4: reducer tests no Angular (funções puras, sem setup)

---

### C02 — JWT_SECRET hardcoded como default (risco ampliado pela multi-tenancy)

**Evidência:** `lash-app/src/main/resources/application.yml`:
```yaml
secret: ${JWT_SECRET:404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970}
```

**Risco:** Se deployado em produção sem definir `JWT_SECRET`, qualquer pessoa com acesso ao código pode assinar tokens válidos. **Agora o token também carrega o claim `tenantId`** — um token forjado permite não só autenticar como qualquer usuário, mas também **acessar dados de qualquer tenant** (o `TenantResolvingFilter` confia no claim sem validação adicional contra o banco). O impacto de um `JWT_SECRET` vazado ficou maior num contexto multi-tenant do que era em single-tenant
**Impacto:** Comprometimento total da autenticação e do isolamento entre tenants

**Correção recomendada:** Remover o valor default do `application.yml`. Fazer o app falhar explicitamente na inicialização se `JWT_SECRET` não estiver definida:
```java
@Value("${app.jwt.secret}") // sem default — falha ao subir se não configurado
private String jwtSecret;
```

---

## 🟡 MODERADO

### C03 — [RESOLVIDO] Conflito de nome: `domain.model.Service` vs `@Service`

**Status:** Resolvido em 2026-07-14. O domain model de `lash-services` foi renomeado de `Service` para `ServiceOffering`. Todos os `*UseCaseImpl` voltaram a usar `@Service` normal, sem qualificação — o conflito não existe mais em lugar nenhum do código.

---

### C04 — Acoplamento Spring no domínio (paginação)

**Evidência:** `domain/port/in/ListClientsUseCase.java`, `ListServicesUseCase.java` e os `port/out` de leitura novos (`*QueryRepository`, ver ARCHITECTURE.md) continuam importando `org.springframework.data.domain.Page`/`Pageable` diretamente:
```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ClientQueryRepository {
    Page<Client> findAll(String search, Boolean active, Pageable pageable);
}
```

**Risco:** O domínio tem dependência direta do Spring Data — viola o princípio da arquitetura hexagonal (domínio livre de frameworks)
**Impacto:** Dificulta testes unitários puros e eventual troca de framework. A separação Command/Query (Fase F) **não resolveu** esse acoplamento — só reorganizou onde ele vive, o `Page`/`Pageable` continua atravessando `domain/port/out` em todos os módulos com listagem

**Decisão tomada:** Aceito pragmaticamente para Fase 1/2 (documentado no design de Clientes e Serviços, reafirmado no `refactor-backend`). Rever se/quando o domínio precisar ser testado sem contexto Spring.

---

### C05 — SVC-13: Desativação de serviço ainda sem verificação de agendamentos (não resolvido)

**Evidência:** `lash-clients/DeactivateClientUseCaseImpl` usa `ClientAppointmentPort` (implementada em `lash-appointments`) e lança `HasFutureAppointmentsException` quando há agendamento futuro ativo. `lash-services/DeactivateServiceUseCaseImpl` **ainda não tem** a checagem equivalente
**Risco:** Um serviço com agendamentos futuros pode ser desativado sem aviso
**Impacto:** Dados inconsistentes — agendamentos futuros referenciando um serviço desativado ficam "órfãos" visualmente
**Status:** Ainda não resolvido. Levantado durante o `refactor-backend` como possível candidato ao padrão de módulo `*-api` visto na Pontta (ver C10), mas nenhuma decisão foi tomada nem implementada

**Correção recomendada:** ver C10 — a restrição de dependência entre `lash-services` e `lash-appointments` precisa ser resolvida antes (ou junto) de implementar essa checagem.

---

### C06 — Bundle Angular acima do budget

**Evidência:** `npm run build` exibe warning de orçamento de bundle excedido
**Risco:** Tempo de carregamento inicial maior; penaliza usuários mobile com conexão lenta
**Impacto:** UX em mobile — target principal do produto

**Correção recomendada:**
1. Auditar dependências importadas desnecessariamente (Angular Material módulos não usados)
2. Verificar se `MatNativeDateModule` está sendo carregado em todo lugar (pesado)
3. Lazy loading já está configurado — verificar se todos os módulos estão realmente lazy

---

### C10 — `lash-services` não pode depender de `lash-appointments` (dependência já invertida)

**Evidência:** `lash-appointments/pom.xml` já depende de `lash-core`, `lash-clients` e `lash-services`. Isso bloqueia C05 pelo caminho usado em `lash-clients`, porque criar `lash-appointments → lash-services` na direção inversa geraria dependência circular no reactor Maven
**Risco:** Baixo isoladamente, mas é uma restrição estrutural que qualquer nova feature cross-módulo entre `lash-services` e `lash-appointments` vai esbarrar
**Impacto:** `lash-services`, `lash-appointments`

**Correção recomendada:** ao resolver C05, decidir entre (a) mover a checagem para dentro de `lash-appointments` (que já pode ler `lash-services`), ou (b) extrair um módulo `*-api` — padrão visto na arquitetura da Pontta (módulos que expõem só contrato, consumidos por múltiplos domínios sem criar dependência direta pesada entre implementações). Ainda não decidido nem avaliado a fundo — próximo passo natural depois de validar o `refactor-backend`.

---

### C11 — Sem teste automatizado de isolamento real entre schemas de tenant (novo, decisão consciente)

**Evidência:** um `TenantIsolationITest` chegou a ser escrito (dois schemas de tenant reais, prova que dado de um não aparece em query do outro) mas foi **removido a pedido da usuária**, replicando a Pontta — que também não tem esse teste (evitam criar schema real em teste; mockam a camada de provisionamento nos poucos testes que a tocam)
**Risco:** O mecanismo de isolamento (Hibernate `MultiTenantConnectionProviderImpl` trocando `search_path`) foi implementado e é exercitado manualmente em ambiente real, mas uma regressão futura nele (ex.: alguém remove o `SET search_path` ou muda a ordem do fallback `public`) não seria pega automaticamente por nenhum teste
**Impacto:** Todo o sistema — isolamento entre tenants é a garantia mais fundamental de um SaaS multi-tenant

**Status:** Risco aceito conscientemente pela usuária (não é um gap descoberto por acidente). Reavaliar se o produto crescer a ponto de justificar o custo de manter esse teste (ex.: schema efêmero via Testcontainers, que resolveria tanto isolamento quanto limpeza — não avaliado ainda)

---

### C12 — Sem job de limpeza de schemas Postgres órfãos

**Evidência:** se um cadastro nunca é ativado (usuário não clica no link), ou se `RegistrationFlowITest`/testes similares rodam repetidamente, schemas `tenant_*` reais se acumulam no banco (dev ou teste) sem nenhuma rotina de remoção — mesmo gap que a própria Pontta tem (RBK-D01, ver `.specs/features/refactor-backend/spec.md`)
**Risco:** Baixo no curto prazo (schema vazio ocupa pouco espaço), mas cresce com o tempo/uso — nenhum mecanismo de observabilidade pra saber quantos schemas órfãos existem hoje
**Impacto:** Banco de dev e de teste; eventualmente banco de produção se o SaaS for ao ar sem essa rotina

**Correção recomendada:** job periódico (ex.: `@Scheduled`) que identifica tenants com `active=false` e criados há mais de N dias sem nunca terem sido ativados, e faz `DROP SCHEMA IF EXISTS ... CASCADE` + remove a linha em `tenants`. Não implementado — deferred desde a criação do `refactor-backend`.

---

## 🟢 BAIXO / OBSERVAÇÃO

### C07 — [RESOLVIDO] Migrations monolíticas

**Status:** Resolvido pela refatoração multi-módulo. Cada módulo tem sua própria migration Flyway (schema `public`) e changelog Liquibase (schema de tenant) — ver STRUCTURE.md e INTEGRATIONS.md.

---

### C08 — [PARCIALMENTE RESOLVIDO] SMTP configurado, agora usado de fato

**Evidência:** `EmailPortImpl` agora envia o e-mail de ativação de conta (`sendActivationEmail`), além do de recuperação de senha (`sendPasswordResetEmail`) — os dois em texto simples, sem Thymeleaf
**Risco residual:** `MAIL_USER`/`MAIL_PASS` ainda vazios em dev por padrão — falha de envio é capturada e só logada como `WARN`, não quebra o fluxo, mas em produção sem essas variáveis configuradas nenhum e-mail de fato chega
**Status:** Endpoints de reset de senha ainda não estão na UI do frontend; o fluxo de registro/ativação (backend) também não tem UI ainda — mas ambos os fluxos de e-mail já funcionam via API/Postman

---

### C09 — Sem Dockerfile / configuração de deploy

**Evidência:** Nenhum `Dockerfile`, `docker-compose.yml` ou pipeline CI/CD no repositório
**Risco:** Deploy manual, sem reprodutibilidade do ambiente de produção
**Status:** Fora de escopo por enquanto — documentar como trabalho pendente para quando o produto for ao ar. Ganhou relevância com a multi-tenancy (provisionamento de schema, banco de teste separado) — vale revisitar junto de C12

---

## Resumo de Prioridades

| ID | Severidade | Título | Ação |
|---|---|---|---|
| C01 | 🔴 Crítico | 7 de 9 módulos sem testes (agravado pela Fase E) | Implementar gradualmente, seguindo o modelo de lash-clients |
| C02 | 🔴 Crítico | JWT_SECRET hardcoded (risco ampliado por multi-tenancy) | Remover default antes do primeiro deploy |
| C03 | ✅ Resolvido | Conflito nome Service | Domain model renomeado para `ServiceOffering` |
| C04 | 🟡 Moderado | Spring no domínio (Page/Pageable) — agora também nos QueryRepository | Aceito pragmaticamente |
| C05 | 🟡 Moderado | SVC-13 deferred (serviços) | Resolver — ver C10 para a restrição de dependência |
| C06 | 🟡 Moderado | Bundle Angular acima do budget | Auditar imports após completar Fase 1 |
| C07 | ✅ Resolvido | Migrations monolíticas | Resolvido pela refatoração multi-módulo |
| C08 | 🟢 Baixo | SMTP — parcialmente resolvido | E-mails funcionam via API; falta UI e config de produção |
| C09 | 🟢 Baixo | Sem Dockerfile | Trabalho pré-produção, ganhou relevância com multi-tenancy |
| C10 | 🟡 Moderado | Restrição de dependência lash-services ↔ lash-appointments | Decidir abordagem ao resolver C05 (avaliar módulo `*-api`) |
| C11 | 🟡 Moderado | Sem teste automatizado de isolamento real entre tenants | Risco aceito conscientemente — reavaliar se o produto crescer |
| C12 | 🟢 Baixo | Sem job de limpeza de schemas órfãos | Deferred (RBK-D01) |
