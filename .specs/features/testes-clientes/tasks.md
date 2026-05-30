# Tasks — Testes do Módulo de Clientes

**Data:** 2026-05-29
**Status:** Pronto para implementação

Legenda: `[P]` = pode rodar em paralelo com outras tasks do mesmo grupo

---

## Grupo 1 — Infraestrutura de teste

### TASK-01 — `application-test.yml`
**O quê:** Criar o arquivo de configuração para o profile `test`.
**Onde:** `lash-backend/src/test/resources/application-test.yml`
**Conteúdo:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5433/lashmanager
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
  mail:
    host: localhost
    port: 3025
    username: test
    password: test
    properties:
      mail:
        smtp:
          auth: false
          starttls:
            enable: false

app:
  jwt:
    secret: 404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970
    expiration: 86400000
    refresh-expiration: 604800000
  cors:
    allowed-origins: http://localhost:4200
```
**Depende de:** nada
**Pronto quando:** arquivo existe em `src/test/resources/`

---

### TASK-02 — `AbstractIntegrationTest`
**O quê:** Criar a classe base para todos os testes de integração.
**Onde:** `lash-backend/src/test/java/com/lashmanager/app/AbstractIntegrationTest.java`
```java
package com.lashmanager.app;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.annotation.Rollback;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

@SpringBootTest
@Transactional
@Rollback
@ActiveProfiles("test")
public abstract class AbstractIntegrationTest {
}
```
**Depende de:** TASK-01
**Pronto quando:** classe compila; `mvn test -Dtest=AbstractIntegrationTest` não lança erro de contexto

---

## Grupo 2 — Testes unitários (paralelo)

### TASK-03 [P] — `CreateClientUseCaseImplTest`
**O quê:** Testes unitários do use case de criação de cliente.
**Onde:** `src/test/java/com/lashmanager/app/application/usecase/CreateClientUseCaseImplTest.java`
**Requisitos cobertos:** TC-03, TC-04

**Estrutura:**
```java
@ExtendWith(MockitoExtension.class)
class CreateClientUseCaseImplTest {

    @Mock ClientRepository clientRepository;
    CreateClientUseCaseImpl useCase;

    @BeforeEach
    void setUp() {
        useCase = new CreateClientUseCaseImpl(clientRepository);
    }
}
```

**Casos:**

`execute_withNewPhone_returnsClientResult`
- `given(clientRepository.existsByPhone("11999999999")).willReturn(false)`
- `given(clientRepository.save(any())).willAnswer(inv -> inv.getArgument(0))`
- Executar com command `("Ana Lima", "11999999999", null, null, null)`
- Assertar: `result.name() == "Ana Lima"`, `result.phone() == "11999999999"`, `result.active() == true`

`execute_withDuplicatePhone_throwsClientAlreadyExistsException`
- `given(clientRepository.existsByPhone(any())).willReturn(true)`
- `assertThatThrownBy(() -> useCase.execute(command)).isInstanceOf(ClientAlreadyExistsException.class)`
- `then(clientRepository).should(never()).save(any())`

**Depende de:** nada
**Pronto quando:** `mvn test -Dtest=CreateClientUseCaseImplTest` → BUILD SUCCESS, 2 testes passando

---

### TASK-04 [P] — `UpdateClientUseCaseImplTest`
**O quê:** Testes unitários do use case de atualização.
**Onde:** `src/test/java/com/lashmanager/app/application/usecase/UpdateClientUseCaseImplTest.java`
**Requisitos cobertos:** TC-05, TC-06, TC-07, TC-08

**Estrutura:**
```java
@ExtendWith(MockitoExtension.class)
class UpdateClientUseCaseImplTest {

    @Mock ClientRepository clientRepository;
    UpdateClientUseCaseImpl useCase;
    UUID clientId;
    Client existingClient;

