# Documento de Especificação Técnica: Portal de Matrículas Inteligente (PRD)

> **Fonte da Verdade Oficial do Projeto**  
> Este documento consolida a especificação técnica integral, arquitetura de software, modelo de dados, contratos de API, regras algorítmicas acadêmicas e premissas de interface para o Portal de Matrículas Inteligente.

---

## 1. Visão Geral

O sistema consiste em um portal web para gestão de inscrições acadêmicas semestrais. Sua função principal é automatizar o processo de matrícula, garantindo que todas as regras de precedência, barreiras de ciclo e requisitos financeiros sejam cumpridos rigorosamente, sem intervenção humana manual, exceto em casos excepcionais (ex: calouros).

---

## 2. Stack Tecnológica & Resiliência

- **Backend:** Node.js com NestJS (TypeScript).
- **Frontend:** Next.js 14+ (App Router), Tailwind CSS.
- **Banco de Dados:** PostgreSQL.
- **ORM:** Prisma (Configuração Multi-Client: `PrismaExternalClient` para consultas e `PrismaInternalClient` para fluxo).
- **Comunicação:** REST API com autenticação via JWT.
- **Geração de Documentos:** Serviço de PDF (`pdfmake` / `puppeteer-core`) para emissão de boletos e comprovantes.

### 2.1 Mecanismo de Idempotência e Resiliência
- **Idempotência de Submissão:** O endpoint `POST /matricula/submeter` obriga o envio do cabeçalho HTTP `Idempotency-Key` (UUIDv4 gerado pelo cliente ao renderizar a tela). Isso previne duplicidade de faturas ou inscrições concorrentes causadas por duplo clique ou instabilidade de rede.
- **Biblioteca de Circuit Breaker:** Especificação formal da biblioteca `opossum` no NestJS para implementar o padrão de disjuntor com timeout configurável (3000ms), limiar de erro de 50%, reset timeout de 15000ms e fallback conservador (*fail-closed*).

---

## 3. Arquitetura de Dados (Dual-Database & Prisma Schema)

O sistema opera entre dois ambientes distintos:

1. **Banco Externo (Fonte da Verdade - Read-Only):** Sistema legado da instituição. Contém histórico acadêmico, matriz curricular oficial e saldos devedores. O portal apenas lê estes dados:
   - `alunos_externo`: `(id, codigo_estudante, senha_hash, ano_ingresso, ano_curricular_atual, semestre_curricular_atual, status)`
   - `historico_externo`: `(id, aluno_id, cadeira_id, resultado, ano_lectivo)`
   - `contas_correntes`: `(id, aluno_id, saldo_devedor, atualizado_em)`

2. **Banco Interno (Controle de Fluxo):** Gerencia o estado das sessões, as inscrições em rascunho (pendentes), logs de boletos gerados e cache de segurança. O portal lê e escreve neste banco.

### 3.1 Esquema do Banco Interno (`schema.prisma`)

As 5 entidades oficiais validadas para o banco PostgreSQL interno:

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

## 4. Regras de Negócio e Motor Acadêmico (Rules Engine)

O coração do sistema é o `RulesEngineService` (Motor de Regras), que processa as seguintes diretrizes em ordem rigorosa de prioridade:

```
[P0: Trava Financeira] -> [P1: Validação Calouro] -> [P2: Paridade Estrita] 
                       -> [P3: Barreira de Ciclo] -> [P4: Próximo Passo Lógico] 
                       -> [P5: Precedências & Injeção Cascata]
```

### 4.1 Prioridade 0: Trava Financeira & Estrutura de Claims JWT
- **Ação:** Logo após a autenticação, o sistema consulta o saldo devedor no Banco Externo através do Circuit Breaker.
- **Bloqueio:** Se `saldo_devedor > 0`, o aluno é impedido de acessar a matrícula.
- **Interface:** Exibe-se apenas a opção "Imprimir Boleto de Liquidação de Dívida".
- **Estrutura de Claims JWT:**
  - O endpoint `POST /auth/login` emite explicitamente a claim `acesso`: `"SOMENTE_BOLETO"` ou `"LIVRE"`.
  - **Guarda de Rotas (Next.js Middleware):** Usuários com `claim: "SOMENTE_BOLETO"` são barrados em nível de middleware caso tentem acessar `/matricula/*` via URL direta, sendo compulsoriamente redirecionados para a tela de liquidação (`/divida/liquidar`).

