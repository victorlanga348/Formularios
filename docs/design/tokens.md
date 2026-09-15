# Design Tokens & Sistema Visual (UniTiva / Unimestre)

## 1. Princípios de Design & Identidade
O design da interface integra a identidade visual institucional da **Universidade Wutivi (UniTiva)** e do portal **Unimestre**, combinando estética moderna, alto contraste operacional (**WCAG 2.1 AA/AAA**) e suporte a temas Dark e Light.

---

## 2. Paleta Cromática Institucional

| Papel | Token | Hex | RGB | HSL | Aplicação Principal |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Primary Brand** | `--color-brand-primary` | `#5B2BE0` | `91, 43, 224` | `256°, 75%, 52%` | Sidebar, botões primários, cabeçalhos e elementos de destaque |
| **Primary Hover** | `--color-brand-primary-hover` | `#481EB8` | `72, 30, 184` | `256°, 72%, 42%` | Estado ativo/hover de botões e links de navegação |
| **Primary Light** | `--color-brand-primary-light` | `#EDE9FE` | `237, 233, 254` | `250°, 91%, 95%` | Fundo de badges sutis e seleções ativas no Light Mode |
| **Secondary Accent** | `--color-brand-accent` | `#8CB82B` | `140, 184, 43` | `79°, 62%, 45%` | Elementos de apoio, detalhes geométricos e badges de atraso |
| **Institutional Navy** | `--color-brand-navy` | `#0D2B6B` | `13, 43, 107` | `221°, 78%, 24%` | Brasão UniTiva, bordas estruturais e cartões institucionais |
| **Institutional Crimson**| `--color-brand-crimson` | `#C1121F` | `193, 18, 31` | `356°, 83%, 41%` | Detalhes do logo, avisos de bloqueio manual e alertas críticos |

---

## 3. Tokens Semânticos por Tema

### 3.1 Dark Mode Operacional (Padrão)
Projetado para reduzir fadiga visual e proporcionar foco contínuo no fluxo de seleção acadêmica:

```css
.dark {
  --bg-app: #0c0a17;              /* Fundo profundo com matiz violeta escuro */
  --bg-surface: #17132a;          /* Superfície de tabelas, painéis e cartões */
  --bg-surface-elevated: #211c3d; /* Modais, dropdowns e tooltips flutuantes */
  --border-subtle: #312a56;       /* Bordas de inputs e separadores */
  --border-highlight: #5b2be0;    /* Bordas de cartões com seleção ativa */
  
  --text-primary: #f8fafc;        /* Títulos, valores de taxas e textos de destaque */
  --text-secondary: #c4b5fd;      /* Rótulos e descrições com tonalidade lavanda suave */
  --text-muted: #94a3b8;          /* Metadados, códigos de cadeira e dicas secundárias */

  --status-success: #10b981;      /* Confirmação e validação regularizada */
  --status-warning: #f59e0b;      /* Advertências e pendências não impeditivas */
  --status-danger: #ef4444;       /* Bloqueios de precedência e erros */
  --status-info: #6366f1;         /* Informativos de ciclo acadêmico */
}
```

### 3.2 Light Mode Institucional (Portal Unimestre)
Espelha o layout tradicional do portal de gestão acadêmica:

```css
.light {
  --bg-app: #f4f5f7;              /* Fundo neutro do portal institucional */
  --bg-surface: #ffffff;          /* Cartões e tabelas brancas */
  --bg-surface-elevated: #ffffff; /* Modais e popovers */
  --border-subtle: #e2e8f0;       /* Separadores e bordas de inputs */
  --border-highlight: #5b2be0;    /* Destaque ativo */

  --text-primary: #0f172a;        /* Texto principal */
  --text-secondary: #475569;      /* Texto de apoio */
  --text-muted: #64748b;          /* Metadados e códigos */
}
```

---

## 4. Tipografia & Escala Visual

- **Fonte Principal:** `Inter`, `system-ui`, `-apple-system`, `sans-serif`
- **Escala Modular:**
  - `text-xs` (12px / 16px): Badges de atraso, precedências e tags de status.
  - `text-sm` (14px / 20px): Linhas de tabela, microcopy e textos auxiliares.
  - `text-base` (16px / 24px): Corpo padrão e campos de formulário.
  - `text-lg` (18px / 28px): Subtítulos de tabela e somatórios no rodapé.
  - `text-xl` (20px / 28px): Títulos de seções (Grupo 1 / Grupo 2).
  - `text-2xl` (24px / 32px): Cabeçalhos principais de página e login.

---

## 5. Estados Interativos e Acessibilidade (WCAG 2.1 AA)

- **Foco por Teclado:** `focus-visible:ring-2 focus-visible:ring-[#5b2be0] focus-visible:ring-offset-2 focus-visible:outline-none`
- **Contraste de Acessibilidade:**
  - Texto Branco (`#FFFFFF`) sobre Roxo UniTiva (`#5B2BE0`): **7.84:1** (Nível AAA)
  - Texto Primário (`#F8FAFC`) sobre Fundo Dark (`#0C0A17`): **18.2:1** (Nível AAA)
  - Accent Lime (`#8CB82B`) sobre Fundo Dark (`#0C0A17`): **8.42:1** (Nível AAA)
- **Checkboxes:**
  - **Regular Obrigatório:** Marcado e travado (`bg-[#5b2be0] opacity-75 cursor-not-allowed`)
  - **Atraso Interativo:** Selecionável (`accent-[#5b2be0] cursor-pointer`)
  - **Bloqueado por Precedência:** Desmarcado e desabilitado com badge vermelho (`#ef4444`)
