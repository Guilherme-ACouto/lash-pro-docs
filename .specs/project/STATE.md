# Estado Atual do Projeto

**Última atualização:** 2026-05-29 (rev 7)

---

## Fase 1 — Status

### ✅ Concluído

- [x] Estrutura hexagonal backend completa
- [x] Autenticação JWT (login, refresh, recuperação de senha)
- [x] Layout responsivo (bottom nav mobile + sidebar desktop)
- [x] NgRx Auth (login, logout)
- [x] Flyway com V1 (schema completo) e V2 (seed)
- [x] **Módulo de Clientes** — spec + design + tasks + implementação + testado pela usuária ✅
- [x] **Módulo de Serviços** — spec + design + tasks + implementação + testado pela usuária ✅ (CLI-13 / SVC-13 deferred)
- [x] Mapeamento do codebase (7 docs em `.specs/codebase/`)
- [x] **Módulo de Agendamentos** — spec + design + tasks + implementação + testado pela usuária ✅
  - **Rev 1:** Grade semanal estilo Google Calendar (segunda a domingo, 7 colunas simultâneas)
  - Timeline 06:00–19:30, agendamentos como blocos absolutos (top/height calculados por horário/duração)
  - forkJoin para carregar 7 dias em paralelo via `AppointmentService.listByDate()`
  - Click em coluna vazia navega para formulário com data+hora pré-preenchidos
  - Ciclo SCHEDULED→CONFIRMED→COMPLETED→CANCELLED + **NO_SHOW** (novo status)
  - Geração automática de entrada financeira ao completar
  - CLI-13 / SVC-13 implementados com parâmetro `force`
  - V3 Flyway migration: `financial_entry_id` em appointments
  - Correção de bug 500: `ListClientsUseCaseImpl` e `ListServicesUseCaseImpl` passavam `null` ao JPQL; Hibernate 6 falha com null em CONCAT — corrigido passando `""` e removendo guarda `IS NULL`
  - Correção de bug Angular Material M3: `(click)="serviceSelect.open()"` no `mat-label` para abrir mat-select ao clicar no label
  - Formulário carrega serviços via NgRx store (`ServiceActions.loadServices`) em vez de HTTP direto
  - **Rev 2 — redesign da agenda (estilo Google Calendar completo):**
    - Sidebar esquerda com mini calendário, legenda de cores e filtro de profissional
    - Toggle de visualização: Dia / Semana / Mês (mat-button-toggle-group)
    - View mês: grade 6×7 com chips coloridos; "+ N mais" abre modal
    - `AppointmentPopupComponent`: dialog com detalhe do agendamento (clique no chip do mês)
    - `DayAppointmentsModalComponent`: lista todos agendamentos do dia (clique em "+ N mais")
    - Endpoint de range: `GET /api/appointments?date=X&endDate=Y` para carregar o mês inteiro
    - `PATCH /{id}/no-show` endpoint; botão "Não compareceu" no detalhe (status CONFIRMED)
  - **Rev 3 (pós-teste) — bugs corrigidos + modal de agendamentos:**
    - Bug: reducer `deactivateClientSuccess` / `deactivateServiceSuccess` usava `.filter()` (excluía) em vez de `.map()` (atualizava `active`)
    - Bug: clientes/serviços inativos sumiam no reload — backend tinha `active = true` hardcoded em `ListClientsUseCaseImpl` / `ListServicesUseCaseImpl`; corrigido com parâmetro `Boolean active` nullable propagado do controller até o JPQL (`IS NULL OR c.active = :active`)
    - Bug: NPE em `AppointmentController.toResponse()` — `r.clientId().toString()` lançava NPE quando cliente foi desvinculado; corrigido para `r.clientId() != null ? r.clientId().toString() : null`; causava agenda vazia via `forkJoin` que swallowa erros silenciosamente
    - Bug: queries com `JOIN FETCH a.client JOIN FETCH a.service` misturavam inner/left join no Hibernate 6; corrigido para `LEFT JOIN FETCH` em todas as queries de listagem
    - Bug: `@Transactional` em `AppointmentJpaRepository` importado de `jakarta.transaction` — Spring Data JPA requer `org.springframework.transaction.annotation.Transactional`
    - Feature: filtros de status (Todos / Ativos / Inativos) nas telas de Clientes e Serviços
    - Feature: botões Desativar e Reativar sempre visíveis lado a lado (antes do Excluir) nas listagens; desabilitados via `[disabled]` conforme `active`
    - Feature: **`AppointmentsWarningDialogComponent`** — modal compartilhada (em `shared/components/`) que exibe lista de agendamentos futuros com link clicável para cada um; usada em Clientes e Serviços; suporta modo bloqueio (só "Entendi") e modo força (`canForce: true` → botão "Inativar mesmo assim")
    - Feature: V4 Flyway migration — `client_id` em `appointments` tornou-se nullable
    - Feature: `AppointmentSummary` — novo domain record com `id, scheduledDate, scheduledTime, serviceName`
    - Feature: `HasFutureAppointmentsException` agora carrega `List<AppointmentSummary>` em vez de `int count`; `GlobalExceptionHandler` serializa como `appointments` no body 409
    - Feature: lógica de exclusão de cliente refinada — bloqueia se houver futuros SCHEDULED/CONFIRMED; se não houver: apaga agendamentos futuros de qualquer status (libera horário) e desvincula o `client_id` apenas nos agendamentos passados (mantém "Cliente não encontrado" na agenda e preserva entradas financeiras)
    - Feature: `AppointmentMapper` com null-guard em `client` (campo nullable após V4)

