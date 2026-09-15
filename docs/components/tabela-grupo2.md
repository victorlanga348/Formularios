# Componente: Tabela Grupo 2 (Atrasos Opcionais)

## 1. Responsabilidade
Exibir as disciplinas pendentes de semestres anteriores compatíveis com a paridade ativa, permitindo a seleção/desmarcação interativa do estudante e acionando o efeito cascata.

---

## 2. Contrato de Props e Estado
```typescript
export interface CadeiraAtraso {
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

export interface TabelaGrupo2Props {
  cadeiras: CadeiraAtraso[];
  onToggleCadeira: (cadeiraId: number) => void;
}
```

---

## 3. Comportamento e Interação
- **Checkbox Interativo:** Vem pré-marcado por padrão para incentivar a regularização acadêmica, mas livre para alteração (`checked = true, disabled = false`).
- **Desmarcação em Cascata:** Ao disparar `onToggleCadeira(id)` com desmarcação:
  - Se esta cadeira for precedência de alguma disciplina regular do Grupo 1, a disciplina do Grupo 1 é desmarcada e bloqueada imediatamente no estado global do Zustand (`useMatriculaStore`).
- **Feedback Visual:** Badge informativo âmbar *"Atraso"* em cada linha.
