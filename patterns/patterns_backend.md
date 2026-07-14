# Padrões de Backend

O backend é um **projeto Maven multi-módulo**: 9 módulos independentes (`lash-core`,
`lash-clients`, `lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`,
`lash-fichas`, `lash-dashboard`, `lash-app`), cada um hexagonal por dentro. Estes padrões
usam `lash-clients` como referência principal (é o módulo com testes completos) e
`lash-appointments` para exemplos de porta cross-módulo.

Ver `.specs/codebase/ARCHITECTURE.md` para a árvore de dependências entre módulos e
`.specs/codebase/STRUCTURE.md` para a árvore de diretórios completa.

---

## Criando um módulo novo

1. `{modulo}/pom.xml` — registrar como `<module>` no `pom.xml` raiz e em `<dependencyManagement>`
2. Domain Model (`domain/model`)
3. Exceções de domínio (`domain/exception`) — estendendo `DomainException`/`BusinessException` de `lash-core`
4. Port In — Use Case interfaces (`domain/port/in`)
5. Port Out — Repository interface, e portas cross-módulo se precisar ler dado de outro módulo (`domain/port/out`)
6. Use Case implementations (`application/usecase`) — anotadas `@Service` diretamente
7. JPA Entity (`infrastructure/persistence/entity`)
8. JPA Repository interface — Spring Data (`infrastructure/persistence/repository`)
9. Mapper Entity ↔ Domain (`infrastructure/persistence/mapper`)
10. Repository Implementation (`infrastructure/persistence/repository`)
11. DTOs de Request/Response (`adapter/web/dto`)
12. Controller (`adapter/web/controller`)
13. Flyway migration, no range de versão reservado do módulo (`V{modulo}00__...`)
14. Adicionar o módulo como dependência de `lash-app/pom.xml`
15. Registrar os handlers de exceção do módulo em `lash-app/adapter/web/GlobalExceptionHandler.java`
16. Testes: `AbstractIntegrationTest` + `{Modulo}TestApplication` (se o módulo tiver teste de integração) + `*UseCaseImplTest` por use case

### `pom.xml` do módulo

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.lashmanager</groupId>
        <artifactId>lash-backend</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>lash-clients</artifactId>
    <name>lash-clients</name>
    <description>Módulo de gestão de clientes</description>

    <dependencies>
        <dependency>
            <groupId>com.lashmanager</groupId>
            <artifactId>lash-core</artifactId>
        </dependency>
        <!-- outros módulos internos dos quais este depende, se houver -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Módulos internos declarados como dependência **sem** `<version>` — a versão vem do
`<dependencyManagement>` do POM pai, que usa `${project.version}` (todos os módulos
sobem de versão juntos).

---

## Domain Model

Zero dependência de framework — sem `@Entity`, sem Spring, sem Jakarta Validation.

```java
package com.lashmanager.clients.domain.model;

@Getter
@Builder(toBuilder = true)
@NoArgsConstructor
@AllArgsConstructor
public class Client {
    private UUID id;
    private String name;
    private String phone;
    private String email;
    private LocalDate birthDate;
    private String notes;
    private boolean active;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

`@Builder(toBuilder = true)` — usar `toBuilder = true` quando o use case de update
precisa clonar o objeto alterando só alguns campos (`client.toBuilder().name(novo).build()`).

---

## Exceções de domínio

Toda exceção de módulo estende `DomainException` (violação de regra/validação) ou
`BusinessException` (conflito de estado de negócio) — ambas em `lash-core`:

```java
// lash-core/domain/exception/DomainException.java
public class DomainException extends RuntimeException {
    public DomainException(String message) { super(message); }
}

// lash-core/domain/exception/BusinessException.java
public class BusinessException extends DomainException {
    public BusinessException(String message) { super(message); }
}
```

```java
// lash-clients/domain/exception/ClientNotFoundException.java
public class ClientNotFoundException extends BusinessException {
    public ClientNotFoundException(UUID id) {
        super("Cliente não encontrado: " + id);
    }
}

// lash-clients/domain/exception/ClientAlreadyExistsException.java
public class ClientAlreadyExistsException extends BusinessException {
    public ClientAlreadyExistsException(String phone) {
        super("Já existe um cliente com o telefone: " + phone);
    }
}
```

---

## Use Case — Port In

```java
package com.lashmanager.clients.domain.port.in;

public interface CreateClientUseCase {

    record CreateClientCommand(
            String name,
            String phone,
            String email,
            LocalDate birthDate,
            String notes
    ) {}