    @BeforeEach
    void setUp() {
        useCase = new UpdateClientUseCaseImpl(clientRepository);
        clientId = UUID.randomUUID();
        existingClient = Client.builder()
                .id(clientId).name("Ana Lima").phone("11999999999")
                .active(true).createdAt(LocalDateTime.now()).build();
    }
}
```

**Casos:**

`execute_withValidData_returnsUpdatedResult`
- `findById` retorna `Optional.of(existingClient)`
- `existsByPhoneAndIdNot` retorna `false`
- `save` retorna argumento recebido
- Assertar `result.name() == "Novo Nome"`

`execute_withUnknownId_throwsClientNotFoundException`
- `findById` retorna `Optional.empty()`
- `assertThatThrownBy(...).isInstanceOf(ClientNotFoundException.class)`
- `save` nunca chamado

`execute_withPhoneOwnedByAnotherClient_throwsClientAlreadyExistsException`
- `findById` retorna cliente
- `existsByPhoneAndIdNot` retorna `true`
- `save` nunca chamado

`execute_withSamePhoneOfSameClient_succeeds`
- `existsByPhoneAndIdNot` retorna `false` (mesmo telefone, mesmo ID → não conflita)
- `save` chamado uma vez

**Depende de:** nada
**Pronto quando:** 4 testes passando

---

### TASK-05 [P] — `GetClientUseCaseImplTest`
**O quê:** Testes unitários do use case de busca por ID.
**Onde:** `src/test/java/com/lashmanager/app/application/usecase/GetClientUseCaseImplTest.java`
**Requisitos cobertos:** TC-09, TC-10

**Casos:**

`execute_withExistingId_returnsClientResult`
- `findById` retorna `Optional.of(client)` com id, nome, telefone preenchidos
- Assertar `result.id() == client.getId()` e `result.name() == client.getName()`

`execute_withUnknownId_throwsClientNotFoundException`
- `findById` retorna `Optional.empty()`
- `assertThatThrownBy(...).isInstanceOf(ClientNotFoundException.class)`

**Depende de:** nada
**Pronto quando:** 2 testes passando

---

### TASK-06 [P] — `ListClientsUseCaseImplTest`
**O quê:** Testes unitários do use case de listagem.
**Onde:** `src/test/java/com/lashmanager/app/application/usecase/ListClientsUseCaseImplTest.java`
**Requisitos cobertos:** TC-11, TC-12, TC-13

**Casos:**

`execute_withNullSearch_passesEmptyStringToRepository`
- `ArgumentCaptor<String> captor`
- Chamar `useCase.execute(null, null, pageable)`
- `verify(clientRepository).findAll(captor.capture(), any(), any())`
- Assertar `captor.getValue().equals("")`

`execute_withWhitespaceSearch_trimsBeforePassing`
- Chamar com `"  Ana  "`
- Assertar que o repositório recebeu `"Ana"` (sem espaços)

`execute_withValidParams_delegatesAndMapsResult`
- `findAll` retorna `new PageImpl<>(List.of(client))`
- Assertar que o Page retornado tem 1 elemento com os dados do client

**Depende de:** nada
**Pronto quando:** 3 testes passando

---

### TASK-07 [P] — `DeleteClientUseCaseImplTest`
**O quê:** Testes unitários do use case de exclusão.
**Onde:** `src/test/java/com/lashmanager/app/application/usecase/DeleteClientUseCaseImplTest.java`
**Requisitos cobertos:** TC-14, TC-15, TC-16

**Estrutura:**
```java
@ExtendWith(MockitoExtension.class)
class DeleteClientUseCaseImplTest {

    @Mock ClientRepository clientRepository;
    @Mock AppointmentRepository appointmentRepository;
    DeleteClientUseCaseImpl useCase;

    @BeforeEach
    void setUp() {
        useCase = new DeleteClientUseCaseImpl(clientRepository, appointmentRepository);
    }
}
```

**Casos:**

`execute_withNoFutureAppointments_deletesInCorrectOrder`
- `findById` retorna cliente
- `findFutureActiveByClientId` retorna `Collections.emptyList()`
- Usar `InOrder inOrder = inOrder(appointmentRepository, clientRepository)`
- Verificar: `inOrder.verify(appointmentRepository).deleteFutureAppointmentsByClientId(id, any())`
- Verificar: `inOrder.verify(appointmentRepository).unlinkClientFromPastAppointments(id, any())`
- Verificar: `inOrder.verify(clientRepository).deleteById(id)`

`execute_withUnknownId_throwsClientNotFoundException`
- `findById` retorna `Optional.empty()`
- `appointmentRepository` nunca chamado

`execute_withFutureActiveAppointments_throwsHasFutureAppointmentsException`
- `findFutureActiveByClientId` retorna lista com 2 `AppointmentSummary`
- `assertThatThrownBy(...).isInstanceOf(HasFutureAppointmentsException.class)`
- `clientRepository.deleteById` nunca chamado

**Depende de:** nada
**Pronto quando:** 3 testes passando

---

### TASK-08 [P] — `DeactivateClientUseCaseImplTest`
**O quê:** Testes unitários do use case de desativação/reativação.
**Onde:** `src/test/java/com/lashmanager/app/application/usecase/DeactivateClientUseCaseImplTest.java`
**Requisitos cobertos:** TC-17, TC-18, TC-19, TC-20, TC-21, TC-22

**Estrutura:**
```java
@ExtendWith(MockitoExtension.class)
class DeactivateClientUseCaseImplTest {

    @Mock ClientRepository clientRepository;
    @Mock AppointmentRepository appointmentRepository;
    DeactivateClientUseCaseImpl useCase;
    UUID clientId;
    Client activeClient;

