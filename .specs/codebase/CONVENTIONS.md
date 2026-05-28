# Convenções — Lash Manager

**Atualizado:** 2026-05-22

---

## Backend — Java

### Nomenclatura de classes

| Tipo | Padrão | Exemplo |
|---|---|---|
| Domain Model | `NomeDoModelo` | `Client`, `Service`, `User` |
| Use Case (port/in) | `VerbNomeUseCase` | `CreateClientUseCase` |
| Use Case (impl) | `VerbNomeUseCaseImpl` | `CreateClientUseCaseImpl` |
| Use Case Mapper | `NomeUseCaseMapper` | `ClientUseCaseMapper` |
| Repository (port/out) | `NomeRepository` | `ClientRepository` |
| JPA Entity | `NomeEntity` | `ClientEntity` |
| JPA Repository | `NomeJpaRepository` | `ClientJpaRepository` |
| Repository Impl | `NomeRepositoryImpl` | `ClientRepositoryImpl` |
| Mapper (infra) | `NomeMapper` | `ClientMapper` |
| Controller | `NomeController` | `ClientController` |
| Request DTO | `VerbNomeRequest` | `CreateClientRequest`, `UpdateClientRequest` |
| Response DTO | `NomeResponse` | `ClientResponse`, `ServiceResponse` |
| Exception | `NomeException` | `ClientNotFoundException`, `ServiceNotFoundException` |

### Tipos obrigatórios

| Caso de uso | Tipo Java |
|---|---|
| IDs de entidade | `UUID` |
| Timestamps (data + hora) | `LocalDateTime` |
| Datas | `LocalDate` |
| Valores monetários | `BigDecimal` |
| Duração | `int` (minutos) |

### Injeção de dependências

- Sempre via **construtor** usando `@RequiredArgsConstructor` do Lombok
- Nunca `@Autowired` em campo
- Use cases e repositories injetados como interfaces (nunca implementações)

### Domain model — Lombok obrigatório

```java
@Getter @Builder @NoArgsConstructor @AllArgsConstructor
public class Client {
    private UUID id;
    private String name;
    // ... zero dependência de framework
}
```

### Anotação conflitante `Service`

Quando a classe de domínio `domain.model.Service` está no escopo, a anotação Spring deve ser qualificada:

```java
@org.springframework.stereotype.Service  // qualificado para evitar ambiguidade
@RequiredArgsConstructor
public class CreateServiceUseCaseImpl implements CreateServiceUseCase { ... }
```

### Result records (use cases)

Cada use case define seu próprio `*Result` record (ou reutiliza o de `Create*UseCase`):

```java
public interface CreateClientUseCase {
    record CreateClientCommand(String name, String phone, ...) {}
    record ClientResult(UUID id, String name, String phone, ..., boolean active, String createdAt) {}
    ClientResult execute(CreateClientCommand command);
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

> **Atenção — Hibernate 6:** Nunca passar `null` para parâmetros usados dentro de `CONCAT(...)`. O Hibernate 6 lança exceção mesmo com guarda `:search IS NULL`. Normalizar no use case: `search = (search != null) ? search.trim() : ""`. Com `""`, o `LIKE '%%'` retorna todas as linhas — comportamento desejado para busca vazia.

### GlobalExceptionHandler

Um único `@RestControllerAdvice` centraliza todos os handlers. Formato padrão de resposta de erro:

```java
new ErrorResponse(status, message, LocalDateTime.now().toString())
```

### Ordem de criação de um módulo

1. Domain model → exceções → port/in → port/out
2. Use case implementations + mapper
3. JPA entity → JPA repository → mapper → repository impl
4. DTOs → controller
5. Flyway migration (se schema mudou)

---

## Frontend — Angular / TypeScript

### Estrutura de componentes

Cada componente é composto por **três arquivos obrigatórios**. Nunca usar `template` ou `styles` inline no decorator `@Component`.

```
features/nome/components/nome-componente/
├── nome-componente.component.ts    # decorator + lógica
├── nome-componente.component.html  # template
└── nome-componente.component.css   # estilos
```

```typescript
// ✅ Correto
@Component({
  selector: 'app-client-list',
  standalone: true,
  templateUrl: './client-list.component.html',
  styleUrl: './client-list.component.css',  // singular — Angular 17+
})