    record ClientResult(
            UUID id,
            String name,
            String phone,
            String email,
            String birthDate,   // datas viram String no Result — já formatadas para a resposta
            String notes,
            boolean active,
            String createdAt
    ) {}

    ClientResult execute(CreateClientCommand command);
}
```

Cada use case define seu próprio `Command`/`Result` como record aninhado. Use cases
irmãos (ex.: `UpdateClientUseCase`) frequentemente reaproveitam o `ClientResult` de
`CreateClientUseCase` em vez de duplicar o record.

---

## Use Case — Port Out (Repository)

```java
package com.lashmanager.clients.domain.port.out;

public interface ClientRepository {
    Client save(Client client);
    Optional<Client> findById(UUID id);
    Page<Client> findAll(String search, Boolean active, Pageable pageable);
    boolean existsByPhone(String phone);
    boolean existsByPhoneAndIdNot(String phone, UUID id);
    void deleteById(UUID id);
}
```

## Use Case — Port Out cross-módulo

Quando um módulo precisa de dado de outro módulo, a porta é declarada no módulo
**consumidor** (não no dono do dado) e implementada como adapter no módulo dono:

```java
// declarada em lash-clients/domain/port/out/ClientAppointmentPort.java
// (lash-clients NÃO depende de lash-appointments no pom.xml)
public interface ClientAppointmentPort {
    List<AppointmentSummary> findFutureActiveByClientId(UUID clientId, LocalDate from);
    void deleteFutureAppointmentsByClientId(UUID clientId, LocalDate from);
    void unlinkClientFromPastAppointments(UUID clientId, LocalDate from);
}
```

```java
// implementada em lash-appointments/infrastructure/adapter/ClientAppointmentPortImpl.java
// (lash-appointments DEPENDE de lash-clients — a direção da dependência é a inversa
// da direção da porta: quem implementa importa a interface do outro módulo)
@Component
@RequiredArgsConstructor
public class ClientAppointmentPortImpl implements ClientAppointmentPort {

    private final AppointmentRepository appointmentRepository;

    @Override
    public List<AppointmentSummary> findFutureActiveByClientId(UUID clientId, LocalDate from) {
        return appointmentRepository.findFutureActiveByClientId(clientId, from);
    }
    // ...
}
```

O bean só existe em runtime porque `lash-app` traz os dois módulos no classpath — em um
módulo isolado (testes de `lash-clients`, por exemplo), essa porta vira `@MockBean`
(ver seção de testes mais abaixo e `.specs/codebase/TESTING.md`).

**Antes de criar uma porta cross-módulo nova**, checar a árvore de dependências em
ARCHITECTURE.md — só é possível declarar `port/out` para consumir um módulo que já
depende de você (nunca o inverso, senão vira dependência circular no reactor Maven;
ver C10 em `.specs/codebase/CONCERNS.md` para um caso real disso acontecendo entre
`lash-services` e `lash-appointments`).

---

## Use Case — Implementation

`@Service` direto na classe, com `@RequiredArgsConstructor` — vira bean automaticamente
via component scan (`lash-app` faz `@SpringBootApplication(scanBasePackages = "com.lashmanager")`,
então o scan cobre todos os módulos). **Não** existe uma classe de configuração
intermediária para registrar use cases — cada módulo é responsável por anotar sua
própria implementação.

```java
package com.lashmanager.clients.application.usecase;

@Service
@RequiredArgsConstructor
public class CreateClientUseCaseImpl implements CreateClientUseCase {

    private final ClientRepository clientRepository;

    @Override
    public ClientResult execute(CreateClientCommand command) {
        if (clientRepository.existsByPhone(command.phone())) {
            throw new ClientAlreadyExistsException(command.phone());
        }

        Client client = Client.builder()
                .id(UUID.randomUUID())
                .name(command.name())
                .phone(command.phone())
                .email(command.email())
                .birthDate(command.birthDate())
                .notes(command.notes())
                .active(true)
                .createdAt(LocalDateTime.now())
                .updatedAt(LocalDateTime.now())
                .build();

        return ClientUseCaseMapper.toResult(clientRepository.save(client));
    }
}
```

> **Nota:** o domain model de `lash-services` se chama `ServiceOffering` (não `Service`)
> justamente para não colidir com `@org.springframework.stereotype.Service` — por isso
> `*ServiceUseCaseImpl` também usa `@Service` normal, sem qualificação, como qualquer
> outro módulo.

### `*UseCaseMapper` — domain model → Result record

```java
package com.lashmanager.clients.application.usecase;

public class ClientUseCaseMapper {

    private ClientUseCaseMapper() {}

