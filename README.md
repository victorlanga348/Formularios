# Sistema de Inscrição

---

### 0. Visão Geral e Princípios Arquiteturais

O sistema gere integralmente o ciclo de inscrição semestral do estudante universitário: autenticação com verificação de papéis, conciliação financeira síncrona/reativa com banco legado, aplicação estrita de regras acadêmicas em cascata (Paridade, Barreira de Ciclo, Próximo Passo Lógico – PPL e Precedências) e emissão de boletos via pipeline transacional.

A arquitetura orienta-se por três pilares inegociáveis:

* **Motor de Regras Acadêmicas Puro (Zero-Trust):** O servidor jamais confia no payload de seleção enviado pelo cliente. Toda combinação de cadeiras é recalculada a partir do histórico acadêmico imutável antes da gravação.
* **Dualidade de Bancos com Prisma Multi-Client:** Isolamento físico entre o **Banco Externo** (fonte primária e autoritativa de identidade e saldos financeiros) e o **Banco Interno PostgreSQL** (controle transacional de fluxo, sessões, caches e inscrições).
* **Reconciliação no Próximo Login:** O aluno em débito é confinado à liquidação de pendências financeiras. A efetivação de uma matrícula previamente pendente ocorre de forma automática e reativa no momento em que um novo login detecta saldo zero no banco externo.

---

### 1. Stack Tecnológica

| Camada | Tecnologia Adotada | Justificação Técnica |
| :--- | :--- | :--- |
| **Backend** | **Node.js + NestJS (TypeScript)** | Modularidade corporativa orientada a injeção de dependência (`RulesEngineModule`, `FinanceiroModule`), facilitando testes unitários determinísticos do motor de regras. |
| **Frontend** | **Next.js 14+ (App Router) + Tailwind CSS** | Server Components com streaming de layout, Route Handlers velozes, tipagem TypeScript ponta a ponta e UI com estética de alta legibilidade operacional. |
| **Banco de Dados** | **PostgreSQL** | Integridade referencial forte, controle estrito de concorrência com transações ACID via `$transaction` do Prisma. |
| **ORM / Acesso a Dados** | **Prisma Multi-Client** | Geração isolada de `PrismaInternalClient` (PostgreSQL local) e `PrismaExternalClient` (Banco Legado Read-Only). Elimina mapeamentos SQL manuais mantendo tipagem segura. |
| **Geração de PDF** | **Serviço NestJS (`pdfmake` / `puppeteer-core`)** | Geração assíncrona de boletos e metadados com streaming de buffer para armazenamento em bucket/volume e persistência em `BoletoLog`. |
| **Autenticação & Sessão** | **NestJS Passport + JWT** | Claims dinâmicos (`acesso: "SOMENTE_BOLETO" | "LIVRE"`), recalculados no ciclo de login e protegidos por `JwtAuthGuard` customizado. |
| **Resiliência Externa** | **Circuit Breaker (Opossum)** | Monitora conexões com a API/DB externa. Em caso de falha contínua, ativa estado *Aberto* com fallback para o último saldo cacheado localmente. |

---

### 2. Modelo de Dados

#### 2.1 Banco Externo
Usado como fonte das verificações para teste.

#### 2.2 Banco Interno
O banco real do projecto que vai guardar os estudantes matriculados com sucesso.

---

### 3. Motor de Regras

As validações são processadas de forma encadeada no backend (`RulesEngineService`). Qualquer incoerência aborta a montagem ou o submit da matrícula.

```
+-------------------------------------------------------------------------------+
|                       RulesEngineService: Pipeline                            |
|                                                                               |
|  [P0: Trava Financeira]                                                       |
|           |                                                                   |
|           v                                                                   |
|  [P1: Validação de Calouro] (Ano 1 / Sem 1 -> Bloqueio Manual)                |
|           |                                                                   |
|           v                                                                   |
|  [P2: Paridade Estrita] (Filtro por Época: Ímpar vs Par)                     |
|           |                                                                   |
|           v                                                                   |
|  [P3: Barreira de Ciclo] (Ano 1 limpo -> Libera Ano 3 | Ano 2 -> Libera Ano 4) |
|           |                                                                   |
|           v                                                                   |
|  [P4: Próximo Passo Lógico - PPL] (Semestre sequencial; veda saltos de etapa) |
|           |                                                                   |
|           v                                                                   |
|  [P5: Precedências & Injeção Cascata] (Cadeiras do Grupo 1 e Grupo 2)        |
+-------------------------------------------------------------------------------+
```

#### Regra 4.1 – Prioridade 0: Trava Financeira
Alunos com dividas não poderão acessar e nem realizar nada enquanto estiverem com dívidas.

#### Regra 4.2 – Validação de Calouro
Alunos de 1 ano do 1 semestre nao poderão fazer a inscrição nesse ssitemas, até agora nessa fase é só presencial.