### 4.2 Prioridade 1: Validação de Calouro
- Estudantes no 1º Ano e 1º Semestre têm inscrição online bloqueada:
  ```typescript
  if (aluno.anoCurricularAtual === 1 && aluno.semestreCurricularAtual === 1) {
    throw new ForbiddenException("Matrícula de calouros é manual. Dirija-se à secretaria acadêmica.");
  }
  ```

### 4.3 Prioridade 2: Regra de Paridade (Sincronia Semestral)
- O portal opera em modo **PAR (2)** ou **ÍMPAR (1)**.
- Só são oferecidas cadeiras cujo semestre curricular coincida com a paridade do período de inscrições vigente:
  $$\text{CadeirasElegiveis} = \{ c \in \text{Catalogo} \mid (c.\text{semestre} \pmod 2) \equiv (\text{CicloAtivo} \pmod 2) \}$$

### 4.4 Prioridade 3: Regra de Barreira de Ciclo (Hard Stop)
Impede o avanço desordenado entre anos curriculares:
- **Barreira 1:** Inscrição em cadeiras do 3º Ano exige 100% de aprovação no 1º Ano.
- **Barreira 2:** Inscrição em cadeiras do 4º Ano exige 100% de aprovação no 2º Ano.
- **Consequência:** Se o aluno falhar na barreira, o sistema impede a seleção de qualquer cadeira do ano avançado.

### 4.5 Prioridade 4: Regra de Sequência Lógica (PPL - Próximo Passo Lógico)
- **Proibição de Saltos:** O aluno não pode pular semestres.
- Exemplo: Se o último semestre concluído foi Ano 2/Semestre 1, o sistema define o Ano 2/Semestre 2 como o alvo regular. Se o portal estiver em período de Semestre 1, o aluno fica "desfasado" e só pode cursar cadeiras em atraso (de Semestre 1), devendo esperar o próximo ciclo para retomar o Ano 2/Semestre 2.

### 4.6 Prioridade 5: Regra de Precedência (Cadeira a Cadeira) & Injeção Automática no Grupo 2
- Se a Cadeira B exige a Cadeira A, e o aluno não aprovou na Cadeira A:
  - A Cadeira B aparece como **Bloqueada** na interface com o motivo de bloqueio explícito.
- **Detalhamento Algorítmico de Injeção Automática no Grupo 2:**
  - Quando uma cadeira do **Grupo 1** estiver bloqueada por falta de precedência, o motor verifica se a disciplina precedente pertence à paridade corrente.
  - Caso pertença, a cadeira em falta deve ser **injetada automaticamente no Grupo 2** marcada por padrão (`checked = true`), dando ao aluno a oportunidade imediata de regularizar o pré-requisito no mesmo semestre acadêmico.

---

## 5. Dinâmica de Inscrição e Interface (UX)

A tela de matrícula é dividida em dois grupos dinâmicos:

### 5.1 Grupo 1: Cadeiras Regulares
- Definidas pelo PPL (Próximo Passo Lógico).
- **Comportamento:** Inscrição obrigatória. Checkbox marcado e travado (`checked = true, disabled = true`).
- **Bloqueio:** Se faltar precedência, a cadeira aparece desmarcada e bloqueada com ícone de cadeado (`bloqueada = true, checked = false`).

### 5.2 Grupo 2: Cadeiras em Atraso (Dependências)
- Cadeiras reprovadas/pendentes de anos anteriores que respeitam a paridade ativa e as barreiras de ciclo.
- **Comportamento:** Opcionalidade. Checkbox marcado por padrão, mas habilitado para desmarcar (`checked = true, disabled = false`).
- **Interdependência (Desmarcação em Cascata em Tempo Real):** Se o aluno desmarcar uma cadeira deste grupo que seja base de uma regular do Grupo 1, o sistema desmarca e bloqueia a regular automaticamente em tempo real:
  $$\text{Desmarcar}(C_{\text{Atraso}} \in \text{Grupo 2}) \implies \forall C_{\text{Regular}} \in \text{Grupo 1} \mid C_{\text{Atraso}} \in \text{Precedencias}(C_{\text{Regular}}), \quad C_{\text{Regular}}.\text{checked} \leftarrow \text{false}$$

### 5.3 Simulação Financeira
- O valor total do boleto deve ser recalculado no frontend e revalidado no backend (`POST /matricula/simular`) sempre que uma cadeira do Grupo 2 for alterada (com debounce).

---

## 6. Fluxo de Processamento & Contrato de Endpoints da API