    public static CreateClientUseCase.ClientResult toResult(Client client) {
        return new CreateClientUseCase.ClientResult(
                client.getId(),
                client.getName(),
                client.getPhone(),
                client.getEmail(),
                client.getBirthDate() != null ? client.getBirthDate().toString() : null,
                client.getNotes(),
                client.isActive(),
                client.getCreatedAt() != null ? client.getCreatedAt().toString() : null
        );
    }
}
```

---

## JPA Entity

```java
package com.lashmanager.clients.infrastructure.persistence.entity;

@Entity
@Table(name = "clients")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ClientEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false, unique = true)
    private String phone;

    private String email;

    @Column(name = "birth_date")
    private LocalDate birthDate;

    @Column(columnDefinition = "TEXT")
    private String notes;

    @Column(nullable = false)
    private boolean active;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
}
```

---

## Mapper (infra) — Entity ↔ Domain

```java
package com.lashmanager.clients.infrastructure.persistence.mapper;

@Component
public class ClientMapper {

    public Client toDomain(ClientEntity entity) {
        return Client.builder()
                .id(entity.getId())
                .name(entity.getName())
                .phone(entity.getPhone())
                .email(entity.getEmail())
                .birthDate(entity.getBirthDate())
                .notes(entity.getNotes())
                .active(entity.isActive())
                .createdAt(entity.getCreatedAt())
                .updatedAt(entity.getUpdatedAt())
                .build();
    }

    public ClientEntity toEntity(Client domain) {
        return ClientEntity.builder()
                .id(domain.getId())
                .name(domain.getName())
                .phone(domain.getPhone())
                .email(domain.getEmail())
                .birthDate(domain.getBirthDate())
                .notes(domain.getNotes())
                .active(domain.isActive())
                .createdAt(domain.getCreatedAt())
                .updatedAt(domain.getUpdatedAt())
                .build();
    }
}
```

---

## Repository Implementation

```java
package com.lashmanager.clients.infrastructure.persistence.repository;

@Repository
@RequiredArgsConstructor
public class ClientRepositoryImpl implements ClientRepository {

    private final ClientJpaRepository jpaRepository;
    private final ClientMapper mapper;

    @Override
    public Client save(Client client) {
        return mapper.toDomain(jpaRepository.save(mapper.toEntity(client)));
    }

    @Override
    public Optional<Client> findById(UUID id) {
        return jpaRepository.findById(id).map(mapper::toDomain);
    }

    @Override
    public Page<Client> findAll(String search, Boolean active, Pageable pageable) {
        return jpaRepository.findAllFiltered(search, active, pageable).map(mapper::toDomain);
    }

    @Override
    public boolean existsByPhone(String phone) {
        return jpaRepository.existsByPhone(phone);
    }

    @Override
    public void deleteById(UUID id) {
        jpaRepository.deleteById(id);
    }
}
```

### JPQL — padrão de busca

```java
@Query("""
    SELECT c FROM ClientEntity c
    WHERE (:activeOnly = false OR c.active = true)
    AND (LOWER(c.name) LIKE LOWER(CONCAT('%', :search, '%'))
         OR c.phone LIKE CONCAT('%', :search, '%'))
    ORDER BY c.name ASC
    """)
Page<ClientEntity> search(@Param("search") String search, @Param("activeOnly") boolean activeOnly, Pageable pageable);
```

> **Atenção — Hibernate 6:** nunca passar `null` para parâmetro usado dentro de
> `CONCAT(...)` — lança exceção mesmo com guarda `:search IS NULL`. Normalizar no use
> case: `search = (search != null) ? search.trim() : ""`. Com `""`, `LIKE '%%'` retorna
> todas as linhas — comportamento desejado para busca vazia.

---

## DTOs (Request / Response)

```java
package com.lashmanager.clients.adapter.web.dto;

public record CreateClientRequest(
        @NotBlank @Size(min = 2, max = 100) String name,
        @NotBlank @Size(min = 10, max = 20) String phone,
        String email,
        LocalDate birthDate,
        @Size(max = 500) String notes
) {}
```

```java
public record ClientResponse(
        UUID id,
        String name,
        String phone,
        String email,
        String birthDate,
        String notes,
        boolean active,
        String createdAt
) {
    public static ClientResponse from(CreateClientUseCase.ClientResult result) {
        return new ClientResponse(
                result.id(), result.name(), result.phone(), result.email(),
                result.birthDate(), result.notes(), result.active(), result.createdAt()
        );
    }
}
```

O `Response` tem um factory estático `from(Result)` — sem `DtoMapper` separado como
classe própria; a conversão fica no próprio record.

---

## Controller

```java
package com.lashmanager.clients.adapter.web.controller;

