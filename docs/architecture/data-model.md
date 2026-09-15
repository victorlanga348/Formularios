# Modelo de Dados e Dualidade de Bancos

## 1. Dualidade de Bancos de Dados
O sistema utiliza **Prisma Multi-Client** para separar fisicamente a consulta aos dados legados corporativos e a persistência transacional interna.

---

## 2. Banco Externo (Referência Read-Only via `PrismaExternalClient`)

Este banco é de posse do sistema corporativo institucional e somente é consultado para validação cadastral e saldo financeiro:

* `alunos_externo`:
  - `id` (PK, Int)
  - `codigo_estudante` (String, Unique)
  - `senha_hash` (String)
  - `ano_ingresso` (Int)
  - `ano_curricular_atual` (Int)
  - `semestre_curricular_atual` (Int)
  - `status` (String: "ACTIVO", "SUSPENSO", "GRADUADO")
* `historico_externo`:
  - `id` (PK, Int)
  - `aluno_id` (FK -> alunos_externo.id)
  - `cadeira_id` (Int)
  - `resultado` (String: "APROVADO", "REPROVADO", "DISPENSADO")
  - `ano_lectivo` (String)
* `contas_correntes`:
  - `id` (PK, Int)
  - `aluno_id` (FK -> alunos_externo.id)
  - `saldo_devedor` (Decimal 12,2)
  - `atualizado_em` (DateTime)

---

## 3. Banco Interno PostgreSQL (`schema.prisma` via `PrismaInternalClient`)

Esquema autoritativo do PostgreSQL interno do sistema:

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
