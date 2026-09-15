# Especificação de Tela: Confirmação de Matrícula (`/matricula/confirmacao`)

## 1. Visão Geral
Tela de sucesso exibida imediatamente após a submissão transacional de matrícula (`POST /matricula/submeter`).

---

## 2. Elementos de Interface
- **Ícone de Sucesso:** Checkmark verde (`status-success`) com título "Inscrição Acadêmica Registada com Sucesso".
- **Card de Resumo da Inscrição:**
  - Identificador da Matrícula (`matriculaId` UUID).
  - Status Atual: `PENDENTE (Aguardando Liquidação de Boleto)`.
  - Valor Total das Taxas.
  - Data Limite de Expiração do Boleto (`expiraEm`).
- **Seção do Boleto Bancário:**
  - Campo com Código de Barras e botão `"Copiar Código"`.
  - Botão principal `"Descarregar Boleto em PDF"`.
- **Ações de Navegação:**
  - Botão secundário `"Voltar ao Início / Terminar Sessão"`.
