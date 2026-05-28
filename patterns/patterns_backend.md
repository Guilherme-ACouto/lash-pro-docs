# Padrões de Backend

## Fluxo Completo de um Módulo

Para cada módulo novo, sempre criar nesta ordem:

1. Domain Model (`domain/model`)
2. Exceções de domínio (`domain/exception`)
3. Port In — Use Case interfaces (`domain/port/in`)
4. Port Out — Repository interface (`domain/port/out`)
5. Use Case implementations (`application/usecase`)
6. JPA Entity (`infrastructure/persistence/entity`)
7. JPA Repository interface (`infrastructure/persistence/repository`)
8. Mapper (`infrastructure/persistence/mapper`)
9. Repository Implementation (`infrastructure/persistence/repository`)
10. DTOs (`adapter/web/dto`)
11. DTO Mapper (`adapter/web/mapper`)
12. Controller (`adapter/web/controller`)
13. Flyway migration (se necessário)

---

## Domain Model

```java
// Sem anotações JPA, sem Spring
@Builder
@Getter
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

---

## Use Case — Port In

```java
public interface CreateClientUseCase {
    ClientResult execute(CreateClientCommand command);

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
        String email
    ) {}
}
```

---

## Use Case — Implementation

```java
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

        Client saved = clientRepository.save(client);
        return new ClientResult(saved.getId(), saved.getName(), saved.getPhone(), saved.getEmail());
    }
}
```

---

## JPA Entity

```java
@Entity
@Table(name = "clients")
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
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

    private String notes;

    @Column(nullable = false)
    private boolean active;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    void prePersist() {
        LocalDateTime now = LocalDateTime.now();
        if (createdAt == null) createdAt = now;
        if (updatedAt == null) updatedAt = now;
    }

    @PreUpdate
    void preUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

---

## Mapper

```java
@Component
public class ClientMapper {

    public Client toDomain(ClientEntity entity) {
        if (entity == null) return null;
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
        if (domain == null) return null;
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

## Controller

```java
@RestController
@RequestMapping("/api/clients")
@RequiredArgsConstructor
public class ClientController {

    private final CreateClientUseCase createClientUseCase;
    private final ListClientsUseCase listClientsUseCase;
    private final FindClientByIdUseCase findClientByIdUseCase;
    private final UpdateClientUseCase updateClientUseCase;
    private final DeleteClientUseCase deleteClientUseCase;
    private final ClientDtoMapper mapper;

    @PostMapping
    public ResponseEntity<ClientResponse> create(@Valid @RequestBody CreateClientRequest request) {
        var result = createClientUseCase.execute(mapper.toCommand(request));
        return ResponseEntity.status(HttpStatus.CREATED).body(mapper.toResponse(result));
    }

    @GetMapping
    public ResponseEntity<Page<ClientResponse>> list(
            @RequestParam(required = false) String search,
            Pageable pageable) {
        return ResponseEntity.ok(listClientsUseCase.execute(search, pageable).map(mapper::toResponse));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ClientResponse> findById(@PathVariable UUID id) {
        return ResponseEntity.ok(mapper.toResponse(findClientByIdUseCase.execute(id)));
    }

    @PutMapping("/{id}")
    public ResponseEntity<ClientResponse> update(@PathVariable UUID id,
                                                  @Valid @RequestBody UpdateClientRequest request) {
        return ResponseEntity.ok(mapper.toResponse(updateClientUseCase.execute(id, mapper.toCommand(request))));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable UUID id) {
        deleteClientUseCase.execute(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Tratamento de Erros

Todas as exceções de domínio estendem uma base:

```java
public abstract class DomainException extends RuntimeException {
    private final int statusCode;
    public DomainException(String message, int statusCode) {
        super(message);
        this.statusCode = statusCode;
    }
}
```

O `GlobalExceptionHandler` captura e formata a resposta de erro padronizada.