#### Regra 4.3 – Paridade Estrita

O portal opera em regime de semestres **Ímpares (1)** ou **Pares (2)**. Cadeiras fora da paridade ativa são suprimidas na raiz da árvore de decisão:

#### Regra 4.4 – Barreira de Ciclo

Bloqueia verticalmente a transição de ciclo antes da conclusão total das etapas base:

* Para ofertar cadeiras do **3º Ano**: $100\%$ das cadeiras do **1º Ano** devem constar como `APROVADO` no histórico.
* Para ofertar cadeiras do **4º Ano**: $100\%$ das cadeiras do **2º Ano** devem constar como `APROVADO` no histórico.
* Caso a condição não seja atendida, o aluno não tem acesso a cadeiras do ciclo avançado, ficando restrito a regularizar pendências.

#### Regra 4.5 – Próximo Passo Lógico
Os alunos não vão poder pular semestres, ex: um aluno que fez 1 ano 2 semestre e quere fazer isncrição para 2 ano e 2 semestre vai ser barrado pois ainda não concluiu o 2 ano 1 semestre.

#### Regra 4.6 & 4.7 – Inscrição Híbrida, Injeção Automática e Desmarcação em Cascata

* **Grupo 1 (Regulares):** As Cadeiras Apresentam-se marcadas e travadas por padrão (`checked = true, disabled = true`). Se uma cadeira possuir precedência não aprovada no histórico, ela é bloqueada (`bloqueada = true, checked = false`).
* **Grupo 2 (Atrasadas):** As Cadeiras reprovadas/pendentes de semestres anteriores que atendem à paridade corrente. Vêm marcadas, mas habilitadas para edição (`checked = true, disabled = false`).
* **Injeção Automática por Precedência em Falta:** Quando uma cadeira do Grupo 1 estiver bloqueada por falta de precedência, o motor verifica se a disciplina precedente pertence à paridade do ciclo ativo. Caso pertença, a cadeira em falta é **injetada automaticamente no Grupo 2** marcada por padrão (`checked = true`), oportunizando a regularização imediata do pré-requisito.
* **Cascata Direta:**

---

### 4. Comportamento da Interface Web (Next.js App Router)

* **Layout e Proteção de Rota:** `middleware.ts` intercepta as requisições via JWT. Estudantes com `acesso: "SOMENTE_BOLETO"` são redirecionados compulsoriamente para `/divida/liquidar`, sem acesso às rotas de `/matricula/*`.
* **Grupo 1 (Regulares):** Tabela superior. Cadeiras válidas exibem `checkbox` selecionado e desabilitado. Cadeiras com precedência ausente exibem badge visual vermelho, ícone de cadeado e tooltip explicativo.
* **Grupo 2 (Atrasos Opcionais):** Tabela secundária interativa. A alternância de estado (desmarcação) engatilha o recálculo do estado cliente (desmarcando regulares dependentes) e dispara um `debounce` para o endpoint `POST /matricula/simular`.
* **Barra de Ação Fixa (Rodapé):** Exibe a somatória das taxas em tempo real com indicador de loading. O botão `"Confirmar Matrícula"` inclui geração de hash UUID único para o header `Idempotency-Key` no momento da submissão.
* **Design System & Acessibilidade:** Interface construída em Dark Mode com tokens de alta legibilidade, contraste WCAG 2.1 AA, navegação fluida por teclado e suporte a leitores de ecrã.

---

### 5. Organização do Trabalho em Grupo e Divisão em Sprints

Para viabilizar o desenvolvimento colaborativo de forma paralela e sem conflitos de branch, as tarefas são organizadas em **3 Trilhas de Trabalho Paralelas (Tracks)** distribuídas pelas semanas de sprint:

* **Trilha A (Backend & Persistência):** Responsável por schemas Prisma, migrations, NestJS modules, autenticação, transações ACID e geração de PDFs.
* **Trilha B (Motor de Regras & Algoritmos):** Responsável pela implementação pura de `RulesEngineService`, testes unitários exaustivos do motor, cálculo de taxas e simulação.
* **Trilha C (Frontend & Experiência do Aluno):** Responsável por Next.js App Router, Tailwind Dark Mode, Zustand, componentes acessíveis, integração de APIs e testes E2E.

---

#### Detalhamento das Sprints de Grupo

