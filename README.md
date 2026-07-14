# lash-docs

Documentação do sistema Lash Manager.

O backend (`lash-backend/`) é um projeto **multi-módulo Maven**: 9 módulos independentes
(`lash-core`, `lash-clients`, `lash-services`, `lash-appointments`, `lash-finance`,
`lash-stock`, `lash-fichas`, `lash-dashboard`, `lash-app`), cada um com sua própria
arquitetura hexagonal interna, testes e migration Flyway. Detalhes em
`.specs/codebase/ARCHITECTURE.md` e `.specs/codebase/STRUCTURE.md`.

## Estrutura

```
.specs/
├── codebase/
│   ├── STACK.md            → tecnologias, versões, portas, variáveis, build multi-módulo
│   ├── ARCHITECTURE.md     → arquitetura hexagonal por módulo, dependências entre módulos
│   ├── STRUCTURE.md        → árvore de diretórios dos 9 módulos + migrations
│   ├── CONVENTIONS.md      → padrões de nomenclatura e código
│   ├── TESTING.md          → testes por módulo, @DisplayName, gate checks
│   ├── INTEGRATIONS.md     → JWT, banco, Flyway por módulo, SMTP, CORS
│   └── CONCERNS.md         → tech debt e riscos conhecidos, por prioridade
├── project/
│   ├── PROJECT.md          → visão do produto, módulos, regras de negócio
│   ├── ROADMAP.md          → fases de entrega
│   └── STATE.md            → estado atual, credenciais, como subir o ambiente
└── features/               → spec/design/tasks por feature (ex.: clientes/, servicos/)

patterns/
├── patterns_backend.md     → exemplos de use case, entity, mapper, controller
└── patterns_frontend.md    → exemplos de service, NgRx, component, rotas
```

## Como usar com IA (SDD)

Ao iniciar uma nova conversa no Cursor ou VS Code com Claude, cole o contexto relevante:

```
Contexto do projeto:
- [cole o conteúdo de ARCHITECTURE.md]
- [cole o conteúdo de CONVENTIONS.md]
- [cole o padrão relevante de patterns_backend.md ou patterns_frontend.md]

Agora implemente: [descreva o que quer]
```

Quanto mais contexto você fornecer, mais aderente ao projeto será o código gerado.