@RestController
@RequestMapping("/api/clients")
@RequiredArgsConstructor
public class ClientController {

    private final CreateClientUseCase createClientUseCase;
    private final UpdateClientUseCase updateClientUseCase;
    private final GetClientUseCase getClientUseCase;
    private final ListClientsUseCase listClientsUseCase;
    private final DeleteClientUseCase deleteClientUseCase;
    private final DeactivateClientUseCase deactivateClientUseCase;

    @PostMapping
    public ResponseEntity<ClientResponse> create(@Valid @RequestBody CreateClientRequest request) {
        var result = createClientUseCase.execute(new CreateClientUseCase.CreateClientCommand(
                request.name(), request.phone(), request.email(), request.birthDate(), request.notes()
        ));
        return ResponseEntity
                .created(URI.create("/api/clients/" + result.id()))
                .body(ClientResponse.from(result));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ClientResponse> getById(@PathVariable UUID id) {
        return ResponseEntity.ok(ClientResponse.from(getClientUseCase.execute(id)));
    }

    @GetMapping
    public ResponseEntity<Page<ClientResponse>> list(
            @RequestParam(required = false) String search,
            @RequestParam(required = false) Boolean active,
            @PageableDefault(size = 20) Pageable pageable
    ) {
        return ResponseEntity.ok(
                listClientsUseCase.execute(search, active, pageable).map(ClientResponse::from)
        );
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable UUID id) {
        deleteClientUseCase.execute(id);
        return ResponseEntity.noContent().build();
    }

    @PatchMapping("/{id}/deactivate")
    public ResponseEntity<Void> deactivate(
            @PathVariable UUID id,
            @RequestParam(defaultValue = "false") boolean force
    ) {
        deactivateClientUseCase.deactivate(id, force);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Tratamento de Erros — `GlobalExceptionHandler` centralizado em `lash-app`

Diferente de um `@RestControllerAdvice` por módulo, existe **um único**
`GlobalExceptionHandler` no sistema, em `lash-app/adapter/web/`. Ele importa as
exceções de todos os módulos e agrupa os handlers por comentário de seção:

```java
package com.lashmanager.app.adapter.web;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // ── Clientes ──────────────────────────────────────────────────────────────

    @ExceptionHandler(ClientNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleClientNotFound(ClientNotFoundException ex) {
        return err(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(ClientAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> handleClientAlreadyExists(ClientAlreadyExistsException ex) {
        return err(HttpStatus.CONFLICT, ex.getMessage());
    }

    // ── Genéricos ─────────────────────────────────────────────────────────────

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex) {
        return err(HttpStatus.UNPROCESSABLE_ENTITY, ex.getMessage());
    }

    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ErrorResponse> handleDomain(DomainException ex) {
        return err(HttpStatus.BAD_REQUEST, ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        log.error("Erro interno não tratado: ", ex);
        return err(HttpStatus.INTERNAL_SERVER_ERROR, "Erro interno do servidor");
    }

    private ResponseEntity<ErrorResponse> err(HttpStatus status, String message) {
        return ResponseEntity.status(status)
                .body(new ErrorResponse(status.value(), message, LocalDateTime.now().toString()));
    }
}
```

**Ao adicionar uma exceção de domínio nova**, sempre registrar o handler específico
aqui (não deixar cair no genérico `BusinessException`/`DomainException`, a menos que o
status HTTP genérico já seja o correto) — sob o comentário de seção do módulo dono.

---

## Testes

Ver `.specs/codebase/TESTING.md` para o padrão completo (por módulo, `@DisplayName`
em português, `{Modulo}TestApplication`, `@MockBean` para portas cross-módulo). Resumo:

```java
// Unit — mocka o Repository
@ExtendWith(MockitoExtension.class)
@DisplayName("Criar cliente")
class CreateClientUseCaseImplTest {
    @Mock private ClientRepository clientRepository;
    private CreateClientUseCaseImpl useCase;

    @BeforeEach
    void setUp() { useCase = new CreateClientUseCaseImpl(clientRepository); }

    @Test
    @DisplayName("deve criar cliente quando o telefone ainda não está cadastrado")
    void execute_withNewPhone_returnsClientResult() { ... }
}
```

```java
// Integração — banco real, dentro do módulo
@DisplayName("Clientes — integração")
class ClientITest extends AbstractIntegrationTest {

    @MockBean
    ClientAppointmentPort clientAppointmentPort;   // porta implementada em outro módulo

    @Test
    @DisplayName("deve criar, buscar, atualizar e excluir um cliente com sucesso")
    void createFindUpdateDelete_fullCrudFlow() { ... }
}
```
