# Spec — Testes do Módulo de Clientes

**Data:** 2026-05-29
**Escopo:** Large
**Status:** Pronto para tasks

---

## Contexto

O módulo de Clientes está implementado e testado manualmente pela usuária. Este feature adiciona cobertura automatizada com:

- Testes **unitários** (`*Test.java`) para cada use case, isolados com Mockito
- Testes de **integração** (`*ITest.java`) cobrindo o fluxo CRUD completo contra o PostgreSQL real
- Classe base `AbstractIntegrationTest` reutilizada por todos os ITests futuros

Padrão definido pela usuária:
- Unitário: `@ExtendWith(MockitoExtension.class)`, `@Mock`, instância manual no `@BeforeEach`, AssertJ
- Integração: `@SpringBootTest + @Transactional + @Rollback`, `@Autowired` nos use cases, banco Docker porta 5433

---

## Requisitos

### Infraestrutura de teste

**TC-01** — Criar `AbstractIntegrationTest` em `src/test/java/.../AbstractIntegrationTest.java`
- Anotada com `@SpringBootTest`, `@Transactional`, `@Rollback`, `@ActiveProfiles("test")`
- Classe base abstrata sem nenhuma lógica própria — só carrega o contexto Spring com profile "test"
- Todos os `*ITest.java` do projeto estendem esta classe

**TC-02** — Criar `src/test/resources/application-test.yml`
- Aponta para o banco PostgreSQL Docker: `jdbc:postgresql://localhost:5433/lashmanager`, user `postgres`, pass `postgres`
- `ddl-auto: validate` (Flyway já aplicou as migrations)
- `flyway.enabled: true` (roda no mesmo schema)
- Desabilitar e-mail: `spring.mail.host: localhost` + porta inativa (evita falha de conexão SMTP nos testes)
- JWT secret e expiração idênticos ao `application.yml` de dev

---

### Testes unitários — CreateClientUseCaseImpl

**TC-03** — `execute` com telefone novo → salva o cliente e retorna `ClientResult` com os dados corretos
- Mock `clientRepository.existsByPhone()` retorna `false`
- Mock `clientRepository.save()` retorna o `Client` construído
- Assertar: `result.name()`, `result.phone()`, `result.active() == true`

**TC-04** — `execute` com telefone já cadastrado → lança `ClientAlreadyExistsException`
- Mock `existsByPhone()` retorna `true`
- Assertar: `assertThatThrownBy(...).isInstanceOf(ClientAlreadyExistsException.class)`
- Verificar que `clientRepository.save()` **nunca** foi chamado (`verify(..., never())`)

---

### Testes unitários — UpdateClientUseCaseImpl

**TC-05** — `execute` com cliente existente e telefone não conflitante → retorna `ClientResult` com dados atualizados
- Mock `findById()` retorna `Optional.of(clientExistente)`
- Mock `existsByPhoneAndIdNot()` retorna `false`
- Mock `save()` retorna o cliente atualizado
- Assertar que `result.name()` reflete o novo nome do command

**TC-06** — `execute` com ID inexistente → lança `ClientNotFoundException`
- Mock `findById()` retorna `Optional.empty()`
- `save()` nunca chamado

**TC-07** — `execute` com telefone pertencente a outro cliente → lança `ClientAlreadyExistsException`
- Mock `findById()` retorna `Optional.of(cliente)`
- Mock `existsByPhoneAndIdNot()` retorna `true`
- `save()` nunca chamado

**TC-08** — `execute` com mesmo telefone do próprio cliente → sucesso (não conflita)
- Mock `existsByPhoneAndIdNot()` retorna `false` (mesmo comportamento — a checagem usa `IdNot`)
- Assertar que `save()` foi chamado uma vez

---

### Testes unitários — GetClientUseCaseImpl

**TC-09** — `execute` com ID existente → retorna `ClientResult` com todos os campos
- Mock `findById()` retorna `Optional.of(client)`
- Assertar `result.id()` == `client.getId()`

**TC-10** — `execute` com ID inexistente → lança `ClientNotFoundException`
- Mock `findById()` retorna `Optional.empty()`

---

### Testes unitários — ListClientsUseCaseImpl

**TC-11** — `execute` com `search = null` → normaliza para `""` antes de chamar o repositório
- Capturar argumento com `ArgumentCaptor<String>`
- Assertar que o repositório recebeu `""` e não `null`

**TC-12** — `execute` com `search = "  Ana  "` → faz trim, passa `"Ana"` ao repositório
- Mesmo padrão com `ArgumentCaptor`

**TC-13** — `execute` com parâmetros completos → delega ao repositório e mapeia resultado
- Mock `findAll()` retorna `Page<Client>` com um item
- Assertar que o resultado tem 1 elemento com os dados corretos

---

### Testes unitários — DeleteClientUseCaseImpl

**TC-14** — `execute` com cliente sem agendamentos futuros ativos → executa as 3 operações na ordem correta
- Mock `findById()` retorna cliente
- Mock `findFutureActiveByClientId()` retorna lista vazia
- Verificar com `InOrder`: `deleteFutureAppointmentsByClientId` → `unlinkClientFromPastAppointments` → `deleteById`