### 🔄 Em andamento

- Fase 2 — Dashboard implementado, aguardando testes completos pela usuária
- **Módulo Financeiro** — implementação completa + correções pós-teste (2026-05-26), aguardando testes finais pela usuária
- **Testes — Módulo de Clientes** — spec + tasks + implementação completa (2026-05-29); ✅ todos os 28 testes passando (2026-07-13)

### ✅ Fase 2 — Concluído

- [x] **Dashboard** — spec + design + tasks + implementação completa
  - Backend: `DashboardRepositoryImpl` (10 queries JPQL via EntityManager), `GetDashboardUseCaseImpl`, `DashboardResponse` DTO, `DashboardController` (`GET /api/dashboard?period=TODAY|WEEK|MONTH`)
  - Frontend: `dashboard.model.ts`, `DashboardService`, NgRx store (actions/reducer/effects/selectors), 4 sub-componentes (`KpiCardComponent`, `AppointmentsChartComponent`, `CashFlowChartComponent`, `TodayAppointmentsComponent`), `DashboardComponent` principal integrado
  - Biblioteca de gráficos: `ng-apexcharts@1.11.0` + `apexcharts` (versão fixada por compatibilidade com Angular 18)
  - Sidebar recolhível: toggle com botão circular flutuante na borda, transição CSS 0.25s, tooltips no modo recolhido
  - **Rev 1 (2026-05-29):** `MiniCalendarComponent` removido — não agregava valor suficiente; `selectDaysWithAppointments` removido dos selectors; `TodayAppointmentsComponent` ocupa sozinha a coluna direita

### ⏳ Pendente (Fase 2)

- [ ] Módulo Financeiro
- [ ] Módulo de Estoque

### 🧪 Testes automatizados (iniciado)

- [x] **Testes Módulo de Clientes** — spec + tasks + 9 arquivos implementados
  - Padrão: unitário `*Test.java` (Mockito + AssertJ), integração `*ITest.java` (@SpringBootTest + @Transactional + @Rollback)
  - `AbstractIntegrationTest` criado como base reutilizável para todos os futuros ITests
  - `application-test.yml` em `src/test/resources/` — aponta para PostgreSQL Docker :5433
  - 28 casos: 21 unitários + 6 integração + 1 infra
  - Comandos: `mvn test -Dtest="*Test"` (unitários) / `mvn test -Dtest="*ITest"` (integração, requer banco)

---

## Decisões Registradas

