# Infraestrutura de Testes — Lash Manager

**Atualizado:** 2026-07-14

## Estado Atual: Cobertura Parcial (2 de 9 módulos)

**Testes implementados:** `lash-core` (1 arquivo) e `lash-clients` (8 arquivos — 7 unitários + 1 de integração)
**Demais módulos** (`lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`, `lash-fichas`, `lash-dashboard`, `lash-app`): frameworks configurados via herança do POM pai, nenhum teste escrito ainda.

---

## Frameworks Configurados

### Backend

| Framework | Versão | Escopo |
|---|---|---|
| JUnit 5 (Jupiter) | 5.11.x | Via `spring-boot-starter-test`, herdado por todos os módulos |
| Mockito | 5.x | Via `spring-boot-starter-test` — testes unitários de use case |
| Spring Boot Test | 3.3.5 | `@SpringBootTest` para testes de integração |
| AssertJ | via `spring-boot-starter-test` | `assertThat` / `assertThatThrownBy` |
| PostgreSQL (real, via Docker) | — | Testes de integração rodam contra `lashmanager-db` (porta 5433), não H2/Testcontainers |

### Frontend

| Framework | Versão | Escopo |
|---|---|---|
| Karma | 6.4.x | Test runner (configurado em `karma.conf.js`) |
| Jasmine | 5.2.x | Assertions e spies |
| Angular Testing | 18.2.x | `TestBed`, `ComponentFixture` |

O diretório `lash-frontend/src/` tem arquivos `.spec.ts` padrão do Angular CLI mas sem conteúdo significativo.

---

## Testes ficam em cada módulo

Cada módulo Maven tem seu próprio `src/test/java/com/lashmanager/{modulo}/` — não existe uma suíte de testes central. `mvn test` no POM pai roda a suíte de todos os módulos em sequência (respeitando a ordem de dependência do reactor); `mvn test -pl {modulo}` roda só um módulo.

```
lash-core/src/test/java/com/lashmanager/core/
└── AbstractIntegrationTest.java      ← @SpringBootTest, @Transactional, @Rollback, @ActiveProfiles("test")

lash-clients/src/test/java/com/lashmanager/clients/
├── AbstractIntegrationTest.java      ← @SpringBootTest(classes = ClientsTestApplication.class)
├── ClientsTestApplication.java       ← @SpringBootApplication de teste, escopo restrito ao módulo
└── application/usecase/
    ├── CreateClientUseCaseImplTest.java     ← unit, @ExtendWith(MockitoExtension.class)
    ├── UpdateClientUseCaseImplTest.java
    ├── GetClientUseCaseImplTest.java
    ├── ListClientsUseCaseImplTest.java
    ├── DeactivateClientUseCaseImplTest.java
    ├── DeleteClientUseCaseImplTest.java
    └── ClientITest.java              ← integração, sufixo "ITest" (não "IT" nem "IntegrationTest")
```

### Por que cada módulo precisa da própria `TestApplication` / contexto Spring

Um módulo isolado (ex.: `lash-clients`) não tem acesso ao `LashManagerApplication` de `lash-app` (que está "acima" dele na árvore de dependências — dependência seria circular). Por isso módulos com teste de integração declaram sua própria classe `@SpringBootApplication` mínima, escopada ao módulo:

```java
// lash-clients/src/test/java/com/lashmanager/clients/ClientsTestApplication.java
@SpringBootApplication(scanBasePackages = "com.lashmanager.clients")
@EnableJpaRepositories(basePackages = "com.lashmanager.clients")
@EntityScan(basePackages = "com.lashmanager.clients")
public class ClientsTestApplication { }
```

`lash-core`, por não depender de nenhum outro módulo, usa `@SpringBootTest` direto (sem classe de configuração própria) — o autoscan já cobre `com.lashmanager.core`.

### Portas cruzando módulo em teste: `@MockBean`

Quando o teste de integração de um módulo depende de uma porta implementada em outro módulo (dependência de infra, não presente no `TestApplication` restrito), ela é substituída por `@MockBean`:

```java
// ClientITest — ClientAppointmentPort é implementada em lash-appointments,
// que lash-clients não tem no classpath de teste (evitaria a dependência circular
// lash-clients → lash-appointments → lash-clients)
@MockBean
ClientAppointmentPort clientAppointmentPort;

@BeforeEach
void setUpAppointmentPortStub() {
    given(clientAppointmentPort.findFutureActiveByClientId(any(), any()))
            .willReturn(Collections.emptyList());
}
```

### `application-test.yml` por módulo

Módulos com teste de integração têm seu próprio `src/test/resources/application-test.yml`, ativado via `@ActiveProfiles("test")`. Aponta para o mesmo banco Docker local (`lashmanager-db`, porta 5433) — não é um banco isolado por módulo:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5433/lashmanager
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
```

---

## Testes com `@DisplayName` em português

**Convenção obrigatória:** toda classe de teste e todo método `@Test` leva `@DisplayName` descrevendo o comportamento em português, no infinitivo/gerúndio como uma frase legível para não-programador ler no relatório de teste.

```java
@DisplayName("Criar cliente")
class CreateClientUseCaseImplTest {

    @Test
    @DisplayName("deve criar cliente quando o telefone ainda não está cadastrado")
    void execute_withNewPhone_returnsClientResult() { ... }

