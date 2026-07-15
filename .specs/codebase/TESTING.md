# Infraestrutura de Testes — Lash Manager

**Atualizado:** 2026-07-15

## Estado Atual: Cobertura Parcial (2 de 9 módulos), 26 testes passando

**Testes implementados:** `lash-core` (4 arquivos — `RefreshToken`/`JwtService` unit + `RegistrationFlowITest` + `CommandInterceptorITest`) e `lash-clients` (8 arquivos — 7 unitários + 1 de integração)
**Demais módulos** (`lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`, `lash-fichas`, `lash-dashboard`, `lash-app`): frameworks configurados via herança do POM pai, nenhum teste escrito ainda.
**Resultado da última execução completa** (`mvn test` no reactor, 2026-07-15): `BUILD SUCCESS`, 26 testes, 0 falhas.

---

## Banco de teste separado do banco de dev (introduzido com a multi-tenancy)

**Antes:** os testes de integração apontavam pro mesmo banco de dev (`lashmanager`), reaproveitando tabelas já migradas por uma execução anterior do app real — funcionava "por sorte", sem rodar Flyway no próprio contexto de teste.

**Agora:** existe um banco Postgres separado, `lashmanager_test`, na mesma instância Docker (`lashmanager-db`, porta 5433) — precisa ser criado manualmente uma vez:

```bash
docker exec lashmanager-db psql -U postgres -c "CREATE DATABASE lashmanager_test;"
```

`application-test.yml` (`lash-core`, `lash-clients`) aponta pra esse banco. Sendo um banco vazio, cada módulo com teste de integração agora **precisa** rodar o próprio Flyway pra migrar o schema `public` — por isso `flyway-core`/`flyway-database-postgresql` foram adicionados como dependência `test`-scope em `lash-core/pom.xml` e `lash-clients/pom.xml` (antes só existiam em `lash-app`, runtime).

## Schema de tenant fixo para os testes (alinhado com o padrão da Pontta)

Depois de consultar como a Pontta testa multi-tenancy, o `AbstractIntegrationTest` (`lash-core`, `lash-clients`) usa um **schema de tenant fixo** (`TEST_TENANT_ID`, um UUID constante) reaproveitado por toda a suíte — não um schema novo por teste:

```java
@SpringBootTest(classes = CoreTestApplication.class)  // ou ClientsTestApplication, em lash-clients
@Transactional
@Rollback
@ActiveProfiles("test")
public abstract class AbstractIntegrationTest {

    protected static final UUID TEST_TENANT_ID = UUID.fromString("00000000-0000-0000-0000-000000000001");

    @BeforeEach
    void setUpTenantSchema() {
        schemaProvisionerPort.provision(TEST_TENANT_ID);  // idempotente — CREATE SCHEMA IF NOT EXISTS
        TenantContext.setCurrentTenant(tenantSchemaNaming.schemaNameFor(TEST_TENANT_ID));
    }

    @AfterEach
    void clearTenantSchema() {
        TenantContext.clear();
    }
}
```

O schema fica no banco de teste entre execuções (não é dropado) — é barato de recriar (idempotente) e evita a complexidade de coordenar `@BeforeAll` estático com injeção de dependência do Spring.

**Decisão explícita: sem teste automatizado de isolamento real entre dois schemas.** Um `TenantIsolationITest` chegou a ser escrito (criava um segundo schema de tenant pra provar que dado de um não vaza pro outro) mas foi **removido** — replicando a Pontta, que também não tem esse teste: eles evitam o problema rodando quase tudo contra um schema fixo (igual ao nosso `TEST_TENANT_ID`) e mockando a camada que cria schema de verdade nos poucos testes que tocam esse fluxo. Risco aceito conscientemente — o mecanismo de isolamento (Hibernate `search_path`) foi implementado e é exercitado manualmente em ambiente real, só não tem prova automatizada de "tenant A não vê dado do tenant B".

## Provisionamento de schema mockado em testes de fluxo completo

`RegistrationFlowITest` (fluxo registro → ativação → login) usa `@MockBean` no `SchemaProvisionerPort` — o teste verifica que a orquestração (`ActivateAccountUseCaseImpl`) chama `provision(tenantId)` com o id certo, **sem** rodar `CREATE SCHEMA`/Liquibase de verdade a cada execução. Mesma escolha da Pontta (`SignatureCreatorTest`/`SignatureInitializerTest` lá também mockam essa camada) — evita acumular um schema Postgres novo (com nome derivado de um `tenantId` aleatório) no banco de teste a cada rodada.

