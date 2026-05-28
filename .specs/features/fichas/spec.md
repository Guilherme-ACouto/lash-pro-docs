# Fichas — Especificação

## Problema

A profissional precisa registrar e consultar fichas de cada cliente em dois formatos distintos: a **Anamnese** (dados de saúde + termo de ciência) e o **Mapping de Cílios** (ficha técnica do atendimento com desenho do olho, fotos e histórico). Hoje esses dados ficam em outro app sem integração com o sistema — a meta é centralizar tudo no Lash Manager.

## Objetivos

- [ ] Registrar e consultar a ficha de anamnese de cada cliente (uma por cliente, sempre atualizável)
- [ ] Registrar fichas de mapping com histórico por cliente (uma nova ficha a cada atendimento)
- [ ] Acessar fichas a partir do cadastro do cliente E pelo menu "Fichas" centralizado
- [ ] Funcionar perfeitamente em celular (preenchimento em campo durante o atendimento)

## Fora de Escopo

| Feature | Motivo |
|---|---|
| Assinatura digital do cliente | Definido como próxima fase pelo usuário |
| Envio da ficha por link para o cliente preencher | Complexidade de acesso externo sem autenticação — radar futuro |
| Integração da ficha de mapping com agendamento | Deferred — fase 3 (baixa automática de estoque também está nessa fase) |
| Ficha ConsultPro | Não especificado ainda — fora do escopo desta entrega |

---

## User Stories

### FIC-01 — Menu Fichas na navegação ⭐ P1

**User Story**: Como profissional, quero um item "Fichas" no menu principal para acessar todas as fichas de clientes de forma centralizada.

**Acceptance Criteria**:
1. WHEN acesso o sistema THEN SHALL existir item "Fichas" na sidebar (desktop) e na bottom nav (mobile) com ícone `assignment`
2. WHEN clico em Fichas THEN SHALL ver duas sub-seções: "Anamnese" e "Mapping"
3. WHEN acesso Fichas > Anamnese THEN SHALL ver lista de clientes com status da ficha (preenchida / não preenchida)
4. WHEN acesso Fichas > Mapping THEN SHALL ver lista de clientes com quantidade de fichas registradas

**Independent Test**: Navegar até /fichas/anamnese e /fichas/mapping sem erro.

---

### FIC-02 — Ficha de Anamnese: preenchimento ⭐ P1

**User Story**: Como profissional, quero preencher a ficha de anamnese de uma cliente com dados pessoais e perguntas de saúde para ter respaldo legal e clínico.

**Seções da ficha:**

**Dados do cliente** (campos de texto):
- Nome completo
- Responsável pela menor (opcional)
- Endereço, Bairro, Cidade, UF
- Data de nascimento, Celular, CPF, RG

**Saúde do cliente** (perguntas com toggle sim/não, exceto onde indicado):
- Já fez extensão de cílios antes?
- Tem rimel ou maquiagem nos olhos?
- Você tem alguma alergia?
- Você tem problema de tireoide?
- Você dorme de qual lado? *(radio: Direito / Esquerdo / Dois lados)*
- Fez algum procedimento na região dos olhos recentemente?
- Você está gestante ou amamentando?
- Já fez ou faz tratamento oncológico?
- Possui alguma doença de pele?
- Está em algum tratamento de saúde?
- Faz uso de algum medicamento?

**Termo de autorização** (texto exibido + checkbox de aceite):
- Texto padrão de ciência do procedimento
- Checkbox: "Certifico que todos os itens foram esclarecidos. Data/Hora automática"
- Status visual: "Termo assinado com sucesso!" quando aceito

**Acceptance Criteria**:
1. WHEN abro a ficha de uma cliente sem anamnese THEN SHALL ver formulário em branco com dados do cliente já pré-preenchidos (nome, celular) vindos do cadastro
2. WHEN preencho e salvo THEN sistema SHALL persistir a ficha vinculada ao cliente
3. WHEN acesso ficha de cliente que já tem anamnese THEN SHALL ver dados preenchidos anteriormente (modo edição)
4. WHEN marco o checkbox do termo THEN SHALL registrar data e hora do aceite e exibir confirmação
5. WHEN salvo ficha THEN SHALL atualizar status para "Preenchida" na listagem

