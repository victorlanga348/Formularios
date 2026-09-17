# Sistema de Matrícula e Inscrição Académica (Consolidado e Retificado)

---

### 0. Visão Geral e Princípios Arquiteturais

O sistema gere integralmente o ciclo de inscrição semestral do estudante universitário: autenticação com verificação de papéis, conciliação financeira síncrona/reativa com banco legado, aplicação estrita de regras acadêmicas em cascata (Paridade, Barreira de Ciclo, Próximo Passo Lógico – PPL e Precedências) e emissão de boletos via pipeline transacional.

A arquitetura orienta-se por três pilares inegociáveis:

* **Motor de Regras Acadêmicas Puro (Zero-Trust):** O servidor jamais confia no payload de seleção enviado pelo cliente. Toda combinação de cadeiras é recalculada a partir do histórico acadêmico imutável antes da gravação.
* **Dualidade de Bancos com Prisma Multi-Client:** Isolamento físico entre o **Banco Externo** (fonte primária e autoritativa de identidade e saldos financeiros) e o **Banco Interno PostgreSQL** (controle transacional de fluxo, sessões, caches e inscrições).
* **Reconciliação no Próximo Login:** O aluno em débito é confinado à liquidação de pendências financeiras. A efetivação de uma matrícula previamente pendente ocorre de forma automática e reativa no momento em que um novo login detecta saldo zero no banco externo.

---

### 1. Stack Tecnológica Oficial (Retificada)

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

#### 2.1 Banco Externo (Referência Read-Only via `PrismaExternal`)

Mapeado de forma estrita para consulta de validação de calouro e débitos financeiros:

* `alunos_externo`: `(id, codigo_estudante, senha_hash, ano_ingresso, ano_curricular_atual, semestre_curricular_atual, status)`.
* `historico_externo`: `(id, aluno_id, cadeira_id, resultado, ano_lectivo)`.
* `contas_correntes`: `(id, aluno_id, saldo_devedor, atualizado_em)`.

#### 2.2 Banco Interno (Esquema PostgreSQL Oficial via `PrismaInternal`)

```prisma
datasource db {
  provider = "postgresql"
  url      = env("INTERNAL_DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
  output   = "../node_modules/@prisma/client-internal"
}

enum ClaimAcesso {
  SOMENTE_BOLETO
  LIVRE
}

enum InscricaoStatus {
  PENDENTE
  EFECTIVADA
  CANCELADA
}

enum GrupoOrigem {
  REGULAR
  ATRASO
}

model EstudanteSessao {
  id              String             @id @default(uuid())
  estudanteCodigo String             @unique
  claimAcesso     ClaimAcesso
  ultimoLogin     DateTime           @default(now())
  createdAt       DateTime           @default(now())
  inscricoes      InscricaoPendente[]
  pendenciaCache  PendenciaCache?

  @@index([estudanteCodigo])
}

model PendenciaCache {
  id                   String          @id @default(uuid())
  estudanteCodigo      String          @unique
  ultimoSaldoConhecido Decimal         @db.Decimal(12, 2)
  emFallback           Boolean         @default(false)
  atualizadoEm         DateTime        @updatedAt
  estudanteSessao      EstudanteSessao @relation(fields: [estudanteCodigo], references: [estudanteCodigo], onDelete: Cascade)
}

model InscricaoPendente {
  id                 String          @id @default(uuid())
  estudanteCodigo    String
  estudanteSessaoId  String
  semestreReferencia String          // Ex: "2026.1"
  status             InscricaoStatus @default(PENDENTE)
  valorTotal         Decimal         @db.Decimal(12, 2)
  idempotencyKey     String          @unique
  createdAt          DateTime        @default(now())
  updatedAt          DateTime        @updatedAt

  estudanteSessao    EstudanteSessao @relation(fields: [estudanteSessaoId], references: [id])
  itens              InscricaoItem[]
  boleto             BoletoLog?

  @@index([estudanteCodigo, status])
}

model InscricaoItem {
  id                  String            @id @default(uuid())
  inscricaoPendenteId String
  cadeiraId           Int
  grupoOrigem         GrupoOrigem
  taxaAplicada        Decimal           @db.Decimal(12, 2)

  inscricao           InscricaoPendente @relation(fields: [inscricaoPendenteId], references: [id], onDelete: Cascade)

  @@index([inscricaoPendenteId])
}

model BoletoLog {
  id                  String            @id @default(uuid())
  inscricaoPendenteId String            @unique
  codigoBarras        String
  pdfPath             String
  geradoEm            DateTime          @default(now())
  expiraEm            DateTime

  inscricao           InscricaoPendente @relation(fields: [inscricaoPendenteId], references: [id], onDelete: Cascade)
}
```

---

### 3. Motor de Regras – Especificação Algorítmica Oficial

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

Executada imediatamente na autenticação:

