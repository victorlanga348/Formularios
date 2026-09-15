# Microcopy & Mensagens do Sistema

## 1. Diretrizes de Tom de Voz
- **Objetividade & Rigor:** Linguagem formal institucional, clara e sem ambiguidades.
- **Foco na Resolução:** Toda mensagem de erro deve indicar exatamente a causa e a ação esperada do estudante.

---

## 2. Catálogo Oficial de Mensagens

### 2.1 Autenticação e Calouros
- **Calouro (HTTP 403):**  
  > *"Matrícula de calouros é manual. Dirija-se à secretaria acadêmica."*
- **Credenciais Inválidas (HTTP 401):**  
  > *"Código de estudante ou senha incorretos."*
- **Sessão Expirada:**  
  > *"A sua sessão expirou por inatividade. Efetue login novamente."*

### 2.2 Confinamento e Financeiro
- **Alerta de Débito:**  
  > *"Acesso suspenso para matrícula online devido a valores pendentes no sistema de contas correntes. Efetue a liquidação do boleto abaixo para regularizar o seu acesso."*
- **Instrução Pós-Pagamento:**  
  > *"Após a confirmação bancária do pagamento, inicie sessão novamente no portal para efetivar a sua inscrição semestral."*

### 2.3 Regras Acadêmicas e Validações
- **Bloqueio por Falta de Precedência:**  
  > *"Bloqueada: Requer aprovação prévia na disciplina {nomeCadeiraPrecedente}."*
- **Desmarcação em Cascata:**  
  > *"Atenção: Ao remover a cadeira de atraso {nomeCadeira}, a disciplina regular dependente {nomeCadeiraDependente} foi desmarcada automaticamente."*
- **Barreira de Ciclo Ativa:**  
  > *"Não é possível cursar disciplinas do {anoAlvo}º Ano enquanto houver pendências não concluídas no {anoBase}º Ano."*
- **Semestre sem Oferta Regular (Descasamento de Paridade):**  
  > *"O seu próximo semestre lógico não coincide com a paridade do ciclo atual. Apenas disciplinas em atraso compatíveis estão disponíveis para seleção."*

### 2.4 Confirmação e Submissão
- **Sucesso na Submissão:**  
  > *"Inscrição submetida com sucesso. O seu comprovativo e boleto bancário foram gerados."*
- **Aviso de Idempotência:**  
  > *"Submissão já processada anteriormente. Exibindo os dados da inscrição confirmada."*