```
==================================================================================================
SPRINT / SEMANA   TRILHA A (Backend/DB)         TRILHA B (Motor/Regras)       TRILHA C (Frontend/UI)
==================================================================================================
Semana 1          - Configurar Docker Postgres  - Mapear catálogo de cadeiras - Setup Next.js 14 App Router
Fase 0: Infra &   - Prisma Multi-Client         - Estruturar matriz de        - Tailwind Dark Mode tokens
Multi-Client      - Migrations e Seeds          precedências e semestres      - Componentes base & Layout
                  [DoD: DBs operacionais]       [DoD: JSON de precedências]   [DoD: Shell da UI funcional]
--------------------------------------------------------------------------------------------------
Semana 2          - AuthModule (Passport/JWT)   - Circuit Breaker Opossum     - Telas Login & Bloqueio
Fase 1: Auth &    - Claims dinâmicos de acesso  - Trava Financeira P0         - Middleware.ts (JWT Guards)
Trava Financeira  - Validação de Calouro P1     - Fallback em PendenciaCache  - Rota /divida/liquidar
                  [DoD: Endpoints Auth OK]      [DoD: Testes de resiliência]  [DoD: Redirecionamento OK]
--------------------------------------------------------------------------------------------------
Semana 3          - Endpoints /matricula/opcoes - Implementar P2 (Paridade)   - Store Zustand de seleção
Fase 2: Motor de  - Mock contratos de dados     - Implementar P3 (Ciclos)     - Mock API para consumo
Regras Core       - Setup de DTOs validados     - Implementar P4 (PPL)        - Visualização Grupo 1 vs 2
                  [DoD: Contratos REST prontos] - Implementar P5 (Cascata)    [DoD: Store com mock OK]
                                                [DoD: 100% testes unitários]
--------------------------------------------------------------------------------------------------
Semana 4          - Endpoint /matricula/simular - Validação de payloads       - Tabela Grupo 1 (Travada)
Fase 3: UI &      - Cálculo de taxas backend    - Recálculo determinístico    - Tabela Grupo 2 (Editável)
Reatividade       - Otimização de queries       - Auditoria Zero-Trust        - Cascata reativa no cliente
                  [DoD: Simulação em <100ms]    [DoD: Zero-trust aprovado]    - Debounce e Barra de Ação
                                                                              [DoD: Fluxo visual completo]
--------------------------------------------------------------------------------------------------
Semana 5          - Endpoint /submeter ACID     - Validação final no submit   - Botão Submeter com UUID
Fase 4: Pipeline  - IdempotencyKey handling     - Cálculo de taxas finais     - Modal de confirmação
de Boletos        - Geração PDF (pdfmake)       - Logs de auditoria           - Visualizador de Boleto PDF
                  - Registro em BoletoLog                                     - Tratamento de retentativas
                  [DoD: Transação atômica OK]   [DoD: Sem duplo submit]       [DoD: Download do PDF OK]
--------------------------------------------------------------------------------------------------
Semana 6          - Reconciliação reativa       - Testes de concorrência      - Testes E2E (Playwright)
Fase 5: Testes,   no próximo login (PENDENTE    - Testes de virada de ciclo   - Auditoria de acessibilidade
Reconciliação e   -> EFECTIVADA pós-pagamento)  - Auditoria de segurança      (WCAG 2.1 AA)
Auditoria E2E     [DoD: Reconciliação 100%]     [DoD: 0 vulnerabilidades]     [DoD: E2E verde em CI]
==================================================================================================
```

---

### 7. Matriz de Riscos Operacionais e Mitigações

| Risco Técnico Identificado | Severidade | Impacto | Estratégia de Mitigação |
| :--- | :---: | :--- | :--- |
| **Indisponibilidade do Banco Externo de Débitos** | Crítica | Sistema incapaz de checar se o estudante possui pendência financeira. | Aplicação do Circuit Breaker (Opossum). O sistema assume estado defensivo (*fail-closed*), lendo o `PendenciaCache` local e rejeitando liberações automáticas por timeout. |
| **Manipulação do Payload de Seleção no Cliente** | Crítica | Estudante tenta enviar IDs de cadeiras sem precedências ou fora da paridade. | O backend descarta qualquer cálculo financeiro ou lista pré-aprovada do frontend, revalidando todas as regras (4.3 a 4.7) via `RulesEngineService` no ato do `POST /matricula/submeter`. |
| **Submissão Duplicada de Matrícula (Duplo Clique / Retry)** | Alta | Criação de inscrições duplicadas e cobrança em duplicidade. | Uso de constraint única no banco interno via `idempotencyKey` enviada pelo cliente no cabeçalho HTTP. Retentativas recebem a resposta original em cache sem reinserção. |
| **Discrepância na Efetivação Financeira** | Média | Aluno paga no banco legado, mas o sistema interno não é notificado por falta de webhook. | A conciliação é desacoplada de webhooks frágeis; toda tentativa subsequente de autenticação executa `verificarTravaFinanceira`, promovendo a transição atômica de `PENDENTE` para `EFECTIVADA`. |
| **Conflitos de Merge em Desenvolvimento em Grupo** | Média | Trabalho paralelo de múltiplos membros sobrescrevendo regras ou contratos. | Divisão rígida de trilhas (Backend, Motor, Frontend), uso de contratos de DTO tipados via TypeScript compartilhados, e branch protections com PR obrigatório. |
