# Estoque — Especificação

## Problem Statement

A profissional controla seus insumos (colas, fios, primers, removedores) manualmente em caderno ou de
cabeça. Não sabe exatamente quanto tem de cada produto, descobre que acabou o material no meio do
atendimento, e não registra o custo das compras de forma que impacte o financeiro. O módulo de Estoque
centraliza o controle de insumos, integra compras ao módulo Financeiro (gerando despesas automaticamente)
e alerta quando o estoque fica abaixo do mínimo.

## Goals

- [ ] Profissional consegue cadastrar insumos com preço de custo e fornecedor
- [ ] Toda compra de insumo gera automaticamente uma despesa no módulo Financeiro
- [ ] Compras à vista marcam a despesa como paga; compras faturadas ficam como pendentes
- [ ] Profissional é alertada quando o estoque fica abaixo do mínimo definido
- [ ] Profissional consegue registrar saídas manuais por uso, perda ou ajuste

## Out of Scope

| Feature | Razão |
|---|---|
| Baixa automática ao completar agendamento | **Deferred — Fase 3** (requer vínculo insumo ↔ serviço; fica no radar) |
| Gestão de fornecedores como entidade separada | Fase 3 — fornecedor é texto livre por ora |
| Pedidos de compra / reposição automática | Fase 3 |
| Código de barras / leitor | Fase 3 |
| Parcelamento de compra (N parcelas) | Fase 3 — por ora só à vista ou faturado (1 vencimento) |
| Relatório de consumo mensal | Fase 3 |
| Multi-localização (estoques por unidade) | Fase 4 — multi-tenant |

---

## User Stories

### P1: Cadastrar Item de Estoque ⭐ MVP

**User Story**: Como profissional, quero cadastrar um insumo com nome, unidade, preço de custo, fornecedor e quantidade mínima para controlar o que tenho e quanto me custa.

**Por que P1**: Sem cadastro não há nenhuma outra operação possível.

**Acceptance Criteria**:

1. WHEN acesso `/inventory/novo` THEN sistema SHALL exibir formulário com campos:
   - Nome (obrigatório)
   - Código interno (opcional — gerado automaticamente se vazio)
   - Unidade (obrigatório: `un` / `ml` / `g` / `cm` / `par`)
   - Preço de custo unitário (obrigatório, > 0, BigDecimal)
   - Fornecedor (opcional, texto livre)
   - Quantidade atual (obrigatório, ≥ 0)
   - Quantidade mínima (obrigatório, ≥ 0)
   - Observações (opcional)
2. WHEN submeto o formulário com dados válidos THEN sistema SHALL salvar o item, exibir mensagem de sucesso e redirecionar para a listagem
3. WHEN submeto sem preencher Nome THEN sistema SHALL exibir erro inline "Nome é obrigatório"
4. WHEN submeto com Preço de custo ≤ 0 THEN sistema SHALL exibir erro inline "Preço de custo deve ser maior que zero"
5. WHEN submeto com código já existente THEN sistema SHALL exibir erro "Já existe um item com este código"
6. WHEN código interno é omitido THEN sistema SHALL gerar automaticamente no formato `INS-XXXXXX`

**Independent Test**: Cadastrar item com nome + unidade + preço de custo + quantidade — item aparece na listagem com os dados corretos.

---

### P1: Listar e Buscar Itens ⭐ MVP

**User Story**: Como profissional, quero ver todos os meus insumos em uma lista e buscar rapidamente por nome ou código para saber o estoque disponível.

**Por que P1**: A listagem é o ponto de entrada do módulo.

**Acceptance Criteria**:

1. WHEN acesso `/inventory` THEN sistema SHALL exibir lista de itens ativos ordenados por nome, com colunas: Código, Nome, Unidade, Qtd. atual, Mínimo, Custo unit., Cadastrado em, Ações
2. WHEN a quantidade de um item é 0 THEN sistema SHALL exibir badge "Sem estoque" vermelho na linha
3. WHEN a quantidade de um item é > 0 e ≤ mínimo THEN sistema SHALL exibir badge "Estoque baixo" amarelo/laranja
4. WHEN digito no campo de busca THEN sistema SHALL filtrar em tempo real (debounce 300ms) por nome ou código
5. WHEN a busca não retorna resultados THEN sistema SHALL exibir mensagem "Nenhum item encontrado"
6. WHEN há mais de 20 itens THEN sistema SHALL paginar com controles de navegação
7. WHEN a lista está carregando THEN sistema SHALL exibir skeleton loader