    @Test
    @DisplayName("deve lançar ClientAlreadyExistsException ao cadastrar telefone duplicado")
    void execute_withExistingPhone_throwsClientAlreadyExistsException() { ... }
}
```

```java
@DisplayName("Clientes — integração")
class ClientITest extends AbstractIntegrationTest {

    @Test
    @DisplayName("deve criar, buscar, atualizar e excluir um cliente com sucesso")
    void createFindUpdateDelete_fullCrudFlow() { ... }
}
```

Padrão de nome de método: `metodoOuCenario_condicao_resultadoEsperado` em inglês (padrão de código) — o `@DisplayName` é que carrega a descrição em português para leitura humana. Os dois nunca se substituem: sempre os dois presentes.

---

## Comandos de Teste

```bash
# Backend — reactor inteiro
cd lash-backend
mvn test                          # roda testes de todos os módulos
mvn test -pl lash-clients         # roda só os testes de lash-clients
mvn test -Dtest=NomeDaClasse      # roda uma única classe (dentro do módulo relevante, requer -pl junto se ambíguo)
mvn test -pl lash-clients -Dtest=ClientITest  # um teste específico de um módulo específico

# Frontend
cd lash-frontend
npm test                      # Karma no modo watch
npm test -- --watch=false     # single run (CI)
```

---

## Matriz de Cobertura por Camada

| Camada | Tipo recomendado | Status atual | Observação |
|---|---|---|---|
| Domain models | Unit | ❌ Não tem (nenhum módulo) | Lógica pura, fácil de testar |
| Use cases (application) — lash-core | Unit + Mockito | ⚠️ Parcial (via integração) | Sem teste unitário isolado, só `AbstractIntegrationTest` |
| Use cases (application) — lash-clients | Unit + Mockito | ✅ Completo | 6 unitários (Create/Update/Get/List/Deactivate/Delete) + `ClientITest` de integração |
| Use cases (application) — demais 7 módulos | Unit + Mockito | ❌ Não tem | Maior gap de cobertura atual |
| Repository (infra) | Integração c/ banco real | ⚠️ Só via `ClientITest` (indireto) | Sem `@DataJpaTest` dedicado em nenhum módulo |
| Controller (adapter) | `@WebMvcTest` | ❌ Não tem em nenhum módulo | |
| Angular Services | Jasmine + `HttpClientTestingModule` | ❌ Não tem | |
| NgRx Reducers | Jasmine (pure functions) | ❌ Não tem | Fácil — funções puras |
| NgRx Effects | Jasmine + `provideMockActions` | ❌ Não tem | |
| Components | `TestBed` + Jasmine | ❌ Não tem | |

---

## Estratégia Recomendada (para os módulos ainda sem teste)

Usar `lash-clients` como modelo de referência ao adicionar testes a `lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`, `lash-fichas`, `lash-dashboard`:

1. Um `*UseCaseImplTest` unitário por use case (`@ExtendWith(MockitoExtension.class)`, mock do `*Repository`), com `@DisplayName` em português em classe e métodos
2. Um `{Modulo}TestApplication` (`@SpringBootApplication` escopado ao pacote do módulo) se o módulo ainda não tiver
3. Um `{Entidade}ITest` de integração estendendo `AbstractIntegrationTest` do módulo, cobrindo o fluxo CRUD completo contra o banco real
4. Qualquer porta cruzando módulo (ex.: `*Port` implementada em módulo dependente) vira `@MockBean` no teste de integração, com stub no `@BeforeEach`

```java
// Modelo de teste unitário — replicar por use case
@ExtendWith(MockitoExtension.class)
@DisplayName("Criar {entidade}")
class Create{Entidade}UseCaseImplTest {
    @Mock private {Entidade}Repository repository;
    private Create{Entidade}UseCaseImpl useCase;

    @BeforeEach
    void setUp() { useCase = new Create{Entidade}UseCaseImpl(repository); }

    @Test
    @DisplayName("deve criar {entidade} quando ...")
    void execute_condicao_resultado() { ... }
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
| Unit tests (Mockito, qualquer módulo) | ✅ Sim | Sem estado compartilhado |
| `ClientITest` / integração backend | ⚠️ Não | `@Transactional` + `@Rollback` isola cada método, mas todos os módulos apontam para o **mesmo banco físico** (`lashmanager-db:5433`) — testes de integração de módulos diferentes rodando ao mesmo tempo contra esse banco não têm isolamento entre si (sem Testcontainers, sem schema por teste) |
| `mvn test` no reactor | Sequencial por módulo (Maven Reactor respeita ordem de dependência) | Dentro de um módulo, JUnit 5 por padrão roda classes sequencialmente |
| Karma (frontend) | ✅ Sim | Cada spec roda em browser isolado |

---

## Gate Checks

| Nível | Comando | Quando usar |
|---|---|---|
| Build | `mvn compile` | Após qualquer alteração Java (compila o reactor inteiro) |
| Testes (módulo) | `mvn test -pl {modulo}` | Após alterar um módulo específico |
| Testes (reactor completo) | `mvn test` | Antes de commit que toca múltiplos módulos |
| Build Angular | `npm run build` | Após qualquer alteração TypeScript |
| Testes Angular | `npm test -- --watch=false` | Antes de commit (quando houver testes) |

> ⚠️ Ver CONCERNS.md — 7 dos 9 módulos ainda sem nenhum teste automatizado é a maior preocupação técnica do projeto.