**Independent Test**: Abrir ficha de cliente, preencher, salvar, reabrir e ver dados persistidos.

---

### FIC-03 — Acesso à anamnese pelo cadastro do cliente ⭐ P1

**User Story**: Como profissional, quero acessar a ficha de anamnese diretamente do cadastro de um cliente.

**Acceptance Criteria**:
1. WHEN visualizo o detalhe de um cliente THEN SHALL ver botão/aba "Anamnese"
2. WHEN clico em Anamnese THEN SHALL abrir a ficha de anamnese desse cliente (nova ou existente)

**Independent Test**: Abrir cliente → clicar em Anamnese → ficha abre corretamente.

---

### FIC-04 — Listagem de anamneses no menu Fichas ⭐ P1

**User Story**: Como profissional, quero ver a lista de todos os clientes com o status da anamnese.

**Acceptance Criteria**:
1. WHEN acesso Fichas > Anamnese THEN SHALL listar todos os clientes com: foto, nome, data de preenchimento (ou "Não preenchida")
2. WHEN clico em um cliente THEN SHALL abrir a ficha de anamnese
3. WHEN busco por nome THEN SHALL filtrar a lista

**Independent Test**: Ver lista com pelo menos 2 clientes com status diferente.

---

### FIC-05 — Ficha de Mapping: preenchimento ⭐ P1

**User Story**: Como profissional, quero preencher uma ficha de mapping de cílios para documentar o atendimento com informações técnicas.

**Campos da ficha** (grid 2 colunas):
- Tipo de Mapping *(select)*
- Curvatura *(select: B, C, D, L, M, J, CC)*
- Umidade *(número)*
- Temperatura *(número)*
- Espessura *(select: 0.03, 0.05, 0.07, 0.10, 0.12, 0.15, 0.18, 0.20, 0.25)*
- Marca do fio *(texto livre)*
- Formato do fio *(select: Natural, Gatinho, Boneca, Modelo, Fox Eye)*
- Adesivo / Cola *(texto livre)*
- Comprimentos usados *(texto livre, ex: "8, 10, 12mm")*
- Observações *(textarea)*

**Canvas de desenho do olho:**
- Fundo: SVG de um olho (contorno ovóide)
- Profissional desenha sobre o SVG com traço livre (freehand)
- Controles: Desfazer | Salvar desenho | Refazer
- Cor e espessura do traço configurável

**Fotos antes/depois:**
- Upload de foto "Antes"
- Upload de foto "Depois"

**Acceptance Criteria**:
1. WHEN abro nova ficha de mapping THEN SHALL ver formulário com data/hora atual e nome da cliente
2. WHEN preencho campos e salvo THEN sistema SHALL criar novo registro de mapping vinculado ao cliente com timestamp
3. WHEN acesso canvas THEN SHALL conseguir desenhar livremente sobre o olho com toque (mobile) e mouse (desktop)
4. WHEN toco Desfazer THEN SHALL remover último traço desenhado
5. WHEN salvo a ficha THEN SHALL persistir imagem do canvas como dados (base64 ou SVG paths)
6. WHEN faço upload de foto THEN SHALL armazenar a imagem vinculada à ficha

**Independent Test**: Preencher ficha, desenhar no canvas, salvar, reabrir e ver tudo persistido.

---

### FIC-06 — Histórico de mappings por cliente ⭐ P1

**User Story**: Como profissional, quero ver o histórico de fichas de mapping de uma cliente para acompanhar a evolução dos atendimentos.

**Acceptance Criteria**:
1. WHEN acesso mappings de uma cliente THEN SHALL ver lista cronológica de fichas (mais recente primeiro) com data
2. WHEN clico em uma ficha THEN SHALL ver todos os dados daquela sessão (read-only por padrão, com opção de editar)
3. WHEN crio nova ficha THEN SHALL criar um registro novo sem sobrescrever os anteriores
4. WHEN há mais de uma ficha THEN SHALL exibir contador "X fichas" na listagem de clientes