**Independent Test**: Cadastrar 3 itens com nomes distintos, buscar por parte do nome do segundo — só ele aparece.

---

### P1: Editar Item ⭐ MVP

**User Story**: Como profissional, quero editar os dados de um insumo para corrigir preço de custo, fornecedor ou quantidade mínima.

**Por que P1**: Dados erram na digitação — edição necessária desde o MVP.

**Acceptance Criteria**:

1. WHEN clico no ícone de editar THEN sistema SHALL abrir formulário pré-preenchido com todos os dados atuais
2. WHEN salvo alterações válidas THEN sistema SHALL atualizar o item, exibir confirmação e fechar o formulário
3. WHEN altero o preço de custo THEN sistema SHALL salvar o novo valor (não retroage em movimentações anteriores)
4. WHEN cancelo THEN sistema SHALL descartar alterações sem salvar

**Independent Test**: Editar o preço de custo e fornecedor de um item — os novos valores aparecem na listagem.

---

### P1: Excluir Item ⭐ MVP

**User Story**: Como profissional, quero excluir um insumo cadastrado por engano.

**Por que P1**: Correção de cadastros errados é necessária desde o MVP.

**Acceptance Criteria**:

1. WHEN clico em "Excluir" THEN sistema SHALL exibir dialog de confirmação antes de prosseguir
2. WHEN o item tem movimentações registradas THEN sistema SHALL bloquear a exclusão com mensagem "Este item possui movimentações e não pode ser excluído — desative-o em vez disso"
3. WHEN confirmo a exclusão de item sem movimentações THEN sistema SHALL remover permanentemente e atualizar a listagem
4. WHEN cancelo THEN sistema SHALL fechar sem excluir

**Independent Test**: Excluir item sem movimentações — some da listagem.

---

### P1: Registrar Compra (Entrada com integração Financeiro) ⭐ MVP

**User Story**: Como profissional, quero registrar uma compra de insumo informando quantidade, fornecedor, data e forma de pagamento, para que o estoque seja atualizado e a despesa entre automaticamente no financeiro.

**Por que P1**: É o fluxo principal do módulo — compra = entrada de estoque + despesa financeira.

**Acceptance Criteria**:

1. WHEN clico em "Registrar compra" de um item THEN sistema SHALL abrir dialog com campos:
   - Quantidade comprada (obrigatório, > 0)
   - Preço de custo unitário (pré-preenchido com o cadastrado, editável para refletir o preço atual da nota)
   - Fornecedor (pré-preenchido se cadastrado no item, editável)
   - Data da compra (obrigatório, padrão: hoje)
   - Forma de pagamento: **À vista** ou **Faturado**
   - Data de vencimento (obrigatório apenas se Faturado)
   - Observação (opcional)
2. WHEN confirmo a compra THEN sistema SHALL:
   - Somar a quantidade ao estoque atual do item
   - Criar automaticamente uma `FinancialEntry` do tipo EXPENSE com:
     - Descrição: `"Compra: {nome do item}"`
     - Categoria: `SUPPLY` (novo tipo de despesa)
     - Valor: `quantidade × preço de custo informado`
     - Data: data da compra informada
     - `isPaid = true` e `paymentDate = data da compra` se À vista
     - `isPaid = false` e `dueDate = data de vencimento` se Faturado
3. WHEN a compra é confirmada THEN sistema SHALL exibir confirmação com resumo: "X unidades adicionadas • Despesa de R$ Y criada no Financeiro"
4. WHEN o preço informado na compra difere do cadastrado THEN sistema SHALL atualizar o `costPrice` do item com o novo valor
5. WHEN a data de vencimento não é informada para compra faturada THEN sistema SHALL exibir erro "Informe a data de vencimento"