### 6.1 Fluxo de Processamento (Step-by-Step)
1. **Login:** API autentica, valida se é calouro (1º Ano/1º Sem bloqueados) e emite JWT com claim (`SOMENTE_BOLETO` ou `LIVRE`).
2. **Check Financeiro:** API valida saldo no Banco Externo via Circuit Breaker (`opossum`).
3. **Processamento de Grade:** Motor de regras gera a lista de cadeiras permitidas (Grupo 1 fixo / Grupo 2 opcional com injeção de precedências).
4. **Seleção:** Aluno interage com a interface (Grupo 1 fixo / Grupo 2 opcional, disparo de cascata e debounce de simulação).
5. **Submissão:** O backend revalida todas as regras (Barreira, Paridade, Precedência) de forma Zero-Trust com o cabeçalho `Idempotency-Key` e grava no Banco Interno com status `PENDENTE` em transação atômica ACID.
6. **Pagamento:** O aluno imprime o boleto gerado.
7. **Efetivação:** No próximo login do aluno, o sistema checa o Banco Externo. Se o saldo for 0, a inscrição `PENDENTE` no Banco Interno é movida atômica e reativamente para `EFECTIVADA`.

### 6.2 Contrato Oficial de Endpoints da API NestJS

| Método | Rota | Headers / Payload | Resposta | Descrição |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/auth/login` | Body: `{ codigo, senha }` | `200 OK` `{ accessToken, claimAcesso, aluno }` | Autentica, checa calouro, avalia débito externo e emite JWT com claim. |
| `GET` | `/financeiro/pendencia` | `Authorization: Bearer <token>` | `200 OK` `{ estudanteCodigo, saldoDevedor, emFallback, boletoLiquidacaoUrl }` | Retorna dados e URL para emissão do boleto da dívida. |
| `GET` | `/matricula/opcoes` | `Authorization: Bearer <token>` | `200 OK` `{ cicloAtivo, grupo1: [], grupo2: [] }` | Retorna o payload processado: Grupo 1 (travadas/bloqueadas) e Grupo 2. |
| `POST` | `/matricula/simular` | Body: `{ cadeirasIds: number[] }` | `200 OK` `{ valorTotal, itensValidos, advertencias: [] }` | Recebe as cadeiras e recalcula total e integridade em tempo real. |
| `POST` | `/matricula/submeter` | Header: `Idempotency-Key: <UUIDv4>`<br>Body: `{ cadeirasIds: number[] }` | `201 Created` `{ matriculaId, status, valorTotal, boletoUrl, codigoBarras }` | Revalida regras no servidor, grava inscrição `PENDENTE` e emite boleto. |
| `GET` | `/matricula/:id/boleto` | `Authorization: Bearer <token>` | `200 OK` Stream `application/pdf` | Retorna o stream ou link assinado do PDF gerado. |

---

## 7. Requisitos de Segurança e Resiliência

- **Não-Confiabilidade do Client (Zero-Trust):** O servidor deve ignorar os cálculos de valor vindos do frontend e refazer toda a validação acadêmica no momento do `POST /matricula/submeter`.
- **Circuit Breaker (`opossum`):** Se o Banco Externo estiver fora do ar, o sistema deve usar o último cache de saldo do Banco Interno (`PendenciaCache`) e emitir um aviso de "Modo de Segurança".
- **Política de Fail-Closed no Circuit Breaker:** Se o Banco Externo estiver inacessível e **não houver** registro prévio em `PendenciaCache`, o sistema adota a política estrita de **Fail-Closed** (bloqueia o avanço por omissão e instrui o aluno a tentar mais tarde ou procurar a secretaria, nunca liberando acesso `LIVRE` por falta de resposta da rede).
- **Audit Log:** Todas as tentativas de desmarcar cadeiras de dependência devem ser registradas para fins de auditoria acadêmica.

---

## 8. Identidade Visual (UI)

- **Tema:** Dark Mode Premium
  - Fundo da Aplicação: `#0A0F1A`
  - Painéis e Cartões: `#111B2D`
  - Superfícies Elevadas/Modais: `#18253E`
  - Destaques e Acentos: Branco (`#FFFFFF`) e Cinza Visível (`#94A3B8` / `#CBD5E1`)
  - Status e Alertas: Vermelho (`#EF4444`) para bloqueios e Verde (`#10B981`) para confirmação
- **Animação de Texto:** Ao carregar seções de "História" ou "Resumo", as letras devem aparecer em cinza escuro e ganhar a cor branca gradualmente em efeito de digitação conforme o scroll da tela.
- **Acessibilidade e Usabilidade:** Conformidade com **WCAG 2.1 AA**, contraste elevado, foco visível e responsividade total para Mobile, Tablet e Desktop.
