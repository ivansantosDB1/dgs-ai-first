## Project Management Rules (Delivery Manager)

> Esta seção governa como agentes de IA geram tasks, issues, ADRs e documentos de
> gestão. Regras são prescritivas e obrigatórias. Em conflito entre regras, vale a
> mais restritiva. Termos: **DEVE** = obrigatório; **NÃO DEVE** = proibido;
> **QUANDO EM DÚVIDA** = comportamento default na ausência de instrução explícita.

### 1. Nomenclatura de tasks e issues

- Toda issue de implementação DEVE ter título no formato:
  `[<MODULO>] <verbo-imperativo> <objeto>` — ex: `[query-endpoint] Validar body com Zod`.
- `<MODULO>` DEVE ser um dos slugs válidos (kebab-case):
  `ingestao-documentos` · `query-endpoint` · `feedback-api` · `bot-teams` · `painel-web`.
- Toda task derivada de spec DEVE conter no corpo o campo `Requisito de origem: REQ-<MOD>-<NN>`.
- Tasks DEVEM usar IDs no formato `TASK-<MOD>-<NNN>` (ex: `TASK-QE-014`).
  Abreviações de módulo: ING, QE, FB, BOT, PNL.
- Toda issue DEVE ter exatamente uma label de tipo e uma de módulo:
  - Tipo (obrigatória, uma): `type:feature` · `type:bug` · `type:spec` · `type:adr` · `type:chore`.
  - Módulo (obrigatória, uma): `mod:ingestao` · `mod:query` · `mod:feedback` · `mod:bot` · `mod:painel`.
  - Gate (condicional): `gate:blocked` enquanto aguarda aprovação de um validation gate.
- NÃO DEVE criar issue sem label de tipo e de módulo.
- NÃO DEVE usar títulos genéricos (`fix`, `update`, `ajustes`, `WIP`).
- Branches DEVEM seguir `feat/<modulo>-<task-id>` (ex: `feat/query-endpoint-TASK-QE-014`).
- Commits DEVEM seguir Conventional Commits com o ID da task:
  `feat(query): TASK-QE-014 validar body com Zod`.

### 2. Documentação de decisões (ADR)

- Toda decisão técnica ou de escopo DEVE ser registrada como ADR em `/docs/adr/`.
- ADRs DEVEM seguir o nome `ADR-<NNNN>-<slug-kebab>.md` (numeração sequencial, 4 dígitos).
- Todo ADR DEVE conter as seções: `## Status` · `## Contexto` · `## Decisão` · `## Consequências`.
- `Status` DEVE ser um de: `Proposto` · `Aceito` · `Substituído por ADR-<NNNN>` · `Obsoleto`.
- ADRs NÃO DEVEM ser apagados. Decisão revertida DEVE virar `Status: Substituído por ADR-<NNNN>`
  (consistente com ADR-0003: documentos obsoletos são marcados, nunca excluídos).
- Todo `plan.md` que dependa de uma decisão DEVE linká-la: `Ref: docs/adr/ADR-<NNNN>-<slug>.md`.
- QUANDO EM DÚVIDA se algo é "decisão", trate como decisão e abra ADR — falso positivo é barato.

### 3. Validation gates (formato consumível por agente)

> Um artefato NÃO DEVE avançar de etapa sem o gate correspondente registrado.
> Cada gate é uma pré-condição verificável antes de gerar o artefato da etapa seguinte.

```yaml
gates:
  - id: G1
    transicao: requirements -> plan
    aprovador: Product Specialist
    pre_condicao: "requirements.md com status: Aprovada"
    bloqueia: "geracao de plan.md"
    label_enquanto_pendente: gate:blocked
  - id: G2
    transicao: tasks -> implement
    aprovador: Tech Lead
    pre_condicao: "tasks.md aprovado pelo Tech Lead"
    bloqueia: "abertura de branch feat/* e inicio de implementacao"
    label_enquanto_pendente: gate:blocked
  - id: G3
    transicao: code -> merge
    aprovador: Tech Lead
    pre_condicao: "PR com >=1 approval do Tech Lead E CI verde"
    bloqueia: "merge na main"
    label_enquanto_pendente: gate:blocked
  - id: G4
    transicao: tests -> deploy
    aprovador: QA (cobertura) + Tech Lead (autoriza deploy)
    pre_condicao: "suite cobre cenarios mapeados E QA registrou validacao"
    bloqueia: "execucao do pipeline de deploy"
    label_enquanto_pendente: gate:blocked
```

- Ao gerar um `plan.md`, o agente DEVE verificar que o `requirements.md` do mesmo módulo
  está `Aprovada` (G1). Se não estiver, NÃO DEVE gerar o plan e DEVE sinalizar `gate:blocked`.
- Ao gerar/abrir tasks para implementação, o agente DEVE confirmar G2 aprovado.
- Ao abrir PR, o agente DEVE referenciar a `TASK-<MOD>-<NNN>` e marcar que o merge depende de G3.
- NÃO DEVE marcar uma spec como `Validada` sem G4 registrado.

### 4. Restrições de comunicação (afetam geração de artefatos)

- Código, nomes de variáveis, comentários e mensagens de log DEVEM ser em **inglês**.
- Documentos de gestão, specs (`requirements.md`, `tasks.md`), ADRs e comunicados de status
  DEVEM ser em **português (pt-BR)**.
- Mensagens de commit DEVEM ser em **inglês** (Conventional Commits).
- Conteúdo voltado ao usuário final do assistente (respostas, mensagens do bot do Teams)
  DEVE ser em **português (pt-BR)**.
- Toda resposta gerada pelo assistente NovaTech DEVE incluir o campo `source_document`
  (rastreabilidade de fonte). Artefato de gestão que descreva a API DEVE refletir esse campo.
- NÃO DEVE inventar tiers de cliente: os únicos válidos são `Gold`, `Silver`, `Standard`.
- NÃO DEVE misturar versões de procedimento em um mesmo artefato sem marcar a vigência
  (ex: PROC-042 v1 vs v2) — consistente com o tratamento de contradições do projeto.

### 5. QUANDO EM DÚVIDA (defaults para agentes)

- Sem módulo claro para uma task → NÃO DEVE adivinhar; DEVE marcar `gate:blocked` e pedir
  classificação humana.
- Sem gate registrado para uma transição → DEVE tratar como bloqueado.
- Sem ADR para uma decisão que está sendo tomada no código → DEVE abrir ADR antes do merge.