**Independent Test**: Registrar compra de 10 unidades a R$ 5,00 à vista — estoque aumenta 10, despesa de R$ 50,00 aparece no Financeiro como paga.

---

### P2: Registrar Saída Manual

**User Story**: Como profissional, quero registrar saídas de estoque por uso, perda ou ajuste para manter a quantidade sempre correta.

**Por que P2**: O controle de saídas é importante, mas o MVP já começa com entradas via compra.

**Acceptance Criteria**:

1. WHEN clico em "Registrar saída" de um item THEN sistema SHALL abrir dialog com campos: Quantidade (obrigatório, > 0), Motivo (obrigatório: Uso / Perda / Ajuste / Outro), Observação (opcional), Data (padrão: hoje)
2. WHEN confirmo a saída THEN sistema SHALL subtrair a quantidade do estoque atual
3. WHEN a saída resultar em estoque negativo THEN sistema SHALL exibir aviso "Estoque ficará negativo" mas permitir confirmar
4. WHEN a saída é salva THEN sistema SHALL atualizar a quantidade na listagem imediatamente
5. Saídas manuais **não geram** entrada financeira (são consumo interno, já coberto no preço do serviço)

**Independent Test**: Registrar saída de 2 unidades de um item com 5 — quantidade passa para 3, sem entrada no Financeiro.

---

### P2: Histórico de Movimentações

**User Story**: Como profissional, quero ver o histórico completo de entradas e saídas de um insumo para rastrear o consumo e as compras realizadas.

**Por que P2**: Importante para gestão, mas o MVP funciona sem histórico detalhado.

**Acceptance Criteria**:

1. WHEN acesso o detalhe de um item THEN sistema SHALL exibir aba "Histórico" com todas as movimentações em ordem cronológica reversa
2. WHEN exibo uma movimentação de compra THEN sistema SHALL mostrar: data, tipo "Compra", quantidade, preço unit., total, fornecedor, forma pagamento e link para a entrada no Financeiro
3. WHEN exibo uma movimentação de saída manual THEN sistema SHALL mostrar: data, tipo, quantidade, motivo e saldo resultante
4. WHEN o histórico está vazio THEN sistema SHALL exibir "Nenhuma movimentação registrada"

**Independent Test**: Após registrar uma compra e uma saída, o histórico exibe os dois registros na ordem correta.

---

### P2: Desativar / Reativar Item

**User Story**: Como profissional, quero desativar insumos que não uso mais para manter a listagem limpa sem perder o histórico.

**Por que P2**: Organização importante, mas não bloqueia o MVP.

**Acceptance Criteria**:

1. WHEN clico em "Desativar" de um item ativo THEN sistema SHALL desativar imediatamente (sem confirm)
2. WHEN item é desativado THEN sistema SHALL aparecer apenas com filtro "Inativos"
3. WHEN clico em "Reativar" THEN sistema SHALL reativar e o item volta à listagem padrão
4. WHEN uso o filtro de status (Todos / Ativos / Inativos) THEN sistema SHALL exibir apenas itens no status selecionado

**Independent Test**: Desativar um item — some da listagem padrão, aparece com filtro "Inativos", pode ser reativado.

---

### P2: Alerta de Estoque Baixo

**User Story**: Como profissional, quero ser alertada sobre insumos com estoque abaixo do mínimo para planejar compras antes de acabar.

**Por que P2**: Evita surpresas operacionais, mas o sistema funciona sem isso.

**Acceptance Criteria**:

1. WHEN acesso `/inventory` e há itens com estoque ≤ mínimo THEN sistema SHALL exibir banner no topo: "X itens com estoque baixo"
2. WHEN clico no banner THEN sistema SHALL filtrar mostrando apenas itens com estoque baixo
3. WHEN o Dashboard é carregado THEN sistema SHALL exibir card de alerta de estoque com link para `/inventory?filter=low`

**Independent Test**: Cadastrar item com qtd 2 e mínimo 5 — badge na linha e banner no topo aparecem.

