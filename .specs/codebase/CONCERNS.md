# Preocupações — Lash Manager

**Atualizado:** 2026-07-14

Preocupações identificadas no código com evidências concretas. Ordenadas por risco.

---

## 🔴 CRÍTICO

### C01 — 7 de 9 módulos backend sem nenhum teste automatizado

**Evidência:** apenas `lash-core` (1 arquivo) e `lash-clients` (8 arquivos) têm `src/test/` com conteúdo. `lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`, `lash-fichas`, `lash-dashboard` e `lash-app` têm `src/test/` vazio ou inexistente, apesar de todos terem domain/use case/controller implementados. Nenhum `*.spec.ts` significativo em `lash-frontend/`  
**Risco:** Qualquer refatoração ou nova feature nesses 7 módulos pode quebrar comportamentos existentes sem ser detectada  
**Impacto:** lash-services, lash-appointments, lash-finance, lash-stock, lash-fichas, lash-dashboard, lash-app

**Correção recomendada:** usar `lash-clients` como modelo de referência (ver TESTING.md) — replicar por módulo, na ordem de dependência do reactor:
1. Prioridade 1: `lash-services` e `lash-appointments` — use case tests com Mockito + `@DisplayName` em português
2. Prioridade 2: `lash-finance`, `lash-stock`, `lash-fichas` — mesmo padrão
3. Prioridade 3: `lash-dashboard` (agregação, poucos use cases) e `@WebMvcTest` para controllers de todos os módulos
4. Prioridade 4: reducer tests no Angular (funções puras, sem setup)

---

### C02 — JWT_SECRET hardcoded como default

**Evidência:** `lash-backend/src/main/resources/application.yml`:
```yaml
secret: ${JWT_SECRET:404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970}
```

**Risco:** Se deployado em produção sem definir `JWT_SECRET`, qualquer pessoa com acesso ao código pode assinar tokens válidos  
**Impacto:** Comprometimento total da autenticação

**Correção recomendada:** Remover o valor default do `application.yml`. Fazer o app falhar explicitamente na inicialização se `JWT_SECRET` não estiver definida:
```java
@Value("${app.jwt.secret}") // sem default — falha ao subir se não configurado
private String jwtSecret;
```

---

## 🟡 MODERADO

### C03 — [RESOLVIDO] Conflito de nome: `domain.model.Service` vs `@Service`

**Status:** Resolvido em 2026-07-14. O domain model de `lash-services` foi renomeado de `Service` para `ServiceOffering` (arquivo `lash-services/domain/model/ServiceOffering.java`) — atualizados `ServiceRepository`, `ServiceMapper`, `ServiceRepositoryImpl`, `ServiceUseCaseMapper` e os 6 `*ServiceUseCaseImpl`, além de `CreateAppointmentUseCaseImpl` e `UpdateAppointmentUseCaseImpl` em `lash-appointments` (que também referenciavam o tipo). Todos os `*UseCaseImpl` desses módulos voltaram a usar `@Service` normal, sem qualificação — o conflito não existe mais em lugar nenhum do código.

**Nota histórica:** durante o refactor multi-módulo, o wiring dos use cases também tinha sido trocado temporariamente por `{Modulo}Config`/`@Bean` (que também "escondia" este conflito, por não usar anotação Spring nenhuma). Essa mudança não tinha sido pedida e foi revertida antes deste fix — o problema real (nome do domain model) só foi resolvido depois, com o rename.

---

### C04 — Acoplamento Spring no domínio (paginação)

**Evidência:** `domain/port/in/ListClientsUseCase.java` e `ListServicesUseCase.java`:
```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ListClientsUseCase {
    Page<CreateClientUseCase.ClientResult> execute(String search, Pageable pageable);
}
```

**Risco:** O domínio tem dependência direta do Spring Data — viola o princípio da arquitetura hexagonal (domínio livre de frameworks)  
**Impacto:** Dificulta testes unitários puros e eventual troca de framework

**Decisão tomada:** Aceito pragmaticamente para Fase 1 (documentado no design de Clientes e Serviços). Rever em Fase 2 se necessário.

**Correção possível:** Criar record de paginação próprio no domínio:
```java
record PageRequest(int page, int size) {}
record PageResult<T>(List<T> content, long totalElements, int page) {}
```

---

### C05 — SVC-13: Desativação de serviço ainda sem verificação de agendamentos (resolvido para clientes)

