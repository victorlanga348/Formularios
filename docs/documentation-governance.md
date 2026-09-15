# Matriz de Governança de Documentação e Trabalho em Grupo

## 1. Princípio Fundamental (Spec-Driven Development)
O código é um reflexo direto e estrito das especificações documentadas. Toda e qualquer alteração de regra de negócio, contrato de API, schema de banco ou fluxo de interface deve ser documentada primeiro em `/docs` ou refletida no [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md).

> **Regra Primária:**  
> Nenhuma linha de código é considerada finalizada ou aceita sem que a documentação correspondente esteja perfeitamente sincronizada e sem desvios.

---

## 2. Estrutura Oficial de Documentação (`/docs`)

```text
/docs
├── README.md                          # Índice Mestre da Documentação
├── documentation-governance.md        # Este documento (Matriz de Impacto e Governança)
├── architecture/                      # Stack e modelo de dados
├── product/                           # Regras de negócio e motor de validação
├── integrations/                      # Contratos REST, auth e boletos
├── design/                            # Tokens de design e responsividade
├── pages/                             # Especificação de cada tela/rota
├── components/                        # Especificação de componentes de UI
├── copywriting/                       # Microcopy e mensagens do sistema
├── audits/                            # Acessibilidade e premissas de segurança
├── tasks/                             # Registros de tarefas e histórico de sprints
│   └── template.md                    # Template obrigatório de tarefa
└── specs/                             # Especificações consolidadas de referência
```

---

## 3. Protocolo de Trabalho em Grupo e Git Flow

1. **Trilhas de Desenvolvimento (Tracks):**
   - **Trilha A (Backend & DB):** Schemas Prisma Multi-Client, migrations, controllers, transações ACID, geração de PDFs e Circuit Breaker.
   - **Trilha B (Motor & Algoritmos):** `RulesEngineService`, paridade estrita, barreiras de ciclo, PPL, desmarcação em cascata e testes determinísticos.
   - **Trilha C (Frontend & UI):** Next.js App Router, Tailwind Dark Mode, Zustand, telas, componentes, acessibilidade WCAG AA e testes E2E.
2. **Padrão de Branches:**
   - `main`: Produção / estável.
   - `develop`: Integração contínua da sprint corrente.
   - `feat/sprint-<X>-trilha-<nome-curto>`: Branches individuais de trabalho.
3. **Pull Requests e Revisão por Pares:**
   - Nenhum merge direto em `main` ou `develop`.
   - Cada PR exige aprovação de ao menos 1 membro da equipe.
   - Testes e typecheck validados via terminal com `rtk` antes da aprovação.
4. **Mensagens de Commit:**
   - Obrigatoriamente em português, seguindo o padrão Conventional Commits (ex: `feat: adicionar trava financeira fail-closed`, `fix: corrigir calculo de PPL no motor de regras`).

---

## 4. Matriz de Impacto e Sincronização Obrigatória

| Componente / Área Alterada | Documentos que DEVEM ser Atualizados |
| :--- | :--- |
| **Modelos de Dados & Schemas** (`schema.prisma`, migrations) | [architecture/data-model.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/architecture/data-model.md), [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md) e [specs/api-matricula.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/specs/api-matricula.md) |
| **Autenticação, JWT & Trava Financeira** (`AuthModule`, `CircuitBreaker`) | [integrations/auth-financeiro.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/integrations/auth-financeiro.md), [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md) e [specs/auth-e-financeiro.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/specs/auth-e-financeiro.md) |
| **Regras Acadêmicas & Algoritmos** (`RulesEngineService`, P0 a P5) | [product/rules-engine.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/product/rules-engine.md), [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md) e [specs/motor-de-regras.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/specs/motor-de-regras.md) |
| **Endpoints REST, Payloads & Boletos** (`MatriculaController`, DTOs) | [integrations/api-matricula.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/integrations/api-matricula.md), [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md) e [specs/api-matricula.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/specs/api-matricula.md) |
| **Layouts, Telas e Rotas** (`/login`, `/matricula`, `/divida/liquidar`) | [pages/](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/pages/), [design/responsive.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/design/responsive.md) e [specs/ui-fluxo-estudante.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/specs/ui-fluxo-estudante.md) |
| **Componentes de UI** (`TabelaGrupo1`, `TabelaGrupo2`, `BarraFixa`) | [components/](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/components/), [design/tokens.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/design/tokens.md) e [specs/ui-fluxo-estudante.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/specs/ui-fluxo-estudante.md) |
| **Microcopy & Mensagens** | [copywriting/microcopy.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/copywriting/microcopy.md) |
| **Acessibilidade e Segurança** | [audits/accessibility-and-security.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/audits/accessibility-and-security.md) |
| **Novas Sprints / Mudança de Escopo / Roadmap** | [docs/README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/README.md) e [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md) |

---

## 5. Checklist de Fechamento de Sprint e Tarefa

Antes de realizar o commit e merge de uma sprint, responda e valide os seguintes pontos:
1. **Quais ficheiros de código foram alterados?** (Garantir que apenas os arquivos autorizados no plano foram tocados).
2. **Que comportamento, layout ou regra foi modificada?** (Validar fidelidade aos critérios de aceitação).
3. **Quais documentos da matriz foram atualizados?** (Confirmar sincronização em `/docs` e no `README.md`).
4. **Nomes de variáveis, rotas, tipos e payloads coincidem rigorosamente entre docs e código?** (Sem discrepâncias de tipo ou contrato).
5. **Existe alguma alteração no projeto que não esteja refletida em `/docs` ou no `README`?**  
   > *(A resposta obrigatória deve ser: "Não, todas as alterações estão 100% refletidas na documentação oficial.")*
