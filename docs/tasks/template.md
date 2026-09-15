# Tarefa: [YYYY-MM-DD] - [Nome Conciso da Tarefa]

## 1. Metadados e Atribuição
- **Data de Início:** AAAA-MM-DD
- **Sprint / Fase:** Sprint X (Fase Y)
- **Trilha / Frente:** [Trilha A (Backend/DB) | Trilha B (Motor/Regras) | Trilha C (Frontend/UI)]
- **Responsável Principal:** [Nome do Membro da Equipa]
- **Revisores de PR:** [Nome(s) dos Pares para Code Review]
- **Branch:** `feat/sprint-X-trilha-[nome-curto]`
- **Specs Afetadas:**
  - [ ] `README.md`
  - [ ] `docs/specs/auth-e-financeiro.md`
  - [ ] `docs/specs/motor-de-regras.md`
  - [ ] `docs/specs/api-matricula.md`
  - [ ] `docs/specs/ui-fluxo-estudante.md`

---

## 2. Contexto e Problema
*Descrição clara do problema ou necessidade técnica que motiva esta tarefa, apontando os módulos ou fluxos impactados.*

---

## 3. Solução Proposta
*Detalhamento cirúrgico da solução técnica a ser implementada, respeitando a arquitetura Zero-Trust, tipagem TypeScript e contratos estritos.*

---

## 4. Análise de Trade-offs
- **Vantagens:**
  - Vantagem técnica 1
  - Vantagem técnica 2
- **Desvantagens:**
  - Desvantagem ou limitação introduzida
- **Riscos e Mitigações:**
  - **Risco:** Descrição do risco operacional ou técnico.
  - **Mitigação:** Como o código ou teste previne o risco.

---

## 5. Critérios de Aceitação (Definition of Done)
- [ ] O comportamento implementado cumpre rigorosamente as regras descritas nas specs.
- [ ] Testes unitários / integrados escritos e validados via terminal com `rtk`.
- [ ] Schemas e DTOs tipados sem recurso a `any`.
- [ ] Acessibilidade e responsividade validadas (se envolver componentes de UI).
- [ ] Documentação correspondente em `/docs` atualizada.

---

## 6. Checklist de Implementação Passo a Passo
- [ ] **Etapa 1:** Preparação de contratos, interfaces e DTOs / testes preliminares (TDD).
- [ ] **Etapa 2:** Implementação mínima do componente / serviço / rota.
- [ ] **Etapa 3:** Integração e execução de testes automatizados (`rtk npm run test`).
- [ ] **Etapa 4:** Code review com o par da equipe e abertura de PR para `develop`.
