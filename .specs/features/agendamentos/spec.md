# Agendamentos — Especificação

## Problem Statement

A profissional gerencia sua agenda hoje via WhatsApp e anotações. Não tem visão clara do dia, perde tempo verificando conflitos manualmente e não tem integração com o financeiro. O módulo de Agendamentos resolve isso com uma agenda visual do dia (06:00–20:00), bloqueio automático de conflitos e geração automática de receita ao concluir um atendimento.

## Goals

- [ ] Profissional consegue ver toda a agenda do dia em uma tela
- [ ] Profissional consegue criar um agendamento em menos de 1 minuto clicando no horário desejado
- [ ] Sistema bloqueia automaticamente horários conflitantes
- [ ] Ao marcar atendimento como realizado, entrada financeira é gerada automaticamente

## Out of Scope

| Feature | Razão |
|---|---|
| Notificações via WhatsApp/e-mail | Fase 3 |
| Portal de auto-agendamento pelo cliente | Fase 3 |
| Agenda mensal | Fase 2 |
| Recorrência de agendamentos | Fase 2 |
| Múltiplas profissionais | Fase 3 |
| Pagamento integrado | Fase 2 (Financeiro) |

---

## User Stories

### P1: Ver Agenda Semanal ⭐ MVP

**User Story**: Como profissional, quero ver todos os agendamentos da semana em uma grade visual simultânea para ter controle total da minha agenda sem precisar navegar dia por dia.

**Por que P1**: Sem visão da agenda não há como operar o módulo.

**Acceptance Criteria**:

1. WHEN acesso `/appointments` THEN sistema SHALL exibir a grade semanal (segunda a domingo) com timeline das 06:00 às 19:30, com os agendamentos de todos os dias visíveis simultaneamente
2. WHEN há agendamentos na semana THEN sistema SHALL exibir em cada bloco: nome do cliente, nome do serviço, horário de início e fim, com cor diferente por status
3. WHEN clico em `<` ou `>` na navegação THEN sistema SHALL exibir a semana anterior ou seguinte
4. WHEN não há agendamentos em um dia THEN sistema SHALL exibir a coluna vazia (slots clicáveis para criar)
5. WHEN a agenda está carregando THEN sistema SHALL exibir spinner centralizado
6. WHEN clico em uma área vazia da coluna de um dia THEN sistema SHALL calcular o horário pelo posicionamento do clique e abrir formulário com data+hora pré-preenchidos

**Cores por status:**
- `SCHEDULED` → azul/cinza (agendado)
- `CONFIRMED` → dourado `#D4AF37` (confirmado)
- `COMPLETED` → verde (realizado)
- `CANCELLED` → vermelho claro / riscado (cancelado)

**Independent Test**: Criar 3 agendamentos em dias distintos — cada dia exibe apenas os seus próprios agendamentos.

---

### P1: Criar Agendamento ⭐ MVP

**User Story**: Como profissional, quero criar um agendamento clicando no horário desejado ou pelo botão "Novo agendamento" para registrar rapidamente um novo atendimento.

**Por que P1**: Criar é a operação central do módulo.

**Acceptance Criteria**:

1. WHEN clico em um horário vazio na timeline THEN sistema SHALL abrir formulário com data e horário pré-preenchidos
2. WHEN clico em "Novo agendamento" THEN sistema SHALL abrir formulário com data atual pré-preenchida
3. WHEN o formulário abre THEN sistema SHALL exibir campos: Cliente (obrigatório, busca por nome), Serviço (obrigatório, lista de serviços ativos), Data (obrigatório), Horário de início (obrigatório), Duração em minutos (obrigatório, pré-preenchido ao selecionar serviço), Observações (opcional)
4. WHEN seleciono um serviço THEN sistema SHALL preencher automaticamente o campo Duração com a duração padrão do serviço
5. WHEN submeto com dados válidos THEN sistema SHALL salvar o agendamento com status `SCHEDULED`, exibir confirmação e redirecionar para a agenda do dia
6. WHEN o horário escolhido conflita com outro agendamento existente THEN sistema SHALL exibir erro "Horário indisponível — já existe um agendamento neste período" sem salvar
7. WHEN o horário de início é anterior às 06:00 THEN sistema SHALL exibir erro "Horário fora do expediente (06:00–20:00)"
8. WHEN o horário de início + duração ultrapassa 20:00 THEN sistema SHALL exibir erro "Atendimento ultrapassaria o fim do expediente (20:00)"
9. WHEN submeto sem preencher Cliente THEN sistema SHALL exibir erro inline "Cliente é obrigatório"
10. WHEN submeto sem preencher Serviço THEN sistema SHALL exibir erro inline "Serviço é obrigatório"