```typescript
async function verificarTravaFinanceira(codigoEstudante: string): Promise<ClaimAcesso> {
  try {
    const saldoExterno = await circuitBreaker.fire(codigoEstudante); // Consulta PrismaExternal
    await prismaInternal.pendenciaCache.upsert({
      where: { estudanteCodigo: codigoEstudante },
      create: { estudanteCodigo: codigoEstudante, ultimoSaldoConhecido: saldoExterno, emFallback: false },
      update: { ultimoSaldoConhecido: saldoExterno, emFallback: false }
    });

    if (saldoExterno > 0) {
      return ClaimAcesso.SOMENTE_BOLETO;
    }

    // Sincronização e Efetivação Automática Pós-Pagamento
    await prismaInternal.inscricaoPendente.updateMany({
      where: { estudanteCodigo: codigoEstudante, status: InscricaoStatus.PENDENTE },
      data: { status: InscricaoStatus.EFECTIVADA }
    });

    return ClaimAcesso.LIVRE;
  } catch (error) {
    // Fallback Conservador: Nunca liberar por omissão
    const cache = await prismaInternal.pendenciaCache.findUnique({ where: { estudanteCodigo: codigoEstudante } });
    if (!cache || cache.ultimoSaldoConhecido.toNumber() > 0) {
      return ClaimAcesso.SOMENTE_BOLETO;
    }
    return ClaimAcesso.SOMENTE_BOLETO; // Política de fail-closed
  }
}
```

#### Regra 4.2 – Validação de Calouro

```typescript
if (aluno.anoCurricularAtual === 1 && aluno.semestreCurricularAtual === 1) {
  throw new ForbiddenException("Matrícula de calouros é manual. Dirija-se à secretaria acadêmica.");
}
```

#### Regra 4.3 – Paridade Estrita

O portal opera em regime de semestres **Ímpares (1)** ou **Pares (2)**. Cadeiras fora da paridade ativa são suprimidas na raiz da árvore de decisão:

$$\text{CadeirasElegiveis} = \{ c \in \text{Catalogo} \mid (c.\text{semestre} \pmod 2) \equiv (\text{CicloAtivo} \pmod 2) \}$$

#### Regra 4.4 – Barreira de Ciclo (Hard Stop)

Bloqueia verticalmente a transição de ciclo antes da conclusão total das etapas base:

* Para ofertar cadeiras do **3º Ano**: $100\%$ das cadeiras do **1º Ano** devem constar como `APROVADO` no histórico.
* Para ofertar cadeiras do **4º Ano**: $100\%$ das cadeiras do **2º Ano** devem constar como `APROVADO` no histórico.
* Caso a condição não seja atendida, o aluno não tem acesso a cadeiras do ciclo avançado, ficando restrito a regularizar pendências.

#### Regra 4.5 – Próximo Passo Lógico (PPL)

Veda a progressão arbitrária entre semestres curriculares. O próximo semestre acadêmico teórico ($S_{\text{logico}}$) é derivado sequencialmente a partir do último semestre concluído com êxito:

```typescript
function calcularProximoSemestreLogico(aluno: HistoricoAluno): { ano: number; semestre: number } {
  const { ultimoAnoConcluido, ultimoSemestreConcluido } = aluno;
  if (ultimoSemestreConcluido === 1) {
    return { ano: ultimoAnoConcluido, semestre: 2 };
  }
  return { ano: ultimoAnoConcluido + 1, semestre: 1 };
}
```

* Se $(S_{\text{logico}} \pmod 2) \neq (\text{CicloAtivo} \pmod 2)$, o **Grupo 1 (Regulares)** permanece vazio. O estudante poderá cursar exclusivamente cadeiras em atraso no **Grupo 2** que correspondam à paridade atual.

#### Regra 4.6 & 4.7 – Inscrição Híbrida, Injeção Automática e Desmarcação em Cascata

* **Grupo 1 (Regulares):** Cadeiras do $S_{\text{logico}}$. Apresentam-se marcadas e travadas por padrão (`checked = true, disabled = true`). Se uma cadeira possuir precedência não aprovada no histórico, ela é bloqueada (`bloqueada = true, checked = false`).
* **Grupo 2 (Atrasadas):** Cadeiras reprovadas/pendentes de semestres anteriores que atendem à paridade corrente. Vêm marcadas, mas habilitadas para edição (`checked = true, disabled = false`).
* **Injeção Automática por Precedência em Falta:** Quando uma cadeira do Grupo 1 estiver bloqueada por falta de precedência, o motor verifica se a disciplina precedente pertence à paridade do ciclo ativo. Caso pertença, a cadeira em falta é **injetada automaticamente no Grupo 2** marcada por padrão (`checked = true`), oportunizando a regularização imediata do pré-requisito.
* **Cascata Direta:**

$$\text{Desmarcar}(C_{\text{Atraso}} \in \text{Grupo 2}) \implies \forall C_{\text{Regular}} \in \text{Grupo 1} \mid C_{\text{Atraso}} \in \text{Precedencias}(C_{\text{Regular}}), \quad C_{\text{Regular}}.\text{checked} \leftarrow \text{false}$$

---

### 4. Endpoints REST Oficiais (NestJS Controllers)