**TC-15** — `execute` com ID inexistente → lança `ClientNotFoundException`
- Mock `findById()` retorna `Optional.empty()`
- Nenhuma operação de appointment chamada

**TC-16** — `execute` com agendamentos futuros ativos → lança `HasFutureAppointmentsException` com a lista de agendamentos
- Mock `findFutureActiveByClientId()` retorna lista com 2 summaries
- Assertar: `isInstanceOf(HasFutureAppointmentsException.class)`
- Assertar que `deleteById` **nunca** foi chamado

---

### Testes unitários — DeactivateClientUseCaseImpl

> Incluído por cobertura completa do módulo (não estava na lista original).

**TC-17** — `deactivate` sem agendamentos futuros e `force=false` → salva cliente com `active=false`
- `findFutureActiveByClientId()` retorna lista vazia
- Capturar o `Client` passado para `save()` e assertar `active == false`

**TC-18** — `deactivate` com agendamentos futuros e `force=false` → lança `HasFutureAppointmentsException`
- `findFutureActiveByClientId()` retorna lista não-vazia
- `save()` nunca chamado

**TC-19** — `deactivate` com agendamentos futuros e `force=true` → ignora os agendamentos e salva com `active=false`
- `findFutureActiveByClientId()` **nunca chamado** (não deve checar quando force=true)
- `save()` chamado com `active=false`

**TC-20** — `deactivate` com cliente inexistente → lança `ClientNotFoundException`

**TC-21** — `reactivate` com cliente existente → salva com `active=true`
- Capturar `Client` e assertar `active == true`

**TC-22** — `reactivate` com cliente inexistente → lança `ClientNotFoundException`

---

### Testes de integração — ClientITest

**TC-23** — Fluxo completo: criar → buscar → atualizar → deletar
- `CreateClientUseCase.execute(command)` → assertar retorno com `active=true`
- `GetClientUseCase.execute(id)` → assertar mesmo nome/telefone
- `UpdateClientUseCase.execute(id, updateCommand)` → assertar que nome foi alterado
- `DeleteClientUseCase.execute(id)` → assertar que `GetClientUseCase.execute(id)` lança `ClientNotFoundException`

**TC-24** — Criar cliente com telefone duplicado → lança `ClientAlreadyExistsException`
- Criar primeiro cliente, tentar criar segundo com mesmo telefone

**TC-25** — Desativar cliente → `active` persiste como `false` no banco
- `DeactivateClientUseCase.deactivate(id, false)` 
- `GetClientUseCase.execute(id)` → assertar `active=false`

**TC-26** — Reativar cliente desativado → `active` persiste como `true`
- Desativar → reativar → buscar → assertar `active=true`

**TC-27** — Busca com search filtra por nome
- Criar dois clientes: "Ana Lima" e "Beatriz Costa"
- `ListClientsUseCase.execute("Ana", null, pageable)` → retorna somente 1 resultado com nome "Ana Lima"

**TC-28** — Filtro `active=false` retorna apenas clientes inativos
- Criar um cliente ativo e um desativado
- `ListClientsUseCase.execute("", false, pageable)` → a lista contém o desativado mas não o ativo

---

## Localização dos arquivos

```
lash-backend/src/test/
├── java/com/lashmanager/app/
│   ├── AbstractIntegrationTest.java
│   └── application/usecase/
│       ├── CreateClientUseCaseImplTest.java
│       ├── UpdateClientUseCaseImplTest.java
│       ├── GetClientUseCaseImplTest.java
│       ├── ListClientsUseCaseImplTest.java
│       ├── DeleteClientUseCaseImplTest.java
│       ├── DeactivateClientUseCaseImplTest.java
│       └── ClientITest.java
└── resources/
    └── application-test.yml
```

---

## Dependências já disponíveis no pom.xml

`spring-boot-starter-test` (escopo test) já inclui:
- JUnit 5 (`@ExtendWith`, `@Test`, `@BeforeEach`)
- Mockito (`@Mock`, `@ExtendWith(MockitoExtension.class)`, `verify`, `ArgumentCaptor`)
- AssertJ (`assertThat`, `assertThatThrownBy`)

Nenhuma dependência adicional necessária.

---

## Restrições e decisões

| Decisão | Razão |
|---|---|
| Banco de integração = Docker local porta 5433 | Sem Testcontainers — reutiliza o container já em execução |
| `@Rollback` nos ITests | Cada teste reverte as alterações; banco não acumula dados entre testes |
| `@ActiveProfiles("test")` | Isola configuração de teste (mail, secrets) sem alterar `application.yml` de dev |
| Sufixo `*Test.java` para unitário, `*ITest.java` para integração | Convenção definida pela usuária; Maven Surefire roda `*Test` por padrão; `*ITest` pode ser separado com failsafe se necessário |
| `DeactivateClientUseCaseImpl` incluído | Pertence ao módulo de Clientes; omiti-lo deixaria cobertura incompleta |
| `ListMappingsByClientUseCaseImpl` não incluído | Depende do módulo de Fichas (não implementado); escopo fora dos 5 use cases solicitados |