**Independent Test**: Criar agendamento às 10:00 com duração 90min — aparece na agenda das 10:00 às 11:30.

---

### P1: Visualizar Agendamento ⭐ MVP

**User Story**: Como profissional, quero abrir um agendamento para ver todos os detalhes e executar ações de status.

**Por que P1**: A tela de detalhe é o hub de todas as ações.

**Acceptance Criteria**:

1. WHEN clico em um agendamento na agenda THEN sistema SHALL navegar para `/appointments/:id` exibindo: cliente (com link para perfil), serviço, data, horário de início e fim, duração, status, observações, data de criação
2. WHEN o status é `SCHEDULED` THEN sistema SHALL exibir botões: "Confirmar" e "Cancelar"
3. WHEN o status é `CONFIRMED` THEN sistema SHALL exibir botões: "Marcar como realizado" e "Cancelar"
4. WHEN o status é `COMPLETED` THEN sistema SHALL exibir apenas informações (sem botões de ação) e link para a entrada financeira gerada
5. WHEN o status é `CANCELLED` THEN sistema SHALL exibir apenas informações com indicação visual de cancelado
6. WHEN o agendamento não existe (ID inválido) THEN sistema SHALL exibir 404 com link para voltar à agenda

**Independent Test**: Abrir detalhe de agendamento `SCHEDULED` — botões "Confirmar" e "Cancelar" visíveis.

---

### P1: Gerenciar Status do Agendamento ⭐ MVP

**User Story**: Como profissional, quero atualizar o status do agendamento conforme o atendimento avança, e ao concluir que o sistema gere automaticamente a receita no financeiro.

**Por que P1**: O ciclo de vida do agendamento e a integração com o financeiro são o coração do produto.

**Acceptance Criteria**:

1. WHEN clico em "Confirmar" num agendamento `SCHEDULED` THEN sistema SHALL atualizar status para `CONFIRMED` e exibir confirmação
2. WHEN clico em "Marcar como realizado" num agendamento `CONFIRMED` THEN sistema SHALL pedir confirmação antes de agir
3. WHEN confirmo "Marcar como realizado" THEN sistema SHALL atualizar status para `COMPLETED`, mudar a cor do bloco para verde e criar automaticamente uma entrada financeira com: tipo `INCOME`, descrição "{Serviço} — {Cliente}", valor igual ao preço do serviço, data de vencimento igual à data do agendamento, status `PENDING`, `appointment_id` vinculado
4. WHEN clico em "Cancelar" em qualquer agendamento ativo THEN sistema SHALL pedir confirmação antes de agir
5. WHEN confirmo o cancelamento THEN sistema SHALL atualizar status para `CANCELLED`
6. WHEN tento confirmar ou realizar um agendamento `CANCELLED` THEN sistema SHALL ignorar a ação (botões não exibidos)

**Independent Test**: Criar agendamento → confirmar → marcar realizado → verificar que entrada financeira aparece em `/financial`.

---

### P2: Editar Agendamento

**User Story**: Como profissional, quero editar um agendamento para corrigir data, horário ou observações quando necessário.

**Por que P2**: Erros acontecem, mas edição não bloqueia o MVP.

**Acceptance Criteria**:

1. WHEN clico em "Editar" num agendamento com status `SCHEDULED` ou `CONFIRMED` THEN sistema SHALL exibir formulário pré-preenchido
2. WHEN salvo alterações válidas THEN sistema SHALL revalidar conflitos de horário com o novo horário (excluindo o próprio agendamento) e salvar se não houver conflito
3. WHEN o novo horário conflita THEN sistema SHALL exibir erro "Horário indisponível" sem salvar
4. WHEN tento editar agendamento `COMPLETED` ou `CANCELLED` THEN sistema SHALL não exibir botão de editar

**Independent Test**: Mudar horário de agendamento de 14:00 para 15:00 — agenda atualiza corretamente.