| Método | Rota | Payload / Headers | Resposta | Descrição |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/auth/login` | `{ codigo, senha }` | `{ accessToken, claimAcesso, aluno }` | JWT com claims de acesso (`SOMENTE_BOLETO` ou `LIVRE`) |
| `GET` | `/financeiro/pendencia` | `Bearer Token` | `{ saldoDevedor, boletoLiquidacaoUrl, emFallback }` | Consulta de saldo devedor e link de regularização |
| `GET` | `/matricula/opcoes` | `Bearer Token` | `{ cicloAtivo, grupo1: [], grupo2: [] }` | Listagem das cadeiras elegíveis divididas em Grupo 1 e 2 |
| `POST` | `/matricula/simular` | `{ cadeirasIds: number[] }` | `{ valorTotal, itensValidos, advertencias: [] }` | Simulação em tempo real com recálculo determinístico |
| `POST` | `/matricula/submeter` | Header: `Idempotency-Key`<br>Payload: `{ cadeirasIds: number[] }` | `{ matriculaId, boletoUrl, status, total }` | Criação atômica da matrícula pendente com transação ACID |
| `GET` | `/matricula/:id/boleto` | `Bearer Token` | Stream do PDF (`application/pdf`) | Download/stream do documento e boleto emitido |

---

### 5. Comportamento da Interface Web (Next.js App Router)

* **Layout e Proteção de Rota:** `middleware.ts` intercepta as requisições via JWT. Estudantes com `acesso: "SOMENTE_BOLETO"` são redirecionados compulsoriamente para `/divida/liquidar`, sem acesso às rotas de `/matricula/*`.
* **Grupo 1 (Regulares):** Tabela superior. Cadeiras válidas exibem `checkbox` selecionado e desabilitado. Cadeiras com precedência ausente exibem badge visual vermelho, ícone de cadeado e tooltip explicativo.
* **Grupo 2 (Atrasos Opcionais):** Tabela secundária interativa. A alternância de estado (desmarcação) engatilha o recálculo do estado cliente (desmarcando regulares dependentes) e dispara um `debounce` para o endpoint `POST /matricula/simular`.
* **Barra de Ação Fixa (Rodapé):** Exibe a somatória das taxas em tempo real com indicador de loading. O botão `"Confirmar Matrícula"` inclui geração de hash UUID único para o header `Idempotency-Key` no momento da submissão.
* **Design System & Acessibilidade:** Interface construída em Dark Mode com tokens de alta legibilidade, contraste WCAG 2.1 AA, navegação fluida por teclado e suporte a leitores de ecrã.

---

### 6. Organização do Trabalho em Grupo e Divisão em Sprints

Para viabilizar o desenvolvimento colaborativo de forma paralela e sem conflitos de branch, as tarefas são organizadas em **3 Trilhas de Trabalho Paralelas (Tracks)** distribuídas pelas semanas de sprint:

* **Trilha A (Backend & Persistência):** Responsável por schemas Prisma, migrations, NestJS modules, autenticação, transações ACID e geração de PDFs.
* **Trilha B (Motor de Regras & Algoritmos):** Responsável pela implementação pura de `RulesEngineService`, testes unitários exaustivos do motor, cálculo de taxas e simulação.
* **Trilha C (Frontend & Experiência do Aluno):** Responsável por Next.js App Router, Tailwind Dark Mode, Zustand, componentes acessíveis, integração de APIs e testes E2E.

#### Estrutura de Branches e Colaboração
- `main`: Código em produção, sempre estável e testado.
- `develop`: Branch de integração contínua do time.
- `feat/<sprint>-<trilha>-<funcionalidade>` (ex: `feat/sprint-1-backend-prisma-schemas`, `feat/sprint-3-frontend-tabelas-selecao`).
- Pull Requests obrigatórios para merge em `develop`, contendo aprovação de pelo menos um par do time e execução de testes via `rtk`.
- Mensagens de commit padronizadas em português (ex: `feat: adicionar circuit breaker opossum no modulo financeiro`).

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

---

### 8. Estrutura do Repositório e Documentação

O repositório é governado pelas especificações detalhadas localizadas no diretório `/docs`:

```
/
├── README.md                      # Fonte oficial mestre de arquitetura e cronograma
├── AGENTS.md                      # Protocolo operacional e diretrizes de desenvolvimento
├── GEMINI.md                      # Regras de governança do assistente
├── docs/
│   ├── PRD.md                     # Documento de Definição de Produto e Arquitetura (Fonte da Verdade)
│   ├── README.md                  # Índice mestre da documentação
│   ├── documentation-governance.md # Matriz de impacto e governança documental
│   ├── tasks/
│   │   └── template.md            # Template oficial para especificação de tarefas/sprints
│   └── specs/
│       ├── auth-e-financeiro.md   # Especificação do AuthModule, Circuit Breaker e Trava P0
│       ├── motor-de-regras.md     # Especificação do RulesEngineService e regras P1 a P5
│       ├── api-matricula.md       # Contratos de rotas REST, DTOs e transações ACID
│       └── ui-fluxo-estudante.md  # Especificação da interface Next.js, Zustand e acessibilidade
```
