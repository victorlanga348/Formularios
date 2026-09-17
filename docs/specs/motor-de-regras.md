# Especificação Técnica: Motor de Regras Acadêmicas (Zero-Trust)

## 1. Visão Geral do Módulo
O `RulesEngineService` é o componente central e determinístico responsável por garantir a integridade acadêmica do processo de inscrição. Ele opera sob premissa **Zero-Trust**: nenhuma informação enviada pelo cliente é aceita sem recálculo completo a partir do histórico acadêmico imutável.

---

## 2. Pipeline de Regras em Cascata

```
[P0: Trava Financeira] -> [P1: Validação Calouro] -> [P2: Paridade Estrita] 
                       -> [P3: Barreira de Ciclo] -> [P4: Próximo Passo Lógico] 
                       -> [P5: Precedências & Cascata]
```

---

## 3. Especificação Detalhada das Regras

### 3.1 Regra P2: Paridade Estrita
O portal acadêmico opera exclusivamente em semestres de paridade correspondente ao ciclo ativo institucional (Ciclo Ímpar = 1 ou Ciclo Par = 2):

$$\text{CadeirasElegiveis} = \{ c \in \text{Catalogo} \mid (c.\text{semestre} \pmod 2) \equiv (\text{CicloAtivo} \pmod 2) \}$$

- Cadeiras cujo semestre curricular não respeite a paridade ativa são purgadas na raiz da árvore de elegibilidade.

### 3.2 Regra P3: Barreira de Ciclo (Hard Stop)
Bloqueia verticalmente o avanço para ciclos avançados até que a base formativa esteja $100\%$ concluída:
- **Acesso ao 3º Ano:** O estudante deve possuir aprovação em $100\%$ das cadeiras do **1º Ano**. Se faltar uma única cadeira do 1º ano, nenhuma cadeira do 3º ano poderá ser ofertada ou selecionada.
- **Acesso ao 4º Ano:** O estudante deve possuir aprovação em $100\%$ das cadeiras do **2º Ano**. Se faltar uma única cadeira do 2º ano, nenhuma cadeira do 4º ano poderá ser ofertada ou selecionada.

### 3.3 Regra P4: Próximo Passo Lógico (PPL)
Impede saltos arbitrários de semestre curricular. O semestre teórico de destino ($S_{\text{logico}}$) é derivado sequencialmente:

```typescript
function calcularProximoSemestreLogico(aluno: HistoricoAluno): { ano: number; semestre: number } {
  const { ultimoAnoConcluido, ultimoSemestreConcluido } = aluno;
  if (ultimoSemestreConcluido === 1) {
    return { ano: ultimoAnoConcluido, semestre: 2 };
  }
  return { ano: ultimoAnoConcluido + 1, semestre: 1 };
}
```

- **Condição de Descasamento de Paridade:**
  Se $(S_{\text{logico}} \pmod 2) \neq (\text{CicloAtivo} \pmod 2)$, o **Grupo 1 (Regulares)** permanece vazio (`grupo1 = []`). O estudante fica restrito a cursar exclusivamente pendências do **Grupo 2 (Atrasos)** compatíveis com a paridade ativa.

### 3.4 Regra P5: Inscrição Híbrida e Desmarcação em Cascata

#### Grupo 1 (Regulares)
- São as cadeiras pertencentes a $S_{\text{logico}}$.
- **Comportamento Padrão:** Marcadas e desabilitadas para desmarcação (`checked = true, disabled = true`).
- **Bloqueio por Precedência:** Se uma cadeira do Grupo 1 possuir precedência não aprovada no histórico, ela é bloqueada para inscrição:
  `bloqueada = true, checked = false, motivo = "Falta precedência: [Nome da Cadeira]"`.

#### Grupo 2 (Atrasos Opcionais)
- Cadeiras reprovadas ou não cursadas de semestres anteriores que atendem à paridade do ciclo ativo e não violam a barreira de ciclo.
- **Comportamento Padrão:** Vêm pré-marcadas para incentivar regularização, mas habilitadas para edição (`checked = true, disabled = false`).
- **Injeção Automática por Precedência em Falta:**
  Quando uma cadeira do **Grupo 1** estiver bloqueada por falta de precedência, o motor verifica se a disciplina precedente pertence à paridade corrente do ciclo ativo. Caso pertença, a cadeira em falta é **injetada automaticamente no Grupo 2** marcada por padrão (`checked = true`), garantindo ao estudante a oportunidade de regularizar o pré-requisito no mesmo semestre acadêmico.

#### Desmarcação em Cascata Direta
Se o estudante desmarcar uma cadeira de atraso no Grupo 2 que seja pré-requisito de uma cadeira regular no Grupo 1, a cadeira dependente do Grupo 1 é compulsoriamente desmarcada e bloqueada:

$$\text{Desmarcar}(C_{\text{Atraso}} \in \text{Grupo 2}) \implies \forall C_{\text{Regular}} \in \text{Grupo 1} \mid C_{\text{Atraso}} \in \text{Precedencias}(C_{\text{Regular}}), \quad C_{\text{Regular}}.\text{checked} \leftarrow \text{false}$$

---

## 4. Auditoria e Validação Zero-Trust
No ato da submissão (`POST /matricula/submeter`):
1. O backend recebe apenas a lista de identificadores (`cadeirasIds: number[]`).
2. O `RulesEngineService` recalcula independentemente a lista de cadeiras válidas a partir do histórico do aluno.
3. Se qualquer ID enviado pelo cliente violar Paridade, Barreira de Ciclo, Precedências ou for inválido, a requisição é rejeitada com `HTTP 422 Unprocessable Entity` detalhando a infração.
