# Lash Manager — Visão do Projeto

## Contexto de Negócio

Sistema de gestão completo para profissionais de lash design (extensão de cílios).
Desenvolvido inicialmente para uso de uma profissional autônoma em salão fixo,
com arquitetura preparada para escalar para múltiplos salões e profissionais.

## Usuária Principal (Fase 1)

- Profissional autônoma de lash design
- Atende em salão fixo
- Trabalha sozinha atualmente, com plano de contratar funcionárias
- Gerencia agenda, clientes, financeiro e estoque

## Objetivos do Produto

### Curto prazo
- Substituir controles manuais (papel, WhatsApp, planilhas)
- Centralizar agenda, clientes, financeiro e estoque em um único sistema
- Funcionar bem no celular e no computador

### Médio prazo
- Suporte a múltiplas funcionárias com controle de comissões
- Notificações automáticas de agendamento via WhatsApp
- Portal público para o cliente agendar pelo próprio link

### Longo prazo
- Plataforma multi-tenant: vários salões usando o mesmo sistema
- App mobile nativo (iOS/Android)
- Marketplace de agendamentos

## Módulos do Sistema

| Módulo | Descrição | Fase | Status |
|---|---|---|---|
| Autenticação | Login, recuperação de senha, controle de acesso por perfil | 1 | ✅ Concluído |
| Clientes | Cadastro, histórico, busca | 1 | ✅ Concluído |
| Serviços | Catálogo de serviços com preço e duração | 1 | ✅ Concluído |
| Agendamentos | Calendário, agenda do dia, status de atendimento | 1 | 🔄 Implementado (aguardando teste) |
| Financeiro | Contas a pagar/receber, fluxo de caixa, DRE, comissões | 2 | ⏳ Pendente |
| Estoque | Insumos, movimentação, alertas de mínimo | 2 | ⏳ Pendente |
| Notificações | Lembretes via WhatsApp para clientes | 3 | ⏳ Pendente |
| Portal do Cliente | Link público para auto-agendamento | 3 | ⏳ Pendente |
| Multi-tenant | Suporte a múltiplos salões | 4 | ⏳ Pendente |
| App Mobile | iOS/Android nativo | 4 | ⏳ Pendente |

## Perfis de Usuário

| Perfil | Descrição |
|---|---|
| OWNER | Dona do salão — acesso total |
| EMPLOYEE | Funcionária — acesso à agenda e clientes, sem financeiro |
| CLIENT | Cliente — acesso apenas ao portal de agendamento (Fase 3) |

## Regras de Negócio Principais

### Agendamentos
- Horário de atendimento é flexível, definido pela profissional no sistema
- Clientes podem agendar via portal público (Fase 3) ou a profissional agenda pelo sistema
- Status: `AGENDADO` → `CONFIRMADO` → `REALIZADO` | `CANCELADO`
- Ao marcar como `REALIZADO`, gera automaticamente uma conta a receber
- Bloqueio automático de horários já ocupados

### Financeiro
- Contas a receber geradas automaticamente ao concluir atendimento
- Contas a pagar registradas manualmente
- Fluxo de caixa: entradas x saídas por período
- DRE mensal com receita, despesas e lucro
- Comissões por funcionária calculadas sobre valor do serviço realizado

### Estoque
- Baixa automática ao concluir serviço (configurável por serviço)
- Ajuste manual sempre disponível
- Alerta visual quando quantidade abaixo do mínimo configurado
