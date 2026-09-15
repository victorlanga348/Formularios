# AGENT OPERATING STANDARD & SPEC-DRIVEN DEVELOPMENT (SISTEMA DE MATRÍCULA E INSCRIÇÃO ACADÉMICA)

## 1. Fonte da Verdade
1. O diretório `/docs` e o [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md) são a fonte oficial da verdade sobre arquitetura, regras de negócio, design e contratos de dados.
2. Nunca assumir requisitos não documentados nem inventar regras de negócio ou payloads de API.
3. Se faltar informação oficial ou houver ambiguidade, marcar como `[PENDENTE DE VALIDAÇÃO]` e perguntar ao utilizador.
4. **Proibido alterar código** antes de apresentar o plano de implementação (com vantagens, desvantagens e riscos) e obter aprovação explícita.

---

## 2. Fluxo Obrigatório por Tarefa / Sprint em Equipa
Para qualquer alteração no projeto:
1. **Identificar as specs afetadas** em `/docs` e no [README.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/README.md).
2. **Definir a Trilha Responsável:**
   - **Trilha A:** Backend, PostgreSQL, Prisma Multi-Client, Auth e Boletos.
   - **Trilha B:** RulesEngineService, regras de negócio acadêmicas P0 a P5 e testes de borda.
   - **Trilha C:** Next.js App Router, Tailwind Dark Mode, Zustand e testes E2E.
3. **Criar o documento da tarefa** em `/docs/tasks/YYYY-MM-DD-nome-da-tarefa.md` utilizando o template em [docs/tasks/template.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/tasks/template.md).
4. **Apresentar o Plano**:
   - Contexto e Problema;
   - Solução proposta;
   - Análise de Trade-offs (Vantagens, Desvantagens, Riscos);
   - Critérios de Aceitação;
   - Checklist passo a passo.
5. **Aguardar Aprovação**: Parar e aguardar validação explícita do utilizador antes de tocar no código.
6. **Implementação Mínima**: Escrever o código estritamente necessário em formato cirúrgico (Diff/Snippet).
7. **Validação Técnica**: Executar typecheck, linter, testes e build via terminal com prefixo `rtk`.
8. **Sincronização de Docs**: Atualizar todas as specs afetadas conforme [docs/documentation-governance.md](file:///C:/Users/victo/OneDrive/Documentos/Github/Formularios/docs/documentation-governance.md).
9. **Relatório Final**: Finalizar respondendo obrigatoriamente:
   > *"Existe alguma alteração no projeto que não esteja refletida em /docs ou no README?"*
10. Depois de cada sprint realize testes nela e se tudo estiver a correr bem faça o commit (em português) dela antes de avançar para a próxima sprint.

---

## 3. Diretrizes de Qualidade, Código e Colaboração
- **Sem Placeholders**: Imagens, textos e links devem ser os reais definidos na documentação oficial.
- **Acessibilidade e Mobile-First**: Qualquer componente ou tela deve ser validado em Mobile, Tablet e Desktop com suporte a navegação por teclado e contraste adequado (Dark Mode operacional de alta legibilidade).
- **Escopo Restrito**: Proibido refatorar arquivos adjacentes ou adicionar dependências sem autorização explícita.
- **Execução de Comandos CLI**: Sempre utilizar o prefixo `rtk` para comandos no terminal (ex: `rtk npm run build`, `rtk git status`).
- **Padrão de Branches e Commits**: Branches nomeadas como `feat/sprint-<X>-trilha-<funcionalidade>` e commits em português no formato Conventional Commits.
