# Serviços — Especificação

## Problem Statement

A profissional oferece um catálogo fixo de serviços (Volume Russo, Fio a Fio, Manutenção, Retirada) com preços e durações definidos. Sem um cadastro centralizado, qualquer alteração de preço ou adição de novo serviço precisa ser feita diretamente no banco. O módulo de Serviços resolve isso permitindo gerenciar o catálogo pelo sistema, e fornece os dados de duração e preço que o módulo de Agendamentos consome.

## Goals

- [ ] Profissional consegue cadastrar e editar serviços com nome, preço e duração
- [ ] Profissional consegue visualizar o catálogo completo em segundos
- [ ] Profissional consegue desativar serviços que não oferece mais sem perder histórico
- [ ] Módulo de Agendamentos consegue consultar serviços ativos com duração e preço

## Out of Scope

| Feature | Razão |
|---|---|
| Categorias de serviços | Fase 2 — catálogo ainda pequeno |
| Preço por cliente (desconto/pacote) | Fase 2 |
| Foto do serviço | Fase 2 |
| Pacotes (combinação de serviços) | Fase 3 |
| Histórico de alterações de preço | Fase 3 |

---

## User Stories

### P1: Cadastrar Serviço ⭐ MVP

**User Story**: Como profissional, quero cadastrar um novo serviço com nome, preço e duração para que ele apareça disponível nos agendamentos.

**Por que P1**: Sem serviços cadastrados não é possível criar agendamentos.

**Acceptance Criteria**:

1. WHEN acesso `/services/novo` THEN sistema SHALL exibir formulário com campos: Nome (obrigatório), Preço (obrigatório), Duração em minutos (obrigatório), Descrição (opcional)
2. WHEN submeto o formulário com dados válidos THEN sistema SHALL salvar o serviço, exibir mensagem de sucesso e redirecionar para a listagem
3. WHEN submeto sem preencher Nome THEN sistema SHALL exibir erro inline "Nome é obrigatório" sem submeter
4. WHEN submeto sem preencher Preço THEN sistema SHALL exibir erro inline "Preço é obrigatório" sem submeter
5. WHEN submeto sem preencher Duração THEN sistema SHALL exibir erro inline "Duração é obrigatória" sem submeter
6. WHEN digito preço com valor zero ou negativo THEN sistema SHALL exibir erro inline "Preço deve ser maior que zero"
7. WHEN digito duração com valor zero ou negativo THEN sistema SHALL exibir erro inline "Duração deve ser maior que zero"

**Independent Test**: Abrir `/services/novo`, preencher nome + preço + duração e salvar — serviço aparece na listagem.

---

### P1: Listar e Buscar Serviços ⭐ MVP

**User Story**: Como profissional, quero ver todos os meus serviços e buscar rapidamente por nome para encontrar o que preciso.

**Por que P1**: A listagem é o ponto de entrada do módulo e base para o picker de agendamentos.

**Acceptance Criteria**:

1. WHEN acesso `/services` THEN sistema SHALL exibir lista de serviços ativos ordenados por nome, com nome, preço e duração visíveis
2. WHEN digito no campo de busca THEN sistema SHALL filtrar a lista em tempo real (debounce 300ms) mostrando apenas serviços cujo nome contenha o texto digitado
3. WHEN a busca não retorna resultados THEN sistema SHALL exibir mensagem "Nenhum serviço encontrado"
4. WHEN a lista está carregando THEN sistema SHALL exibir skeleton loader ou spinner
5. WHEN a duração é exibida THEN sistema SHALL formatar em horas e minutos (ex: "1h 30min", "2h", "30min")

**Independent Test**: Cadastrar 3 serviços com nomes distintos, buscar por parte do nome do segundo — só ele aparece.

---

### P1: Visualizar e Editar Serviço ⭐ MVP

**User Story**: Como profissional, quero abrir um serviço para ver seus dados completos e corrigir preço ou duração quando necessário.