| Data | Decisão | Razão |
|---|---|---|
| 2026-05-22 | `Page<T>` / `Pageable` do Spring usados diretamente em `port/in` | Pragmatismo Fase 1 — domínio estritamente puro seria over-engineering para o porte atual |
| 2026-05-22 | CLI-13 / SVC-13 deferred | Verificação de agendamentos futuros ao desativar cliente/serviço depende do módulo de Agendamentos |
| 2026-05-22 | `@org.springframework.stereotype.Service` qualificado | Conflito de nome com `domain.model.Service` — renomear domain model para `ServiceOffering` está no backlog |
| 2026-05-22 | `DurationPipe` em `shared/pipes/` | Pipe compartilhado entre Serviços e Agendamentos |
| 2026-05-22 | Módulo de Serviços antes de Agendamentos | Agendamentos dependem de Serviços (`service_id` FK) |
| 2026-05-22 | Grade semanal no AppointmentDayComponent em vez de dia único | Usuária solicitou visão simultânea de todos os dias da semana, estilo Google Calendar |
| 2026-05-22 | JPQL de busca: nunca passar `null` para parâmetros usados em `CONCAT` | Hibernate 6 lança exceção mesmo com guarda `IS NULL` — passar `""` e remover a guarda |
| 2026-05-22 | Componentes separados em 3 arquivos (.ts + .html + .css) | Padronização: todos os componentes de Clientes, Serviços e Agendamentos foram migrados |
| 2026-05-22 | `mat-select` do filtro profissional substituído por `<select>` nativo | `MatSelectModule` causava erro de compilação ao ser removido; `<select>` nativo com CSS resolve sem dependência extra |
| 2026-05-22 | `[value]="currentView" (change)="setView($event.value)"` em vez de `[(ngModel)]` no toggle | Two-way binding atualizava o estado antes do `setView()` ser chamado, causando race condition na troca de view |
| 2026-05-22 | `MatDialog` para popup de agendamento no mês (não routerLink) | Chip do mês é pequeno; dialog é mais adequado; routerLink sairia da agenda |
| 2026-05-22 | `DayAppointmentsModalComponent` abre `AppointmentPopupComponent` ao clicar num item | Mantém consistência de UX: sempre dialog para detalhes; a modal de dia fecha antes de abrir o popup |
| 2026-05-22 | `AppointmentsWarningDialogComponent` em `shared/components/` | Modal reutilizada por Clientes e Serviços; evita duplicação; recebe `AppointmentsWarningData` via `MAT_DIALOG_DATA` |
| 2026-05-22 | Exclusão de cliente apaga futuros cancelados, desvincula passados | Futuros SCHEDULED/CONFIRMED bloqueiam a exclusão; futuros CANCELLED são removidos (libera horário); passados mantêm entrada financeira com "Cliente não encontrado" |
| 2026-05-22 | Componente chama service diretamente para deactivate/delete (bypassa effects) | Fluxo de erro 409 com dialog interativo é difícil de modelar em effects puros; sucesso despacha action manualmente |
| 2026-05-22 | `LEFT JOIN FETCH` obrigatório em todas as queries com associações nullable | Hibernate 6 com `JOIN FETCH` em campo nullable lança MultipleBagFetchException ou NPE; `LEFT JOIN FETCH` é seguro em todos os casos |
| 2026-05-23 | `EntityManager` para queries de agregação do dashboard | GROUP BY complexos não se encaixam em Spring Data JPA; JPQL ad-hoc via EM é mais limpo e sem proliferação de interfaces |
| 2026-05-23 | `ng-apexcharts@1.11.0` fixado (não `@latest`) | Versão 2.x exige Angular ≥ 20; projeto usa Angular 18 — fixar na 1.11.0 evita conflito de peer deps |
| 2026-05-23 | Sidebar recolhível: `overflow: visible` no `<aside>` + `.sidebar-inner` com `overflow: hidden` | Botão circular flutuante precisa ultrapassar a borda direita; conteúdo interno (labels) precisa ser clipado na animação — dois níveis de overflow resolvem ambos |
| 2026-05-23 | Tipo `DashboardPeriod` movido para array no TS (não inline no template) | Objeto inline no `*ngFor` do template é inferido como `string`, causando erro de tipo com `DashboardPeriod` |
| 2026-05-23 | `FinancialSummaryRepository` separado do `FinancialEntryRepository` | Queries de agregação com EntityManager ficam isoladas das operações CRUD — mesmo padrão do DashboardRepositoryImpl |
| 2026-05-23 | ManyToOne read-only em `FinancialEntryEntity` (insertable=false, updatable=false) | Permite LEFT JOIN FETCH para resolver nome do cliente; FK controlado pelo UUID `appointmentId` |
| 2026-05-23 | `ConfirmDeleteDialogComponent` inline no `financial.component.ts` | Componente simples de confirmação sem arquivo separado, evita criar arquivo para 10 linhas de template |
| 2026-05-23 | Entrada financeira criada como PAID ao completar agendamento | Pagamento já ocorreu no ato do atendimento; `paymentDate = scheduledDate`; `paymentMethod` coletado via dialog no frontend |
| 2026-05-23 | `PaymentMethodDialogComponent` inline em `appointment-detail.component.ts` | Dialog simples de coleta de forma de pagamento; não justifica arquivo separado |
| 2026-05-26 | `NEXT_MONTH` adicionado ao `FinancialPeriod` | Usuária precisava visualizar despesas cadastradas para o próximo mês |
| 2026-05-26 | `isToggleable(entry: FinancialEntry)` em vez de `isToggleable(status: string)` | Necessário para bloquear toggle de INCOME+PAID — checar só o status não é suficiente |
| 2026-05-26 | Denominador do card Despesas = `expensePending` (`predicted − paid`) | Mostrar o total previsto confunde quando já foi pago — usuária quer ver o que ainda falta pagar |
| 2026-05-26 | `GET /api/financial/categories` endpoint independente | Filtro de categoria carregava apenas categorias da aba/período atual; endpoint retorna todas as categorias distintas do banco |
| 2026-05-26 | `isIncomePaid` desabilita todo o form e exibe locked-banner | Receitas já recebidas são imutáveis por regra de negócio — qualquer edição comprometeria o histórico financeiro |
| 2026-05-26 | Date pickers De/Até aparecem quando período = CUSTOM | Alternativa ao campo de texto livre para datas customizadas — MatDatepicker garante formato correto |
| 2026-05-29 | `MiniCalendarComponent` removido do Dashboard | Não agregava valor suficiente para a usuária; simplifica a tela e elimina código morto (`selectDaysWithAppointments`, `onCalendarDayClick`, `Router`) |
| 2026-07-14 | Use cases voltam a usar `@Service` direto (não `{Modulo}Config`/`@Bean`) | Durante o refactor multi-módulo, a IA trocou o wiring de use cases por classes `{Modulo}Config` sem ter sido pedido; usuária pediu para reverter |
| 2026-07-14 | `domain.model.Service` renomeado para `ServiceOffering` (RENAME-01) | Resolve o conflito de nome com `@org.springframework.stereotype.Service` de vez, sem precisar de anotação qualificada em `lash-services` |

