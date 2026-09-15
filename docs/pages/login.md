# Especificação de Tela: Login (`/login`)

## 1. Visão Geral
Tela inicial de autenticação onde o estudante insere suas credenciais institucionais para acesso ao portal acadêmico.

---

## 2. Elementos de Interface
- **Título / Cabeçalho:** Logotipo institucional e título "Portal de Matrícula Académica".
- **Formulário de Acesso:**
  - Campo `Código de Estudante` (Input numérico/alfanumérico, autofocus).
  - Campo `Senha` (Input password com alternador de visibilidade).
  - Botão de Ação: `"Entrar no Portal"` (com spinner em estado de loading).

---

## 3. Fluxos de Comportamento e Redirecionamento

```mermaid
graph TD
    A[Usuário submete credenciais] --> B[POST /auth/login]
    B -- "200 OK + LIVRE" --> C[Redirecionar para /matricula]
    B -- "200 OK + SOMENTE_BOLETO" --> D[Redirecionar para /divida/liquidar]
    B -- "403 Calouro" --> E[Exibir Banner de Bloqueio Manual]
    B -- "401 Credenciais Inválidas" --> F[Exibir Alerta Vermelho de Erro]
```

### 3.1 Tratamento de Erros Específicos
- **Calouro (HTTP 403):** Exibe banner em destaque: *"Matrícula de calouros é manual. Dirija-se à secretaria acadêmica."*
- **Credenciais Inválidas (HTTP 401):** *"Código de estudante ou senha incorretos."*
- **Serviço Indisponível (HTTP 503 / Timeout):** *"Serviço temporariamente indisponível. Tente novamente em instantes."*
