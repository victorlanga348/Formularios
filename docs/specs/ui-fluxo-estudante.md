# Especificação Técnica: Interface Web e Fluxo do Estudante (Next.js)

## 1. Visão Geral da Interface
A aplicação frontend é construída em **Next.js 14+ (App Router)** com **Tailwind CSS**, operando sob estética **Dark Mode de alta legibilidade operacional**, responsividade estrita (Mobile, Tablet, Desktop) e conformidade de acessibilidade **WCAG 2.1 AA**.

---

## 2. Estrutura de Rotas e Proteção por Middleware

### 2.1 Mapeamento de Rotas
- `/login`: Autenticação do estudante (Código de Estudante + Senha).
- `/divida/liquidar`: Tela de confinamento para estudantes com `claimAcesso: "SOMENTE_BOLETO"`. Exibe valor da dívida pendente e botão de download do boleto de regularização.
- `/matricula`: Tela principal de seleção acadêmica (Grupo 1 e Grupo 2).
- `/matricula/confirmacao`: Tela pós-submissão com detalhes da inscrição pendente e visualizador/download do boleto gerado.

### 2.2 Proteção via `middleware.ts`
```typescript
// Intercepta todas as requisições para /matricula/*
if (token.claimAcesso === "SOMENTE_BOLETO" && pathname.startsWith("/matricula")) {
  return NextResponse.redirect(new URL("/divida/liquidar", request.url));
}

if (token.claimAcesso === "LIVRE" && pathname.startsWith("/divida/liquidar")) {
  return NextResponse.redirect(new URL("/matricula", request.url));
}
```

---

## 3. Estado Global do Cliente (Zustand)

A store Zustand (`useMatriculaStore`) orquestra o estado das seleções de cadeiras, a desmarcação em cascata e o disparo de simulação:

```typescript
interface MatriculaState {
  grupo1: CadeiraRegular[];
  grupo2: CadeiraAtraso[];
  selecionadasIds: number[];
  valorTotal: number;
  isSimulando: boolean;
  
  toggleCadeiraAtraso: (cadeiraId: number) => void;
  carregarOpcoes: (dados: OpcoesMatriculaDTO) => void;
  simularTaxas: () => Promise<void>;
}
```

### 3.1 Reatividade da Desmarcação em Cascata
Quando `toggleCadeiraAtraso(id)` desmarca uma cadeira em atraso:
1. O ID é removido de `selecionadasIds`.
2. Para cada cadeira regular em `grupo1`:
   - Se a cadeira de atraso constar em suas `precedencias`, a cadeira regular é automaticamente marcada como `checked = false` e seu estado passa a exibir badge de bloqueio por falta de precedência.
3. É acionado um `debounce` (300ms) para chamar `POST /matricula/simular` e atualizar o `valorTotal`.

---

## 4. Componentes e Telas

### 4.1 Tabela Grupo 1 (Regulares)
- Exibe as cadeiras do Próximo Passo Lógico ($S_{\text{logico}}$).
- **Cadeira Elegível:** Checkbox pré-marcado e desabilitado (`checked = true, disabled = true`), indicando que a matrícula em disciplinas do semestre regular é mandatória.
- **Cadeira com Precedência Ausente:** Checkbox desmarcado e travado (`checked = false, disabled = true`), com ícone de cadeado âmbar/vermelho, badge visual "Precedência Pendente" e tooltip com o nome da cadeira pré-requisito.

### 4.2 Tabela Grupo 2 (Atrasos Opcionais)
- Exibe cadeiras reprovadas/pendentes da mesma paridade.
- Checkbox interativo (`checked = true, disabled = false`), permitindo ao aluno optar por cursar ou adiar a regularização.

### 4.3 Barra de Ação Fixa (Rodapé)
- Componente persistente no rodapé da viewport.
- Exibe:
  - Quantidade total de cadeiras selecionadas.
  - Somatório monetário em tempo real (`valorTotal`) formatado em moeda nacional.
  - Indicador de carregamento tipo skeleton/spinner discreto durante chamadas de simulação.
  - Botão `"Confirmar Matrícula"`. Ao clicar:
    - Gera UUID v4 no cliente (`crypto.randomUUID()`).
    - Envia requisição com o cabeçalho `Idempotency-Key`.
    - Redireciona para `/matricula/confirmacao`.

---

## 5. Diretrizes de Acessibilidade e Design System
- **Dark Mode Operacional:** Fundo neutro escuro (`#0f172a` / `#1e293b`), texto de alto contraste (`#f8fafc` / `#cbd5e1`), respeitando contraste mínimo de 4.5:1 (WCAG AA).
- **Foco e Teclado:** Elementos interativos com outline visível (`ring-2 ring-blue-500`). Navegação por `Tab`, `Shift+Tab`, `Space` e `Enter` em todos os checkboxes e botões.
- **Microcopy Precisa:** Mensagens de erro e avisos diretos e objetivos, sem ambiguidades técnicas para o usuário final.