// ❌ Nunca
@Component({
  template: `<div>...</div>`,
  styles: [`.class { ... }`],
})
```

> Nota: `styleUrl` (singular) é a forma correta no Angular 17+. `styleUrls` (plural, array) é o padrão legado — não usar.

### Nomenclatura de arquivos

| Tipo | Padrão | Exemplo |
|---|---|---|
| Componente | `nome-kebab.component.ts` | `client-list.component.ts` |
| Service HTTP | `nome.service.ts` | `client.service.ts` |
| NgRx Actions | `nome.actions.ts` | `client.actions.ts` |
| NgRx Reducer | `nome.reducer.ts` | `client.reducer.ts` |
| NgRx Effects | `nome.effects.ts` | `client.effects.ts` |
| NgRx Selectors | `nome.selectors.ts` | `client.selectors.ts` |
| Model/Interface | `nome.model.ts` | `client.model.ts` |
| Rotas | `nome.routes.ts` | `clients.routes.ts` |
| Pipe | `nome.pipe.ts` | `currency-br.pipe.ts`, `duration.pipe.ts` |

### NgRx — prefixo de actions

```typescript
'[FeatureName] Verb Noun'
// Exemplos:
'[Clients] Load Clients'
'[Clients] Create Client'
'[Services] Load Services'
'[Services] Deactivate Service'
```

### NgRx — slice key

O nome do slice no store é o plural lowercase do módulo:

```typescript
provideStore({
  auth: authReducer,
  clients: clientReducer,
  services: serviceReducer,
})
```

### Injeção de dependências — Angular

```typescript
// ✅ Correto
@Injectable({ providedIn: 'root' })
export class ClientService {
  private readonly http = inject(HttpClient);
}

// ✅ Correto em componentes
export class ClientListComponent {
  private store = inject(Store);
  private router = inject(Router);
}

// ❌ Nunca
constructor(private http: HttpClient) {}
```

### Templates — padrões obrigatórios

```html
<!-- ✅ Control flow Angular 17+ -->
@if (loading$ | async) { <spinner /> }
@for (item of items$ | async; track item.id) { ... }
@else { <empty-state /> }

<!-- ✅ Async pipe — nunca subscribe no componente -->
{{ (clients$ | async)?.length }}

<!-- ❌ Nunca usar *ngIf, *ngFor -->
```

### Reactive Forms — padrão

```typescript
form = this.fb.group({
  name: ['', [Validators.required]],
  price: [null, [Validators.required, Validators.min(0.01)]],
});

// Erro inline no template:
@if (form.get('name')?.hasError('required') && form.get('name')?.touched) {
  <mat-error>Nome é obrigatório</mat-error>
}
```

### Roteamento padrão de CRUD

```typescript
export const clientsRoutes: Routes = [
  { path: '', component: ClientListComponent },
  { path: 'novo', component: ClientFormComponent },
  { path: ':id', component: ClientDetailComponent },
  { path: ':id/editar', component: ClientFormComponent },
];
```

### Pipes compartilhados (`shared/pipes/`)

| Pipe | Seletor | Uso |
|---|---|---|
| `CurrencyBrPipe` | `currencyBr` | Formata número como R$ com Intl.NumberFormat |
| `DatePtbrPipe` | `datePtbr` | Formata data em pt-BR, aceita `'short'`\|`'long'` |
| `DurationPipe` | `duration` | Converte minutos em "Xh Ymin" (ex: 90 → "1h 30min") |

Todos standalone — importar diretamente no componente.

### HTTP Service — padrão

```typescript
@Injectable({ providedIn: 'root' })
export class ClientService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = '/api/clients'; // proxy → :8080

  list(search = '', page = 0, size = 20): Observable<PageResponse<Client>> {
    const params = new HttpParams()
      .set('page', page).set('size', size)
      .set('sort', 'name,asc').set('search', search);
    return this.http.get<PageResponse<Client>>(this.baseUrl, { params });
  }
}
```

### Cores e tema

| Token CSS | Valor | Uso |
|---|---|---|
| `--mat-sys-primary` | `#C8A2A2` | Rosé — cor primária |
| `--mat-sys-tertiary` | `#D4AF37` | Dourado — cor de destaque |
| Background | `#FDF6F0` | Nude — fundo da aplicação |
| Texto principal | `#3D3D3D` | Títulos e conteúdo |
