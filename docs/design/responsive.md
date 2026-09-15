# Responsividade e Comportamento por Breakpoints

## 1. Breakpoints Adotados
O sistema adota os breakpoints do Tailwind CSS, com validação obrigatória nas três faixas de dispositivos:

| Dispositivo | Breakpoint | Largura | Comportamento Principal |
| :--- | :--- | :--- | :--- |
| **Mobile** | `< 640px` (Default) | 320px - 639px | Layout em coluna única, tabelas com scroll horizontal suave ou exibição em cartões, barra de ação fixa adaptada. |
| **Tablet** | `md:` | 768px - 1023px | Tabelas com colunas compactas, cartões de resumo lado a lado. |
| **Desktop** | `lg:` e `xl:` | 1024px+ | Tabelas completas com espaçamento confortável, barra de rodapé com métricas expandidas. |

---

## 2. Padrões Específicos por Componente

### 2.1 Tabelas de Cadeiras (Grupo 1 e Grupo 2)
- **Desktop:** Tabela tradicional com colunas `[Checkbox, Código, Nome da Cadeira, Ano/Semestre, Precedências, Taxa, Status]`.
- **Mobile:** Container com `overflow-x-auto` e sombra sutil à direita para indicar rolagem horizontal, garantindo que o checkbox e o nome da cadeira nunca sejam truncados.

### 2.2 Barra de Ação Fixa (Rodapé)
- Componente `fixed bottom-0 left-0 right-0 z-50` com backdrop-blur (`bg-slate-900/90 backdrop-blur-md`).
- **Mobile:** Total monetário e botão de confirmação empilhados ou dispostos em linha compacta com `safe-area-inset-bottom`.
- **Desktop:** Layout flex horizontal espaçado (`justify-between`), com métricas à esquerda e CTA principal à direita.