**Independent Test**: Criar 2 fichas para o mesmo cliente e ver ambas no histórico.

---

### FIC-07 — Acesso ao mapping pelo cadastro do cliente ⭐ P1

**User Story**: Como profissional, quero acessar e criar fichas de mapping diretamente do cadastro de um cliente.

**Acceptance Criteria**:
1. WHEN visualizo o detalhe de um cliente THEN SHALL ver botão/aba "Mapping"
2. WHEN clico em Mapping THEN SHALL ver lista histórica das fichas + botão "Nova ficha"
3. WHEN clico em "Nova ficha" THEN SHALL abrir formulário de mapping em branco para esse cliente

**Independent Test**: Abrir cliente → Mapping → Nova ficha → preencher → salvar → ver no histórico.

---

### FIC-08 — Listagem de mappings no menu Fichas ⭐ P1

**User Story**: Como profissional, quero ver todos os clientes e acessar suas fichas de mapping pelo menu centralizado.

**Acceptance Criteria**:
1. WHEN acesso Fichas > Mapping THEN SHALL listar clientes com: foto, nome, número de fichas, data da última ficha
2. WHEN clico em cliente THEN SHALL ver histórico de fichas desse cliente
3. WHEN busco por nome THEN SHALL filtrar a lista

**Independent Test**: Ver lista com contador de fichas por cliente.

---

### FIC-09 — Tipos de Mapping configuráveis ⭐ P2

**User Story**: Como profissional, quero que os valores do select "Tipo de Mapping" reflitam os estilos que ofereço.

**Acceptance Criteria**:
1. WHEN abro select "Tipo de Mapping" THEN SHALL ver opções padrão: Natural, Clássico Volume, Mega Volume, Híbrido, Russa, Wet Look
2. WHEN seleciono um tipo THEN SHALL salvar com a ficha

---

### FIC-10 — Impressão / exportação da anamnese ⭐ P3

**User Story**: Como profissional, quero imprimir ou exportar a anamnese em PDF para guardar fisicamente.

**Acceptance Criteria**:
1. WHEN estou na ficha de anamnese THEN SHALL ver botão "Imprimir / PDF"
2. WHEN clico THEN SHALL abrir dialog de impressão do browser com layout limpo

---

## Edge Cases

- WHEN cliente não tem cadastro no sistema THEN SHALL não ser possível criar ficha (ficha sempre vinculada a cliente existente)
- WHEN acesso Fichas > Anamnese e nenhum cliente tem ficha THEN SHALL exibir estado vazio com CTA "Selecione um cliente para começar"
- WHEN canvas está vazio e salvo ficha THEN SHALL salvar ficha sem imagem (campo nulo)
- WHEN ficha de mapping já tem foto e faço novo upload THEN SHALL substituir a foto
- WHEN cliente é inativado THEN SHALL fichas existentes continuarem acessíveis (read-only)

---

## Rastreabilidade de Requisitos

| ID | Story | Status |
|---|---|---|
| FIC-01 | Menu Fichas navegação | Pendente |
| FIC-02 | Anamnese preenchimento | Pendente |
| FIC-03 | Anamnese pelo cadastro | Pendente |
| FIC-04 | Listagem anamneses | Pendente |
| FIC-05 | Mapping preenchimento + canvas | Pendente |
| FIC-06 | Histórico de mappings | Pendente |
| FIC-07 | Mapping pelo cadastro | Pendente |
| FIC-08 | Listagem mappings | Pendente |
| FIC-09 | Tipos de mapping | Pendente |
| FIC-10 | Impressão anamnese | Pendente |

---

## Sucesso

- [ ] Profissional consegue preencher anamnese completa de uma cliente em menos de 3 minutos no celular
- [ ] Ficha de mapping com desenho salvo e fotos antes/depois acessível no histórico
- [ ] Zero perda de dados ao fechar e reabrir a ficha
