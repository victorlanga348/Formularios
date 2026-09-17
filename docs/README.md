# Sistema de Matrícula e Inscrição Académica — Índice Mestre da Documentação (`/docs`)

Bem-vindo ao centro oficial de especificações técnicas, contratos de dados, design system e governança do **Sistema de Matrícula e Inscrição Académica**.

> **Regra Primária de Desenvolvimento (Spec-Driven Development):**  
> O diretório `/docs` e o [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md) são a **única fonte oficial da verdade**. Nenhuma linha de código deve ser alterada ou criada sem que a especificação correspondente esteja atualizada, o plano de tarefa estruturado e aprovado pelo utilizador.

---

## 1. Mapa Completo da Documentação

```text
/
├── AGENTS.md                          # Regras invioláveis para agentes de IA e desenvolvedores
├── GEMINI.md                          # Ponte de governança para assistentes
├── README.md                          # Visão geral consolidada, esquemas e regras
└── docs/
    ├── README.md                      # Este índice mestre
    ├── PRD.md                         # Documento de Definição de Produto e Arquitetura (Fonte da Verdade)
    ├── documentation-governance.md    # Matriz de impacto, trilhas de equipa e checklist de sprint
    │
    ├── architecture/                  # Stack, infraestrutura e esquemas de dados
    │   ├── stack.md                   # NestJS, PostgreSQL, Prisma Multi-Client, Next.js, Opossum
    │   └── data-model.md              # Prisma Schema interno (PostgreSQL) e externo (Legado)
    │
    ├── product/                       # Regras acadêmicas e requisitos de negócio
    │   └── rules-engine.md            # Regras P0 a P5, Paridade, Barreira de Ciclo, PPL e Cascata
    │
    ├── integrations/                  # Contratos de API REST, autenticação e boletos
    │   ├── auth-financeiro.md         # JWT claims, Circuit Breaker fail-closed e Opossum
    │   └── api-matricula.md           # Endpoints REST, Idempotency-Key e Transações ACID
    │
    ├── design/                        # Design System, tokens visuais e responsividade
    │   ├── tokens.md                  # Paleta Dark Mode, contraste e tipografia
    │   └── responsive.md              # Breakpoints Mobile, Tablet e Desktop
    │
    ├── pages/                         # Especificações detalhadas por tela/rota
    │   ├── login.md                   # Tela de autenticação (/login)
    │   ├── divida-liquidar.md         # Confinamento financeiro (/divida/liquidar)
    │   ├── matricula.md               # Tela principal de inscrição (/matricula)
    │   └── confirmacao.md             # Confirmação e download de boleto (/matricula/confirmacao)
    │
    ├── components/                    # Especificação técnica de componentes de UI
    │   ├── tabela-grupo1.md           # Tabela de cadeiras regulares (bloqueios/precedências)
    │   ├── tabela-grupo2.md           # Tabela de atrasos interativa e efeito cascata
    │   └── barra-fixa-rodape.md       # Barra de ação fixa, debounce e chave de idempotência
    │
    ├── copywriting/                   # Catálogo de microcopy e mensagens institucionais
    │   └── microcopy.md               # Textos de erro, avisos de calouro e bloqueios
    │
    ├── audits/                        # Conformidade e auditorias de qualidade
    │   └── accessibility-and-security.md # Checklist WCAG 2.1 AA e premissas Zero-Trust
    │
    ├── tasks/                         # Planos de tarefas e histórico de sprints
    │   └── template.md                # Template obrigatório para abertura de tarefas
    │
    └── specs/                         # Especificações originais consolidadas (preservadas)
        ├── auth-e-financeiro.md
        ├── motor-de-regras.md
        ├── api-matricula.md
        └── ui-fluxo-estudante.md
```

---

## 2. Guias de Governança e Processo

- [docs/PRD.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/PRD.md) — Documento de Especificação Técnica e Definição de Produto (Fonte da Verdade Definitiva).
- [AGENTS.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/AGENTS.md) — Regras invioláveis para desenvolvimento agentic e desenvolvedores humanos.
- [docs/documentation-governance.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/documentation-governance.md) — Matriz de Impacto e Checklist de encerramento de tarefas.
- [docs/tasks/template.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/tasks/template.md) — Template padrão para criação de tarefas (`/docs/tasks/YYYY-MM-DD-nome-da-tarefa.md`).

