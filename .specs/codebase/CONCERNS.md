# Preocupações — Lash Manager

**Atualizado:** 2026-05-22

Preocupações identificadas no código com evidências concretas. Ordenadas por risco.

---

## 🔴 CRÍTICO

### C01 — Zero cobertura de testes

**Evidência:** `lash-backend/src/test/` vazio; nenhum `*.spec.ts` significativo em `lash-frontend/`  
**Risco:** Qualquer refatoração ou nova feature pode quebrar comportamentos existentes sem ser detectada  
**Impacto:** Todos os módulos (Auth, Clients, Services)

**Correção recomendada:**
1. Prioridade 1: use case tests com Mockito (`CreateClientUseCaseImplTest`, etc.)
2. Prioridade 2: reducer tests no Angular (funções puras, sem setup)
3. Prioridade 3: `@WebMvcTest` para controllers

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

### C03 — Conflito de nome: `domain.model.Service` vs `@Service`

**Evidência:** Todos os `*ServiceUseCaseImpl.java` usam:
```java
@org.springframework.stereotype.Service  // qualificado
```

**Risco:** Baixo risco funcional (código compila e funciona), mas é um smell — viola o princípio de nomes claros e pode causar confusão em novos devs  
**Impacto:** `CreateServiceUseCaseImpl`, `UpdateServiceUseCaseImpl`, `GetServiceUseCaseImpl`, `ListServicesUseCaseImpl`, `DeactivateServiceUseCaseImpl`

**Correção recomendada:** Renomear `domain.model.Service` para `ServiceOffering` ou `LashService`. Requer atualização em ~15 arquivos. Baixa urgência.

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

### C05 — CLI-13 / SVC-13: Desativação sem verificação de agendamentos

**Evidência:** `DeactivateClientUseCaseImpl.java` e `DeactivateServiceUseCaseImpl.java`:
```java
// CLI-13: future appointments check will be added when Appointments module is implemented
// SVC-13: future appointments check will be added when Appointments module is implemented
```

**Risco:** Uma cliente ou serviço com agendamentos futuros pode ser desativado sem aviso  
**Impacto:** Dados inconsistentes — agendamentos futuros ficam "órfãos" visualmente

**Status:** Intencional — deferred até o módulo de Agendamentos ser implementado. Deve ser resolvido no design de Agendamentos.

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

### C07 — Migrations monolíticas (V1 cria schema completo)

**Evidência:** `V1__create_schema.sql` cria 8 tabelas de uma vez, incluindo tabelas de módulos ainda não implementados (appointments, financial_entries, inventory_items, inventory_movements)

**Risco:** Baixo no estado atual. Em produção, se um rollback for necessário em módulo específico, não há granularidade  
**Status:** Aceitável para Fase 1. Para Fase 2+, criar migrations incrementais por módulo (V3, V4, etc.)

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

## Resumo de Prioridades

| ID | Severidade | Título | Ação |
|---|---|---|---|
| C01 | 🔴 Crítico | Zero testes | Implementar gradualmente, começar por use cases |
| C02 | 🔴 Crítico | JWT_SECRET hardcoded | Remover default antes do primeiro deploy |
| C03 | 🟡 Moderado | Conflito nome Service | Renomear domain model em refactor futuro |
| C04 | 🟡 Moderado | Spring no domínio (Page/Pageable) | Aceito pragmaticamente, rever em Fase 2 |
| C05 | 🟡 Moderado | CLI-13 / SVC-13 deferred | Resolver no módulo de Agendamentos |
| C06 | 🟡 Moderado | Bundle Angular acima do budget | Auditar imports após completar Fase 1 |
| C07 | 🟢 Baixo | Migrations monolíticas | Aceitável para Fase 1 |
| C08 | 🟢 Baixo | SMTP não funcional | Implementar quando UI de reset existir |
| C09 | 🟢 Baixo | Sem Dockerfile | Trabalho pré-produção |