**Evidência:** com `lash-appointments` implementado, `lash-clients` já resolveu a checagem — `DeactivateClientUseCaseImpl` e `DeleteClientUseCaseImpl` usam `ClientAppointmentPort` (implementada em `lash-appointments/infrastructure/adapter`) e lançam `HasFutureAppointmentsException` quando há agendamento futuro ativo. `lash-services/DeactivateServiceUseCaseImpl` **ainda não tem** a checagem equivalente — não depende de `lash-appointments` no `pom.xml`  
**Risco:** Um serviço com agendamentos futuros pode ser desativado sem aviso  
**Impacto:** Dados inconsistentes — agendamentos futuros referenciando um serviço desativado ficam "órfãos" visualmente

**Correção recomendada:** replicar o padrão de `lash-clients` em `lash-services` — criar `ServiceAppointmentPort` em `lash-services/domain/port/out`, implementar em `lash-appointments/infrastructure/adapter`, adicionar `lash-appointments` como dependência de `lash-services` no `pom.xml` (cuidado: isso inverteria a direção atual da dependência, já que hoje `lash-appointments` depende de `lash-services` — avaliar se a porta deve, em vez disso, viver em `lash-appointments` consultando `lash-services` só por leitura, ou se o check deve ser movido para `lash-appointments` no momento da criação do agendamento)

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

## 🟢 BAIXO / OBSERVAÇÃO

### C07 — [RESOLVIDO] Migrations monolíticas

**Status:** Resolvido pela refatoração multi-módulo. Cada módulo agora tem sua própria migration (`V100`–`V800`, um range de 100 por módulo) em vez de um único `V1__create_schema.sql` — ver STRUCTURE.md e INTEGRATIONS.md. Mantido aqui só como registro histórico da decisão.

---

### C08 — JavaMailSender configurado mas não usado

**Evidência:** `application.yml` configura SMTP completo; endpoints de forgot-password existem no backend mas não há implementação de envio real

**Risco:** Variáveis `MAIL_USER`/`MAIL_PASS` vazias em dev — se qualquer código tentar enviar e-mail, vai falhar silenciosamente  
**Status:** Não é bloqueante pois os endpoints de reset de senha não estão na UI atual

---

### C09 — Sem Dockerfile / configuração de deploy

**Evidência:** Nenhum `Dockerfile`, `docker-compose.yml` ou pipeline CI/CD no repositório

**Risco:** Deploy manual, sem reprodutibilidade do ambiente de produção  
**Status:** Fora de escopo para Fase 1 — documentar como trabalho pendente para quando o produto for ao ar

---

### C10 — `lash-services` não pode depender de `lash-appointments` (dependência já invertida)

**Evidência:** `lash-appointments/pom.xml` já depende de `lash-core`, `lash-clients` e `lash-services`. Isso bloqueia C05 (checagem de agendamento futuro na desativação de serviço) pelo caminho usado em `lash-clients`, porque criar `lash-appointments → lash-services` na direção inversa geraria dependência circular no reactor Maven  
**Risco:** Baixo isoladamente, mas é uma restrição estrutural que qualquer nova feature cross-módulo entre `lash-services` e `lash-appointments` vai esbarrar  
**Impacto:** `lash-services`, `lash-appointments`

**Correção recomendada:** ao resolver C05, decidir entre (a) mover a checagem para dentro de `lash-appointments` (que já pode ler `lash-services`) — ex.: validar na criação do agendamento que o serviço está ativo, ou expor um evento/consulta somente-leitura; (b) extrair um port neutro em `lash-core` que ambos os módulos implementam/consultam sem dependência direta entre si. Não decidido ainda — registrar a decisão em STATE.md quando escolhida.

---

## Resumo de Prioridades

| ID | Severidade | Título | Ação |
|---|---|---|---|
| C01 | 🔴 Crítico | 7 de 9 módulos sem testes | Implementar gradualmente, seguindo o modelo de lash-clients |
| C02 | 🔴 Crítico | JWT_SECRET hardcoded | Remover default antes do primeiro deploy |
| C03 | ✅ Resolvido | Conflito nome Service | Domain model renomeado para `ServiceOffering` |
| C04 | 🟡 Moderado | Spring no domínio (Page/Pageable) | Aceito pragmaticamente, rever em Fase 2 |
| C05 | 🟡 Moderado | SVC-13 deferred (serviços) | Resolver — ver C10 para a restrição de dependência |
| C06 | 🟡 Moderado | Bundle Angular acima do budget | Auditar imports após completar Fase 1 |
| C07 | ✅ Resolvido | Migrations monolíticas | Resolvido pela refatoração multi-módulo |
| C08 | 🟢 Baixo | SMTP não funcional | Implementar quando UI de reset existir |
| C09 | 🟢 Baixo | Sem Dockerfile | Trabalho pré-produção |
| C10 | 🟡 Moderado | Restrição de dependência lash-services ↔ lash-appointments | Decidir abordagem ao resolver C05 |