    @BeforeEach
    void setUp() {
        useCase = new DeactivateClientUseCaseImpl(clientRepository, appointmentRepository);
        clientId = UUID.randomUUID();
        activeClient = Client.builder()
                .id(clientId).name("Ana Lima").phone("11999999999")
                .active(true).createdAt(LocalDateTime.now()).build();
    }
}
```

**Casos:**

`deactivate_withNoFutureAppointmentsAndForcefalse_savesWithActiveFalse`
- `findById` retorna `activeClient`
- `findFutureActiveByClientId` retorna lista vazia
- `ArgumentCaptor<Client> captor` → `verify(clientRepository).save(captor.capture())`
- Assertar `captor.getValue().isActive() == false`

`deactivate_withFutureAppointmentsAndForceFalse_throwsHasFutureAppointmentsException`
- `findFutureActiveByClientId` retorna lista não-vazia
- `clientRepository.save` nunca chamado

`deactivate_withFutureAppointmentsAndForceTrue_savesIgnoringAppointments`
- `findById` retorna `activeClient`
- `findFutureActiveByClientId` **nunca deve ser chamado** (`verifyNoInteractions(appointmentRepository)`)
- `save` chamado com `active=false`

`deactivate_withUnknownId_throwsClientNotFoundException`
- `findById` retorna `Optional.empty()`

`reactivate_withExistingClient_savesWithActiveTrue`
- `findById` retorna `activeClient` (com `active=false` para ser realista)
- `ArgumentCaptor<Client> captor` → assertar `captor.getValue().isActive() == true`

`reactivate_withUnknownId_throwsClientNotFoundException`
- `findById` retorna `Optional.empty()`

**Depende de:** nada
**Pronto quando:** 6 testes passando

---

## Grupo 3 — Teste de integração

### TASK-09 — `ClientITest`
**O quê:** Teste de integração cobrindo o fluxo CRUD completo e filtros, contra o banco PostgreSQL real.
**Onde:** `src/test/java/com/lashmanager/app/application/usecase/ClientITest.java`
**Requisitos cobertos:** TC-23, TC-24, TC-25, TC-26, TC-27, TC-28

**Estrutura:**
```java
class ClientITest extends AbstractIntegrationTest {

    @Autowired CreateClientUseCase createClientUseCase;
    @Autowired UpdateClientUseCase updateClientUseCase;
    @Autowired GetClientUseCase getClientUseCase;
    @Autowired ListClientsUseCase listClientsUseCase;
    @Autowired DeleteClientUseCase deleteClientUseCase;
    @Autowired DeactivateClientUseCase deactivateClientUseCase;
}
```

**Helper privado:**
```java
private CreateClientUseCase.ClientResult createClient(String name, String phone) {
    return createClientUseCase.execute(
        new CreateClientUseCase.CreateClientCommand(name, phone, null, null, null));
}
```

**Casos:**

`createFindUpdateDelete_fullCrudFlow`
- Criar "Ana Lima" com telefone único
- `getClientUseCase.execute(id)` → assertar nome e `active=true`
- Atualizar para "Ana Costa"
- `getClientUseCase.execute(id)` → assertar `result.name() == "Ana Costa"`
- `deleteClientUseCase.execute(id)`
- `assertThatThrownBy(() -> getClientUseCase.execute(id)).isInstanceOf(ClientNotFoundException.class)`

`create_withDuplicatePhone_throwsClientAlreadyExistsException`
- Criar cliente com telefone X → sucesso
- Tentar criar segundo cliente com mesmo telefone X → `ClientAlreadyExistsException`

`deactivate_persistsActiveFalseInDatabase`
- Criar cliente
- `deactivateClientUseCase.deactivate(id, false)`
- `getClientUseCase.execute(id)` → assertar `result.active() == false`

`reactivate_persistsActiveTrueInDatabase`
- Criar → desativar → reativar
- Buscar → assertar `active == true`

`listBySearch_returnsOnlyMatchingClients`
- Criar "Ana Lima" e "Beatriz Costa" (telefones distintos)
- `listClientsUseCase.execute("Ana", null, PageRequest.of(0, 10))`
- Assertar `page.getTotalElements() >= 1`
- Assertar que todos os resultados têm "Ana" no nome (case-insensitive)

`listByActiveFilter_returnsOnlyInactiveClients`
- Criar "Cliente Ativo" (ativo) e "Cliente Inativo" (criar + desativar)
- `listClientsUseCase.execute("", false, PageRequest.of(0, 10))`
- Assertar que "Cliente Ativo" **não** está nos resultados
- Assertar que "Cliente Inativo" está nos resultados

> **Nota sobre ITest e dados pré-existentes:** O banco de teste é o mesmo de desenvolvimento e pode ter dados do seed (V2). Os asserts de `listBySearch` e `listByActiveFilter` devem ser **parciais** (contém / não contém) e não checar contagem total — `@Rollback` garante que os dados criados no teste somem após cada método, mas dados do seed permanecem.

**Depende de:** TASK-02
**Pronto quando:** `mvn test -Dtest=ClientITest` → BUILD SUCCESS, 6 testes passando com banco Docker ativo

---

## Ordem de Execução

```
TASK-01 (application-test.yml)
    ↓
TASK-02 (AbstractIntegrationTest)
    ↓
[Paralelo] TASK-03 + TASK-04 + TASK-05 + TASK-06 + TASK-07 + TASK-08
    ↓
TASK-09 (ClientITest)
```

**Total: 9 tasks | 28 casos de teste | Unitário: 21 | Integração: 6 | Infra: 2 arquivos**
