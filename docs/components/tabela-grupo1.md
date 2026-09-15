# Componente: Tabela Grupo 1 (Regulares)

## 1. Responsabilidade
Exibir as disciplinas regulares obrigatórias do Próximo Passo Lógico ($S_{\text{logico}}$), sinalizando visualmente bloqueios por precedências não atendidas.

---

## 2. Contrato de Props e Estado
```typescript
export interface CadeiraRegular {
  id: number;
  codigo: string;
  nome: string;
  anoCurricular: number;
  semestreCurricular: number;
  taxa: number;
  checked: boolean;
  disabled: boolean;
  bloqueada: boolean;
  motivoBloqueio: string | null;
  precedencias: number[];
}

export interface TabelaGrupo1Props {
  cadeiras: CadeiraRegular[];
}
```

---

## 3. Comportamento e Regras Visuais
- **Cadeira Elegível:** Checkbox marcado e desabilitado (`checked = true, disabled = true`), indicando que a inscrição regular é obrigatória.
- **Cadeira com Precedência Pendente:** Checkbox desmarcado e desabilitado (`checked = false, disabled = true`), exibindo badge vermelho/âmbar *"Precedência Pendente"* e tooltip com a lista de cadeiras faltantes.
- **Linha de Tabela:** Destaque em `bg-surface`, bordas em `border-subtle`, tipografia em `text-primary`.
