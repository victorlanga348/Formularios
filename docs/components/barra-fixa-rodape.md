# Componente: Barra Fixa Rodapé (Sticky Action Bar)

## 1. Responsabilidade
Componente persistente no rodapé da viewport para exibir totais de cadeiras selecionadas, somatório financeiro em tempo real e disparar a submissão transacional com chave de idempotência.

---

## 2. Contrato de Props e Estado
```typescript
export interface BarraFixaRodapeProps {
  quantidadeSelecionadas: number;
  valorTotal: number;
  isSimulando: boolean;
  isSubmetendo: boolean;
  onSubmeter: () => Promise<void>;
}
```

---

## 3. Comportamento e Detalhes de Implementação
- **Posicionamento:** `fixed bottom-0 left-0 right-0 z-50` com fundo `bg-slate-900/95` e `backdrop-blur-md`.
- **Feedback de Simulação:** Exibe skeleton ou spinner sutil ao lado do valor financeiro durante os 300ms de debounce da chamada `POST /matricula/simular`.
- **Submissão com Idempotência:**
  - Gera UUID v4 no cliente (`crypto.randomUUID()`).
  - Passa o header `Idempotency-Key` na requisição `POST /matricula/submeter`.
  - Desabilita o botão e exibe spinner de carregamento enquanto a transação é processada.