```java
@MockBean
private SchemaProvisionerPort schemaProvisionerPort;

// ...
Mockito.verify(schemaProvisionerPort).provision(activationResult.tenantId());
```

---

## Frameworks Configurados

### Backend

| Framework | Versão | Escopo |
|---|---|---|
| JUnit 5 (Jupiter) | 5.11.x | Via `spring-boot-starter-test`, herdado por todos os módulos |
| Mockito | 5.x | Via `spring-boot-starter-test` — testes unitários de use case, `@MockBean` em testes de integração |
| Spring Boot Test | 3.3.5 | `@SpringBootTest` para testes de integração |
| AssertJ | via `spring-boot-starter-test` | `assertThat` / `assertThatThrownBy` |
| PostgreSQL (real, via Docker) | — | Testes de integração rodam contra `lashmanager_test` (porta 5433), banco separado do de dev — não H2/Testcontainers |
| Flyway (test-scope) | 10.x | `lash-core`/`lash-clients` — migra o schema `public` do banco de teste isolado |
| Liquibase | 4.27.0 | Via `SchemaProvisionerPort` (mockado ou real) — migra o schema de tenant fixo |

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
├── CoreTestApplication.java          ← @SpringBootApplication de teste, escopo com.lashmanager.core
│                                        (lash-core não depende de nenhum outro módulo — precisa da
│                                        própria config mínima, não tem acesso ao LashManagerApplication)
├── AbstractIntegrationTest.java      ← @SpringBootTest(classes=CoreTestApplication), schema fixo
├── RegistrationFlowITest.java        ← registro → ativação (schema mockado) → login
└── CommandInterceptorITest.java      ← valida validação/auditoria do CommandInterceptor (3 cenários)

lash-clients/src/test/java/com/lashmanager/clients/
├── AbstractIntegrationTest.java      ← @SpringBootTest(classes = ClientsTestApplication.class), schema fixo
├── ClientsTestApplication.java       ← @SpringBootApplication de teste, escopo restrito ao módulo
└── application/usecase/
    ├── CreateClientUseCaseImplTest.java     ← unit, @ExtendWith(MockitoExtension.class)
    ├── UpdateClientUseCaseImplTest.java      ← mocka ClientRepository (escrita)
    ├── GetClientUseCaseImplTest.java        ← mocka ClientQueryRepository (leitura)
    ├── ListClientsUseCaseImplTest.java      ← mocka ClientQueryRepository (leitura)
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

`lash-core`, por não depender de nenhum outro módulo, também precisa da própria (`CoreTestApplication`, escopo `com.lashmanager.core`) — **não** usa `@SpringBootTest` "puro" sem classes, porque isso exigiria uma `@SpringBootConfiguration` em algum pacote ancestral, que não existe dentro do próprio `lash-core`. (Esse gap existia desde antes da Fase C, mas só foi descoberto ao escrever o primeiro teste de integração de `lash-core` que de fato precisava subir o contexto.)

### Testes que mockam ClientQueryRepository vs ClientRepository

Depois da separação Command/Query (Fase F/E do `refactor-backend`), `GetClientUseCaseImplTest`/`ListClientsUseCaseImplTest` mockam `ClientQueryRepository` (porta de leitura); `CreateClientUseCaseImplTest`/`UpdateClientUseCaseImplTest`/etc. continuam mockando `ClientRepository` (porta de escrita). Reflete a mesma separação existente em produção.

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

Mesmo mecanismo é usado pra mockar `SchemaProvisionerPort` em `RegistrationFlowITest` (ver seção acima) — não é uma dependência cross-módulo, mas o motivo de usar `@MockBean` é análogo: isolar o teste de uma operação cara/externa (nesse caso, `CREATE SCHEMA` real).

### `application-test.yml` por módulo

