# lash-docs

Documentação do sistema Lash Manager.

## Estrutura

```
.specs/
├── codebase/
│   ├── ARCHITECTURE.md     → arquitetura hexagonal, decisões técnicas
│   ├── CONVENTIONS.md      → padrões de nomenclatura e código
│   └── STACK.md            → tecnologias, versões, portas, variáveis
└── project/
    ├── PROJECT.md          → visão do produto, módulos, regras de negócio
    ├── ROADMAP.md          → fases de entrega
    └── STATE.md            → estado atual, credenciais, como subir o ambiente

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