---

## Edge Cases

- WHEN campo de busca é limpo THEN sistema SHALL restaurar a listagem completa
- WHEN qtd mínima é maior que qtd atual no cadastro THEN sistema SHALL salvar normalmente (badge aparece automaticamente)
- WHEN servidor retorna erro ao salvar THEN sistema SHALL exibir mensagem de erro sem perder dados do formulário
- WHEN preço da compra é diferente do custo cadastrado THEN sistema SHALL atualizar o `costPrice` do item automaticamente
- WHEN navego para `/inventory` sem autenticação THEN sistema SHALL redirecionar para login
- WHEN código gerado automaticamente colide THEN sistema SHALL incrementar sufixo até ser único

---

## Requirement Traceability

| Requirement ID | Story | Status |
|---|---|---|
| INV-01 | P1: Cadastrar — formulário com todos os campos | Pending |
| INV-02 | P1: Cadastrar — salvar e redirecionar | Pending |
| INV-03 | P1: Cadastrar — validações inline (nome, preço, código único) | Pending |
| INV-04 | P1: Cadastrar — geração automática de código | Pending |
| INV-05 | P1: Listar — colunas e ordenação | Pending |
| INV-06 | P1: Listar — badge sem estoque (vermelho) e estoque baixo (laranja) | Pending |
| INV-07 | P1: Listar — busca em tempo real por nome/código | Pending |
| INV-08 | P1: Listar — paginação e estado vazio | Pending |
| INV-09 | P1: Editar — formulário pré-preenchido e salvar | Pending |
| INV-10 | P1: Editar — cancelar sem salvar | Pending |
| INV-11 | P1: Excluir — dialog de confirmação | Pending |
| INV-12 | P1: Excluir — bloquear se tem movimentações | Pending |
| INV-13 | P1: Compra — dialog com qtd, preço, fornecedor, data, forma pgto | Pending |
| INV-14 | P1: Compra — soma ao estoque | Pending |
| INV-15 | P1: Compra — cria FinancialEntry EXPENSE automaticamente | Pending |
| INV-16 | P1: Compra — à vista: isPaid=true; faturado: isPaid=false + dueDate | Pending |
| INV-17 | P1: Compra — atualiza costPrice se valor difere | Pending |
| INV-18 | P2: Saída manual — dialog com qtd, motivo, data | Pending |
| INV-19 | P2: Saída manual — subtrai do estoque, sem entrada financeira | Pending |
| INV-20 | P2: Saída manual — aviso de estoque negativo | Pending |
| INV-21 | P2: Histórico — movimentações em ordem reversa | Pending |
| INV-22 | P2: Histórico — compra exibe link para entrada no Financeiro | Pending |
| INV-23 | P2: Desativar/Reativar item | Pending |
| INV-24 | P2: Filtro de status na listagem | Pending |
| INV-25 | P2: Banner de alerta de estoque baixo | Pending |
| INV-26 | P2: Card de alerta no Dashboard | Pending |

**Cobertura:** 26 requisitos, 0 verificados, 26 pendentes

---

## Deferred — No Radar

| ID | Descrição | Quando |
|---|---|---|
| INV-D01 | Baixa automática de estoque ao completar agendamento | Fase 3 — requer vínculo insumo ↔ serviço |
| INV-D02 | Fornecedor como entidade separada com CNPJ, contato, histórico | Fase 3 |
| INV-D03 | Parcelamento de compra em N parcelas (N FinancialEntries) | Fase 3 |
| INV-D04 | Relatório de consumo por período | Fase 3 |

---

## Success Criteria

- [ ] Profissional cadastra um insumo do zero em menos de 1 minuto
- [ ] Registrar uma compra cria a despesa no Financeiro automaticamente, sem etapa extra
- [ ] Badge de estoque baixo/zerado aparece corretamente em tempo real
- [ ] Compra à vista: despesa aparece como paga no Financeiro
- [ ] Compra faturada: despesa aparece como pendente com data de vencimento correta
- [ ] Item com movimentações não pode ser excluído — somente desativado
