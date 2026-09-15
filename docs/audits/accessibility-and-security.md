# Auditorias: Acessibilidade (WCAG AA) e Segurança (Zero-Trust)

## 1. Diretrizes de Acessibilidade (WCAG 2.1 AA)

### 1.1 Navegação por Teclado
- **Foco Visível:** Todos os elementos interativos (botões, links, inputs e checkboxes) possuem outline visível `ring-2 ring-blue-500`.
- **Ordem de Tabulação Lógica:** Sequência natural de tabulação:
  1. Cabeçalho de perfil
  2. Tabela Grupo 1 (disciplinas regulares)
  3. Tabela Grupo 2 (disciplinas em atraso)
  4. Barra fixa no rodapé (botão de confirmação).

### 1.2 Semântica e Atributos ARIA
- Checkboxes desabilitados por regra ou bloqueados por precedência utilizam `aria-disabled="true"` e `disabled`.
- Badges de erro e bloqueio associados via `aria-describedby` ao input correspondente.
- Banners de alerta e notificações com `role="alert"` e `aria-live="polite"`.

### 1.3 Contraste de Cores
- Texto primário (`#f8fafc`) sobre fundo escuro (`#0f172a` / `#1e293b`) garante taxa de contraste superior a **12:1** (exigência mínima WCAG AA: 4.5:1).
- Badges de perigo (`#ef4444`) e aviso (`#f59e0b`) utilizam tipografia de alto contraste dedicada.

---

## 2. Auditoria de Segurança & Premissas Zero-Trust

### 2.1 Validação Servidor Independente
- O payload do cliente (`cadeirasIds: number[]`) é tratado como não confiável.
- O `RulesEngineService` recalcula Paridade, Barreira de Ciclo, PPL e Precedências consultando unicamente o histórico acadêmico persistido no banco de dados.

### 2.2 Controle de Concorrência e Idempotência
- Toda submissão transacional exige cabeçalho `Idempotency-Key` (UUID v4).
- A gravação ocorre sob transação atômica ACID no PostgreSQL (`prismaInternal.$transaction`), impedindo condições de corrida e duplicidade de inscrições.

### 2.3 Resiliência e Fail-Closed
- Consultas a sistemas legados utilizam Circuit Breaker (`Opossum`).
- Em falha contínua ou timeout (3000ms), o sistema adota política conservadora **fail-closed**, confinando o acesso a `SOMENTE_BOLETO` para prevenir evasão financeira indevida.
