# Infraestrutura de Testes — Lash Manager

**Atualizado:** 2026-05-22

## Estado Atual: Zero Cobertura

**Testes implementados:** nenhum  
**Status:** frameworks configurados, nenhum teste escrito

---

## Frameworks Configurados

### Backend

| Framework | Versão | Escopo |
|---|---|---|
| JUnit 5 (Jupiter) | 5.11.x | Via `spring-boot-starter-test` |
| Mockito | 5.x | Via `spring-boot-starter-test` |
| Spring Boot Test | 3.3.5 | `@SpringBootTest`, `@WebMvcTest`, etc. |
| H2 (in-memory) | — | Não configurado — banco de teste não definido |

O diretório `lash-backend/src/test/java/` existe mas está vazio.

### Frontend

| Framework | Versão | Escopo |
|---|---|---|
| Karma | 6.4.x | Test runner (configurado em `karma.conf.js`) |
| Jasmine | 5.2.x | Assertions e spies |
| Angular Testing | 18.2.x | `TestBed`, `ComponentFixture` |

O diretório `lash-frontend/src/` tem arquivos `.spec.ts` padrão do Angular CLI mas sem conteúdo significativo.

---

## Comandos de Teste

```bash
# Backend
cd lash-backend
mvn test                      # roda todos os testes (atualmente nenhum)
mvn test -Dtest=NomeDaClasse  # roda um único teste

# Frontend
cd lash-frontend
npm test                      # Karma no modo watch
npm test -- --watch=false     # single run (CI)
```

---

## Matriz de Cobertura por Camada

| Camada | Tipo recomendado | Status atual | Observação |
|---|---|---|---|
| Domain models | Unit | ❌ Não tem | Lógica pura, fácil de testar |
| Use cases (application) | Unit + Mockito | ❌ Não tem | Mockar `*Repository` interfaces |
| Repository (infra) | Integração c/ banco | ❌ Não tem | Requer Testcontainers ou H2 |
| Controller (adapter) | `@WebMvcTest` | ❌ Não tem | Mockar use cases |
| Angular Services | Jasmine + `HttpClientTestingModule` | ❌ Não tem | |
| NgRx Reducers | Jasmine (pure functions) | ❌ Não tem | Fácil — funções puras |
| NgRx Effects | Jasmine + `provideMockActions` | ❌ Não tem | |
| Components | `TestBed` + Jasmine | ❌ Não tem | |

---

## Estratégia Recomendada (quando implementar)

### Backend — prioridade

1. **Use cases**: mockar `*Repository` com Mockito, testar lógica de negócio pura
2. **Controllers**: `@WebMvcTest` mockar use cases, testar serialização/status HTTP
3. **Repository**: `@DataJpaTest` com Testcontainers PostgreSQL (manter paridade com prod)

```java
// Exemplo estrutura use case test
@ExtendWith(MockitoExtension.class)
class CreateClientUseCaseImplTest {
    @Mock ClientRepository clientRepository;
    @InjectMocks CreateClientUseCaseImpl useCase;

    @Test void shouldCreateClient() { ... }
    @Test void shouldThrowWhenPhoneExists() { ... }
}
```

### Frontend — prioridade

1. **NgRx Reducers**: testar estado inicial e cada `on()` handler (funções puras, zero setup)
2. **Effects**: `provideMockActions` + `HttpClientTestingModule`
3. **Components**: `TestBed` com store MockStore

---

## Paralelismo

| Tipo | Seguro em paralelo? | Motivo |
|---|---|---|
| Unit tests (backend) | ✅ Sim | Sem estado compartilhado (Mockito) |
| `@WebMvcTest` (backend) | ✅ Sim | Spring context isolado por classe |
| `@DataJpaTest` (backend) | ⚠️ Depende | Requer Testcontainers ou H2 em memória separados por teste |
| Karma (frontend) | ✅ Sim | Cada spec roda em browser isolado |

---

## Gate Checks

| Nível | Comando | Quando usar |
|---|---|---|
| Build | `mvn compile` | Após qualquer alteração Java |
| Testes | `mvn test` | Antes de commit (quando houver testes) |
| Build Angular | `npm run build` | Após qualquer alteração TypeScript |
| Testes Angular | `npm test -- --watch=false` | Antes de commit (quando houver testes) |

> ⚠️ Ver CONCERNS.md — ausência de testes é a maior preocupação técnica do projeto.
