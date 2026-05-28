# Clientes — Especificação

## Problem Statement

A profissional registra e gerencia seus clientes hoje via WhatsApp e caderno.
Não tem busca rápida, nem histórico organizado, e perde tempo procurando dados
de clientes na hora de agendar. O módulo de Clientes resolve isso centralizando
o cadastro e permitindo encontrar qualquer cliente em segundos.

## Goals

- [ ] Profissional consegue cadastrar um cliente completo em menos de 1 minuto
- [ ] Profissional consegue encontrar qualquer cliente pelo nome ou telefone em segundos
- [ ] Profissional consegue ver e editar os dados de um cliente existente
- [ ] Profissional consegue desativar clientes inativos sem perder o histórico

## Out of Scope

| Feature | Razão |
|---|---|
| Histórico de serviços por cliente | Depende do módulo de Agendamentos (Fase 1 — posterior) |
| Importação em lote (CSV/Excel) | Fase 2 |
| Foto do cliente | Fase 2 |
| Portal de auto-cadastro pelo cliente | Fase 3 |
| Envio de mensagem WhatsApp pelo sistema | Fase 3 |

---

## User Stories

### P1: Cadastrar Cliente ⭐ MVP

**User Story**: Como profissional, quero cadastrar um novo cliente com seus dados de contato para ter todas as informações organizadas no sistema.

**Por que P1**: Sem cadastro não há nenhuma outra operação possível.

**Acceptance Criteria**:

1. WHEN acesso `/clients/novo` THEN sistema SHALL exibir formulário com campos: Nome (obrigatório), Telefone/WhatsApp (obrigatório), E-mail (opcional), Data de nascimento (opcional), Observações (opcional)
2. WHEN submeto o formulário com dados válidos THEN sistema SHALL salvar o cliente, exibir mensagem de sucesso e redirecionar para a listagem
3. WHEN submeto sem preencher Nome THEN sistema SHALL exibir erro inline "Nome é obrigatório" sem submeter
4. WHEN submeto sem preencher Telefone THEN sistema SHALL exibir erro inline "Telefone é obrigatório" sem submeter
5. WHEN tento cadastrar um cliente com telefone já existente no sistema THEN sistema SHALL exibir erro "Já existe um cliente com este telefone"

**Independent Test**: Abrir `/clients/novo`, preencher nome + telefone e salvar — cliente aparece na listagem.

---

### P1: Listar e Buscar Clientes ⭐ MVP

**User Story**: Como profissional, quero ver todos os meus clientes e buscar rapidamente por nome ou telefone para encontrar quem preciso sem rolar longas listas.

**Por que P1**: A listagem é o ponto de entrada de todo o módulo.

**Acceptance Criteria**:

1. WHEN acesso `/clients` THEN sistema SHALL exibir lista de clientes ativos ordenados por nome, com nome, telefone e e-mail visíveis
2. WHEN digito no campo de busca THEN sistema SHALL filtrar a lista em tempo real (debounce 300ms) mostrando apenas clientes cujo nome ou telefone contenha o texto digitado
3. WHEN a busca não retorna resultados THEN sistema SHALL exibir mensagem "Nenhum cliente encontrado"
4. WHEN há mais de 20 clientes THEN sistema SHALL paginar a lista com controles de navegação
5. WHEN a lista está carregando THEN sistema SHALL exibir skeleton loader ou spinner

**Independent Test**: Cadastrar 3 clientes com nomes distintos, buscar por parte do nome do segundo — só ele aparece.

---

### P1: Visualizar e Editar Cliente ⭐ MVP

**User Story**: Como profissional, quero abrir o perfil de um cliente para ver seus dados completos e corrigir informações desatualizadas.

**Por que P1**: Dados erram na digitação — edição é necessária desde o MVP.

**Acceptance Criteria**:

1. WHEN clico em um cliente na listagem THEN sistema SHALL navegar para `/clients/:id` exibindo todos os campos do cadastro
2. WHEN clico em "Editar" THEN sistema SHALL exibir o formulário pré-preenchido com os dados atuais
3. WHEN salvo alterações válidas THEN sistema SHALL atualizar o cliente, exibir confirmação e voltar para o perfil
4. WHEN cancelo a edição THEN sistema SHALL descartar alterações e voltar para o perfil sem salvar
5. WHEN o cliente não existe (ID inválido) THEN sistema SHALL exibir página de erro 404 com link para voltar à listagem

**Independent Test**: Editar o telefone de um cliente cadastrado — o novo telefone aparece no perfil e na listagem.

---

### P2: Desativar / Reativar Cliente

**User Story**: Como profissional, quero desativar clientes que não me procuram mais para manter a listagem principal limpa, sem perder o histórico.

**Por que P2**: Importante para manter a base organizada, mas não bloqueia o MVP.

**Acceptance Criteria**:

1. WHEN clico em "Desativar" na listagem de um cliente ativo THEN sistema SHALL tentar desativar imediatamente (sem confirm do browser)
2. WHEN confirmo a desativação e cliente não tem agendamentos futuros THEN sistema SHALL marcar o cliente como inativo (permanece visível na listagem com filtro "Inativos")
3. WHEN o cliente tem agendamentos futuros ativos (SCHEDULED/CONFIRMED) THEN sistema SHALL exibir modal com lista de agendamentos e opção "Inativar mesmo assim" ou "Cancelar"
4. WHEN profissional clica "Inativar mesmo assim" THEN sistema SHALL desativar o cliente ignorando os agendamentos futuros
5. WHEN clico em "Reativar" em um cliente inativo THEN sistema SHALL marcá-lo como ativo e ele volta a aparecer na listagem padrão
6. WHEN uso o filtro de status (Todos / Ativos / Inativos) THEN sistema SHALL exibir apenas clientes no status selecionado

**Independent Test**: Desativar um cliente — ele permanece na listagem com visual de inativo, aparece com filtro "Inativos", e pode ser reativado.

---

### P2: Excluir Cliente

**User Story**: Como profissional, quero excluir permanentemente um cliente cadastrado por engano, sem perder o histórico financeiro de atendimentos já realizados.

**Por que P2**: Raro mas necessário para correção de cadastros errados.

**Acceptance Criteria**:

1. WHEN clico em "Excluir" na listagem THEN sistema SHALL pedir confirmação nativa do browser antes de prosseguir
2. WHEN cliente tem agendamentos futuros ativos (SCHEDULED/CONFIRMED) THEN sistema SHALL exibir modal bloqueante com a lista de agendamentos e mensagem pedindo para cancelá-los antes
3. WHEN cliente não tem agendamentos futuros ativos THEN sistema SHALL excluir o cliente, apagar agendamentos futuros cancelados (liberando o horário) e desvincular o nome dos agendamentos passados (que passam a exibir "Cliente não encontrado")
4. WHEN cliente excluído tinha entradas financeiras THEN sistema SHALL manter todas as entradas financeiras intactas
5. WHEN agendamentos passados são desvinculados THEN sistema SHALL exibir "Cliente não encontrado" no lugar do nome do cliente nesses agendamentos

**Independent Test**: Excluir um cliente sem agendamentos futuros — ele some da listagem; agendamentos passados dele continuam na agenda com "Cliente não encontrado".

---

## Edge Cases

- WHEN o campo de busca é limpo THEN sistema SHALL restaurar a listagem completa
- WHEN submeto o formulário com e-mail em formato inválido THEN sistema SHALL exibir "E-mail inválido"
- WHEN a data de nascimento é futura THEN sistema SHALL exibir "Data de nascimento inválida"
- WHEN o servidor retorna erro ao salvar THEN sistema SHALL exibir mensagem de erro genérica sem perder os dados do formulário
- WHEN navego para `/clients` sem estar autenticada THEN sistema SHALL redirecionar para o login

---

## Requirement Traceability

| Requirement ID | Story | Status |
|---|---|---|
| CLI-01 | P1: Cadastrar — formulário com campos obrigatórios | Verified |
| CLI-02 | P1: Cadastrar — salvar e redirecionar | Verified |
| CLI-03 | P1: Cadastrar — validação inline de campos | Verified |
| CLI-04 | P1: Cadastrar — unicidade do telefone | Verified |
| CLI-05 | P1: Listar — exibição ordenada por nome | Verified |
| CLI-06 | P1: Listar — busca em tempo real por nome/telefone | Verified |
| CLI-07 | P1: Listar — estado vazio e paginação | Verified |
| CLI-08 | P1: Visualizar — perfil completo do cliente | Verified |
| CLI-09 | P1: Editar — formulário pré-preenchido e salvar | Verified |
| CLI-10 | P1: Editar — cancelar sem salvar | Verified |
| CLI-11 | P2: Desativar — atualiza status sem remover da lista | Verified |
| CLI-12 | P2: Reativar — volta à listagem padrão | Verified |
| CLI-13 | P2: Desativar — modal com lista de agendamentos futuros; suporte a force | Verified |
| CLI-14 | P2: Filtro de status (Todos / Ativos / Inativos) na listagem | Verified |
| CLI-15 | P2: Excluir — bloqueia se futuros ativos, exibe modal bloqueante | Verified |
| CLI-16 | P2: Excluir — apaga futuros cancelados, desvincula passados, mantém financeiro | Verified |

**Cobertura:** 16 requisitos, 16 verificados ✅

---

## Success Criteria

- [ ] Profissional consegue cadastrar um cliente do zero sem consultar manual
- [ ] Busca retorna resultado correto em menos de 500ms com até 500 clientes
- [ ] Zero perda de dados ao cancelar edição
- [ ] Cliente desativado não aparece em nenhum fluxo de agendamento futuro
