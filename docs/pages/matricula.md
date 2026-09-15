# Especificação de Tela: Matrícula e Inscrição Acadêmica (`/matricula`)

## 1. Visão Geral
Tela principal do fluxo de matrícula semestral para estudantes com `claimAcesso: "LIVRE"`. Permite a visualização das cadeiras regulares obrigatórias e a inclusão/remoção opcional de cadeiras em atraso.

---

## 2. Estrutura da Página

1. **Cabeçalho de Identificação:**
   - Nome e Código do Estudante.
   - Ciclo Ativo Institucional (Ex: *1º Semestre / Ciclo Ímpar*).
   - Próximo Passo Lógico derivado ($S_{\text{logico}}$, ex: *2º Ano / 1º Semestre*).

2. **Seção 1: Cadeiras Regulares (Grupo 1):**
   - Incorpora o componente `TabelaGrupo1`.
   - Exibe as disciplinas do semestre lógico corrente com status de elegibilidade e precedências.

3. **Seção 2: Cadeiras em Atraso (Grupo 2):**
   - Incorpora o componente `TabelaGrupo2`.
   - Exibe as disciplinas pendentes de semestres anteriores com paridade compatível.

4. **Rodapé Fixo Persistente:**
   - Incorpora o componente `BarraFixaRodape`.
   - Monitora em tempo real o número de cadeiras e valor monetário total com recálculo determinístico via backend (`POST /matricula/simular`).