Módulos com teste de integração têm seu próprio `src/test/resources/application-test.yml`, ativado via `@ActiveProfiles("test")`. Aponta pro banco de teste **separado** (`lashmanager_test`, não o banco de dev):

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5433/lashmanager_test
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    out-of-order: true    # mesma flag do application.yml de produção — ver STACK.md
  liquibase:
    enabled: false         # idem — Liquibase só via API programática (SchemaProvisionerImpl)
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
# Backend — pré-requisito único: criar o banco de teste (uma vez só)
docker exec lashmanager-db psql -U postgres -c "CREATE DATABASE lashmanager_test;"

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
| Use cases (application) — lash-core | Unit + integração | ⚠️ Parcial | `RegistrationFlowITest`/`CommandInterceptorITest` cobrem o fluxo novo via integração; sem unit test isolado do `RegisterUseCaseImpl`/`ActivateAccountUseCaseImpl` ainda |
| Use cases (application) — lash-clients | Unit + Mockito | ✅ Completo | 6 unitários (Create/Update/Get/List/Deactivate/Delete) + `ClientITest` de integração |
| Use cases (application) — demais 6 módulos de negócio | Unit + Mockito | ❌ Não tem | Maior gap de cobertura atual (inclui as novas classes Command/ApplicationService dessa fase) |
| CommandInterceptor (AOP) | Integração | ✅ Coberto | `CommandInterceptorITest` — validação, sucesso, falha de negócio |
| Repository (infra) | Integração c/ banco real | ⚠️ Só via `ClientITest` (indireto) | Sem `@DataJpaTest` dedicado em nenhum módulo |
| Controller (adapter) | `@WebMvcTest` | ❌ Não tem em nenhum módulo | |
| Isolamento real entre schemas de tenant | Integração | ❌ Não tem (decisão explícita) | Ver seção acima — risco aceito, replicando a Pontta |
| Angular Services | Jasmine + `HttpClientTestingModule` | ❌ Não tem | |
| NgRx Reducers | Jasmine (pure functions) | ❌ Não tem | Fácil — funções puras |
| NgRx Effects | Jasmine + `provideMockActions` | ❌ Não tem | |
| Components | `TestBed` + Jasmine | ❌ Não tem | |

---

## Estratégia Recomendada (para os módulos ainda sem teste)

Usar `lash-clients` como modelo de referência ao adicionar testes a `lash-services`, `lash-appointments`, `lash-finance`, `lash-stock`, `lash-fichas`, `lash-dashboard` — agora também cobrindo as classes `*Command`/`*ApplicationService` novas de cada módulo:

1. Um `*UseCaseImplTest` unitário por use case (`@ExtendWith(MockitoExtension.class)`, mock do `*Repository` ou `*QueryRepository` conforme a operação), com `@DisplayName` em português em classe e métodos
2. Um `{Modulo}TestApplication` (`@SpringBootApplication` escopado ao pacote do módulo) se o módulo ainda não tiver
3. Um `{Entidade}ITest` de integração estendendo `AbstractIntegrationTest` do módulo, cobrindo o fluxo CRUD completo contra o banco de teste isolado (schema fixo, ver seção acima)
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
| `ClientITest`/`RegistrationFlowITest`/`CommandInterceptorITest` (integração) | ⚠️ Não | `@Transactional` + `@Rollback` isola cada método, mas todos os módulos apontam para o **mesmo banco de teste físico** (`lashmanager_test:5433`) e reaproveitam o **mesmo schema de tenant fixo** — testes de integração de módulos diferentes rodando ao mesmo tempo contra esse banco não têm isolamento entre si |
| `mvn test` no reactor | Sequencial por módulo (Maven Reactor respeita ordem de dependência) | Dentro de um módulo, JUnit 5 por padrão roda classes sequencialmente |
| Karma (frontend) | ✅ Sim | Cada spec roda em browser isolado |

---

## Gate Checks

| Nível | Comando | Quando usar |
|---|---|---|
| Build | `mvn compile` | Após qualquer alteração Java (compila o reactor inteiro) |
| Testes (módulo) | `mvn test -pl {modulo}` | Após alterar um módulo específico |
| Testes (reactor completo) | `mvn test` | Antes de commit que toca múltiplos módulos — **requer o banco `lashmanager_test` já criado** |
| Build Angular | `npm run build` | Após qualquer alteração TypeScript |
| Testes Angular | `npm test -- --watch=false` | Antes de commit (quando houver testes) |

> ⚠️ Ver CONCERNS.md — 7 dos 9 módulos ainda sem nenhum teste automatizado é a maior preocupação técnica do projeto, agora agravada pelo volume de código novo (Command/ApplicationService/Query) sem cobertura nesses módulos.