---

## Blockers / Riscos Ativos

| ID | Severidade | Descrição | Ação necessária |
|---|---|---|---|
| B01 | 🔴 | Zero cobertura de testes | Implementar gradualmente, começar por use cases |
| B02 | 🔴 | `JWT_SECRET` hardcoded no `application.yml` | Remover default antes do primeiro deploy em produção |

---

## Deferred (implementar no módulo correto)

| ID | Descrição | Quando resolver |
|---|---|---|
| CLI-13 | ✅ Implementado — modal com lista de agendamentos futuros ao desativar; exclusão bloqueia se futuros ativos, apaga futuros cancelados, desvincula passados | Concluído |
| SVC-13 | ✅ Implementado — modal com lista de agendamentos futuros ao desativar; exclusão bloqueia se futuros ativos | Concluído |
| RENAME-01 | ✅ Implementado (2026-07-14) — `domain.model.Service` renomeado para `ServiceOffering`; use cases de `lash-services` e `lash-appointments` voltaram a usar `@Service` sem qualificação | Concluído |
| CQRS-01 | Implementar padrão CQRS nos endpoints — separar operações de leitura (Query) das de escrita (Command) | Após refatoração multi-módulo |

---

## Credenciais de Desenvolvimento

| Campo | Valor |
|---|---|
| Email | admin@lashmanager.com |
| Senha | admin123 |

## Ambiente Local

| Serviço | Comando |
|---|---|
| Banco | `docker start lashmanager-db` |
| Backend | `sdk use java 21.0.5-zulu && mvn spring-boot:run` |
| Frontend | `nvm use 20 && npm start` |

---

## Preferências

- Respostas sempre em português brasileiro
- Cada módulo = pasta própria com `spec.md` + `design.md` + `tasks.md` antes de implementar
- Módulos são chamados de "fases"
