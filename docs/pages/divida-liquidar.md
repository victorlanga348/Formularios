# Especificação de Tela: Confinamento Financeiro (`/divida/liquidar`)

## 1. Visão Geral
Tela restritiva destinada a estudantes cujo token possui claim `SOMENTE_BOLETO`. Bloqueia o acesso ao fluxo de matrícula até a regularização financeira.

---

## 2. Elementos de Interface
- **Alerta de Confinamento:** Banner com ícone de aviso âmbar/vermelho indicando existência de débitos pendentes no banco legado.
- **Card de Resumo Financeiro:**
  - Código do Estudante.
  - Saldo Devedor Total formatado em moeda nacional.
  - Data da última sincronização com o banco financeiro legado.
- **Ações Disponíveis:**
  - Botão `"Descarregar Boleto de Liquidação"` (dispara download do PDF via `boletoLiquidacaoUrl`).
  - Botão `"Sair / Terminar Sessão"`.

---

## 3. Instruções ao Usuário
- Mensagem informativa explicando que, após o pagamento do boleto e compensação bancária, o estudante deve efetuar novo login para que o sistema efetive automaticamente qualquer matrícula pendente e libere o fluxo acadêmico.