---

## 3. Módulos Técnicos e Diretórios

### 3.1 Arquitetura e Dados (`/docs/architecture`)
- [docs/architecture/stack.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/architecture/stack.md) — Stack técnica completa e justificativas.
- [docs/architecture/data-model.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/architecture/data-model.md) — Prisma schemas interno (PostgreSQL) e legado externo.

### 3.2 Produto e Motor de Regras (`/docs/product`)
- [docs/product/rules-engine.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/product/rules-engine.md) — Validações encadeadas Zero-Trust (P0 a P5), barreira de ciclo, PPL e cascata.

### 3.3 Integrações e APIs (`/docs/integrations`)
- [docs/integrations/auth-financeiro.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/integrations/auth-financeiro.md) — JWT claims, Circuit Breaker Opossum e conciliação reativa.
- [docs/integrations/api-matricula.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/integrations/api-matricula.md) — Endpoints REST, concorrência, idempotência e boletos.

### 3.4 Design System & Responsividade (`/docs/design`)
- [docs/design/tokens.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/design/tokens.md) — Paleta Dark Mode, tipografia e estados de foco.
- [docs/design/responsive.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/design/responsive.md) — Comportamento nos breakpoints Mobile, Tablet e Desktop.

### 3.5 Telas e Rotas (`/docs/pages`)
- [docs/pages/login.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/pages/login.md) — Fluxo e validações da rota `/login`.
- [docs/pages/divida-liquidar.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/pages/divida-liquidar.md) — Confinamento financeiro na rota `/divida/liquidar`.
- [docs/pages/matricula.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/pages/matricula.md) — Tela principal de seleção acadêmica `/matricula`.
- [docs/pages/confirmacao.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/pages/confirmacao.md) — Comprovativo e download do boleto em `/matricula/confirmacao`.

### 3.6 Componentes de UI (`/docs/components`)
- [docs/components/tabela-grupo1.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/components/tabela-grupo1.md) — Tabela de cadeiras regulares com bloqueio por precedência.
- [docs/components/tabela-grupo2.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/components/tabela-grupo2.md) — Tabela de atrasos com seleção interativa e disparo de cascata.
- [docs/components/barra-fixa-rodape.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/components/barra-fixa-rodape.md) — Barra fixa com totalização em tempo real, debounce e envio com `Idempotency-Key`.

### 3.7 Copywriting e Mensagens (`/docs/copywriting`)
- [docs/copywriting/microcopy.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/copywriting/microcopy.md) — Catálogo oficial de mensagens de bloqueio, erro e sucesso.

### 3.8 Auditorias (`/docs/audits`)
- [docs/audits/accessibility-and-security.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/audits/accessibility-and-security.md) — Conformidade WCAG 2.1 AA e premissas de segurança Zero-Trust.

---

## 4. Trilhas de Desenvolvimento em Equipa

| Trilha | Foco de Atuação | Specs Primárias |
| :--- | :--- | :--- |
| **Trilha A** | Backend NestJS, PostgreSQL, Prisma Multi-Client, Auth, Circuit Breaker e Boletos | [architecture/stack.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/architecture/stack.md), [architecture/data-model.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/architecture/data-model.md), [integrations/auth-financeiro.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/integrations/auth-financeiro.md), [integrations/api-matricula.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/integrations/api-matricula.md) |
| **Trilha B** | Motor de Regras (`RulesEngineService`), pipeline P0 a P5, algoritmos de cascata e testes determinísticos | [product/rules-engine.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/product/rules-engine.md) |
| **Trilha C** | Frontend Next.js App Router, Tailwind Dark Mode, Zustand, Telas, Componentes e Acessibilidade | [design/tokens.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/design/tokens.md), [design/responsive.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/design/responsive.md), [pages/](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/pages/), [components/](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/components/), [audits/accessibility-and-security.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/audits/accessibility-and-security.md) |