**Por que P1**: Preços mudam — edição é necessária desde o MVP.

**Acceptance Criteria**:

1. WHEN clico em um serviço na listagem THEN sistema SHALL navegar para `/services/:id` exibindo todos os campos do cadastro
2. WHEN clico em "Editar" THEN sistema SHALL exibir o formulário pré-preenchido com os dados atuais
3. WHEN salvo alterações válidas THEN sistema SHALL atualizar o serviço, exibir confirmação e voltar para o perfil
4. WHEN cancelo a edição THEN sistema SHALL descartar alterações e voltar para o perfil sem salvar
5. WHEN o serviço não existe (ID inválido) THEN sistema SHALL exibir página de erro 404 com link para voltar à listagem

**Independent Test**: Editar o preço de um serviço — o novo preço aparece no perfil e na listagem.

---

### P2: Desativar / Reativar Serviço

**User Story**: Como profissional, quero desativar serviços que não ofereço mais para manter o catálogo limpo, sem perder o histórico de agendamentos antigos.

**Por que P2**: Importante para manter o catálogo organizado, mas não bloqueia o MVP nem os agendamentos.

**Acceptance Criteria**:

1. WHEN clico em "Desativar" no perfil de um serviço ativo THEN sistema SHALL pedir confirmação antes de agir
2. WHEN confirmo a desativação THEN sistema SHALL marcar o serviço como inativo e removê-lo da listagem padrão
3. WHEN acesso a listagem com filtro "Inativos" THEN sistema SHALL exibir apenas serviços inativos com opção de reativar
4. WHEN reativo um serviço THEN sistema SHALL marcá-lo como ativo e ele volta a aparecer na listagem padrão
5. WHEN um serviço inativo tem agendamentos futuros THEN sistema SHALL alertar a profissional antes de confirmar a desativação

**Independent Test**: Desativar um serviço — ele some da listagem padrão, aparece com filtro "Inativos", e pode ser reativado.

---

## Edge Cases

- WHEN o campo de busca é limpo THEN sistema SHALL restaurar a listagem completa
- WHEN o servidor retorna erro ao salvar THEN sistema SHALL exibir mensagem de erro genérica sem perder os dados do formulário
- WHEN navego para `/services` sem estar autenticada THEN sistema SHALL redirecionar para o login
- WHEN a duração é exibida no picker de agendamentos THEN sistema SHALL mostrar formato legível (ex: "2h 30min")

---

## Requirement Traceability

| Requirement ID | Story | Status |
|---|---|---|
| SVC-01 | P1: Cadastrar — formulário com campos obrigatórios | Verified |
| SVC-02 | P1: Cadastrar — salvar e redirecionar | Verified |
| SVC-03 | P1: Cadastrar — validação inline de campos | Verified |
| SVC-04 | P1: Cadastrar — preço e duração maiores que zero | Verified |
| SVC-05 | P1: Listar — exibição ordenada por nome com duração formatada | Verified |
| SVC-06 | P1: Listar — busca em tempo real por nome | Verified |
| SVC-07 | P1: Listar — estado vazio | Verified |
| SVC-08 | P1: Visualizar — perfil completo do serviço | Verified |
| SVC-09 | P1: Editar — formulário pré-preenchido e salvar | Verified |
| SVC-10 | P1: Editar — cancelar sem salvar | Verified |
| SVC-11 | P2: Desativar — confirmação e remoção da lista | Verified |
| SVC-12 | P2: Reativar — volta à listagem padrão | Verified |
| SVC-13 | P2: Desativar — alerta se há agendamentos futuros | Deferred → Agendamentos |

**Cobertura:** 13 requisitos, 12 verificados, 1 deferred ✅

---

## Success Criteria

- [ ] Profissional consegue cadastrar um serviço do zero sem consultar manual
- [ ] Duração sempre exibida em formato legível (horas e minutos)
- [ ] Serviço desativado não aparece no picker de agendamentos
- [ ] Zero perda de dados ao cancelar edição
