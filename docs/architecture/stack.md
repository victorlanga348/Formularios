# Arquitetura e Stack Tecnológica

## 1. Visão Geral da Arquitetura
O **Sistema de Matrícula e Inscrição Académica** adota uma arquitetura desacoplada e resiliente com dualidade de bancos de dados, motor de validação pura Zero-Trust e interface web Server/Client de alta performance.

```
+-------------------------------------------------------------------------+
|                        Frontend (Next.js 14+)                           |
|       App Router | Tailwind Dark Mode | Zustand | Middleware Auth       |
+-------------------------------------------------------------------------+
                                    |
                           REST API / JWT Bearer
                                    v
+-------------------------------------------------------------------------+
|                         Backend (NestJS API)                            |
|  +-------------------------------------------------------------------+  |
|  | AuthModule (JWT + Dynamic Claims: SOMENTE_BOLETO / LIVRE)         |  |
|  +-------------------------------------------------------------------+  |
|  | FinanceiroModule + Opossum Circuit Breaker (Fail-Closed)          |  |
|  +-------------------------------------------------------------------+  |
|  | RulesEngineModule (Validação Pura Zero-Trust P0 -> P5)             |  |
|  +-------------------------------------------------------------------+  |
|  | MatriculaModule (Transações ACID PostgreSQL + Idempotency)        |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
           |                                              |
   PrismaExternal (Read-Only)                     PrismaInternal (ACID)
           v                                              v
+-----------------------+                      +--------------------------+
| Banco Externo Legado  |                      | Banco Interno PostgreSQL |
| Alunos / Histórico /  |                      | Sessões / Caches /       |
| Contas Correntes      |                      | Inscrições / Boletos Log |
+-----------------------+                      +--------------------------+
```

---

## 2. Stack Tecnológica Oficial

| Camada | Tecnologia | Justificativa Técnica |
| :--- | :--- | :--- |
| **Backend Framework** | **NestJS (Node.js + TypeScript)** | Arquitetura modular corporativa orientada a injeção de dependências, facilitando testes unitários isolados do motor de regras. |
| **Frontend Framework** | **Next.js 14+ (App Router)** | Server Components com streaming, Route Handlers velozes e tipagem TypeScript ponta a ponta. |
| **Estilização / UI** | **Tailwind CSS** | Design system consistente em Dark Mode operacional de alta legibilidade e conformidade WCAG AA. |
| **Gerenciamento de Estado** | **Zustand** | Store leve e determinística (`useMatriculaStore`) para controle de seleção, cascata e debounce de simulação. |
| **Banco de Dados Principal** | **PostgreSQL** | Integridade referencial forte, controle estrito de concorrência e suporte a transações ACID. |
| **Acesso a Dados (ORM)** | **Prisma Multi-Client** | Clientes tipados isolados: `PrismaInternalClient` (PostgreSQL) e `PrismaExternalClient` (Legado Read-Only). |
| **Resiliência Externa** | **Circuit Breaker (Opossum)** | Monitoramento ativo da API/DB externa com política fail-closed e fallback para cache local. |
| **Geração de PDF** | **Serviço NestJS (`pdfmake` / `puppeteer-core`)** | Geração e streaming assíncrono de boletos bancários com persistência em `BoletoLog`. |
| **Autenticação & Sessão** | **NestJS Passport + JWT** | Claims dinâmicos recalculados no ciclo de login com guardas `JwtAuthGuard` customizadas. |

---

## 3. Padrões de Design e Segurança
1. **Zero-Trust Input:** O servidor nunca aceita o estado do cliente; recalcula toda a elegibilidade acadêmica a partir do histórico bruto.
2. **Idempotência Transacional:** Toda submissão de matrícula exige `Idempotency-Key` (UUID v4) no header.
3. **Fail-Closed Financeiro:** Se o serviço financeiro externo estiver inacessível e não houver histórico seguro, o acesso é retido em `SOMENTE_BOLETO`.