---

### P2: Implementar CLI-13 e SVC-13

**User Story**: Como profissional, quero ser avisada ao tentar desativar uma cliente ou serviço que tem agendamentos futuros, para não criar inconsistências na agenda.

**Por que P2**: Resolve os deferreds CLI-13 e SVC-13, agora que o módulo de Agendamentos existe.

**Acceptance Criteria**:

1. WHEN tento desativar uma cliente que tem agendamentos com status `SCHEDULED` ou `CONFIRMED` em datas futuras THEN sistema SHALL exibir alerta "Esta cliente tem X agendamento(s) futuro(s). Desativar mesmo assim?"
2. WHEN confirmo a desativação com agendamentos futuros THEN sistema SHALL desativar a cliente (agendamentos existentes não são cancelados automaticamente)
3. WHEN tento desativar um serviço que tem agendamentos com status `SCHEDULED` ou `CONFIRMED` em datas futuras THEN sistema SHALL exibir alerta "Este serviço tem X agendamento(s) futuro(s). Desativar mesmo assim?"
4. WHEN confirmo a desativação do serviço THEN sistema SHALL desativar o serviço

**Independent Test**: Criar agendamento futuro para cliente ativa → tentar desativar a cliente → alerta aparece com contagem correta.

---

## Edge Cases

- WHEN seleciono serviço inativo no formulário THEN sistema NÃO SHALL exibir serviços inativos na lista de seleção
- WHEN busco cliente inativo no formulário THEN sistema NÃO SHALL exibir clientes inativos na busca
- WHEN dois agendamentos terminam e começam no mesmo minuto (ex: 10:00-12:00 e 12:00-14:00) THEN sistema SHALL considerar como sem conflito
- WHEN navego para data passada THEN sistema SHALL exibir a agenda histórica normalmente (somente leitura não é obrigatório — profissional pode criar agendamentos retroativos)
- WHEN o servidor retorna erro ao salvar THEN sistema SHALL exibir mensagem genérica sem perder os dados do formulário
- WHEN navego para `/appointments` sem estar autenticada THEN sistema SHALL redirecionar para o login

---

## Requirement Traceability

| Requirement ID | Story | Status |
|---|---|---|
| AGD-01 | P1: Agenda semanal — grade 7 colunas, timeline 06:00-19:30, todos os dias simultâneos | Implemented |
| AGD-02 | P1: Agenda semanal — navegação por semana (prev/next) | Implemented |
| AGD-03 | P1: Agenda semanal — click em coluna vazia abre formulário com data+hora | Implemented |
| AGD-04 | P1: Agenda semanal — botão "Novo" com data atual | Implemented |
| AGD-05 | P1: Agenda semanal — cores por status nos blocos absolutos | Implemented |
| AGD-06 | P1: Criar — formulário com todos os campos obrigatórios | Implemented |
| AGD-07 | P1: Criar — duração auto-preenchida ao selecionar serviço | Implemented |
| AGD-08 | P1: Criar — bloqueio de conflito de horário | Implemented |
| AGD-09 | P1: Criar — validação de horário de expediente (06:00-20:00) | Implemented |
| AGD-10 | P1: Visualizar — detalhe completo com botões por status | Implemented |
| AGD-11 | P1: Status — SCHEDULED → CONFIRMED | Implemented |
| AGD-12 | P1: Status — CONFIRMED → COMPLETED + gera entrada financeira | Implemented |
| AGD-13 | P1: Status — cancelar com confirmação | Implemented |
| AGD-14 | P2: Editar — formulário pré-preenchido com revalidação | Implemented |
| AGD-15 | P2: CLI-13 — alerta ao desativar cliente com agendamentos futuros | Implemented |
| AGD-16 | P2: SVC-13 — alerta ao desativar serviço com agendamentos futuros | Implemented |

**Cobertura:** 16 requisitos, 16 implementados ✅ (aguardando verificação)

---

## Success Criteria

- [ ] Profissional consegue criar um agendamento em menos de 1 minuto
- [ ] Sistema bloqueia 100% dos conflitos de horário
- [ ] Ao concluir atendimento, entrada financeira criada automaticamente sem ação adicional
- [ ] Agenda do dia carrega em menos de 1 segundo
- [ ] CLI-13 e SVC-13 resolvidos e funcionando
