# Exercício 2.1 — Workflow de Desenvolvimento AI First

> **Trilha de Certificação AI First — DGS / DB1 Global Software** · Cenário 2 (Estruturação do Trabalho)
> **Papel:** Delivery Manager · **Repositório:** `db1/novatech-assistant`
> **Time:** 1 Tech Lead · 2 Desenvolvedores (1 pleno, 1 sênior) · 1 QA · 1 Product Specialist · 1 Delivery Manager
> **Ferramentas:** Claude (chat) — elaboração do fluxo · Claude Cowork — checklist de validation gates
> **Entregável único cobrindo as 3 tarefas:** (1) fluxo de trabalho · (2) checklist de gates · (3) evidência de uso das ferramentas.

---

## Índice

1. [Tarefa 1 — Fluxo de trabalho AI First](#tarefa-1)
2. [Tarefa 2 — Template de checklist de validation gates (Cowork)](#tarefa-2)
3. [Tarefa 3 — Evidência de uso das ferramentas](#tarefa-3)

---

<a name="tarefa-1"></a>
# Tarefa 1 — Fluxo de trabalho AI First

## 1.1 Princípio do fluxo

A IA **gera**; o humano **valida**. Cada etapa do ciclo tem um responsável humano (accountable) que opera uma ferramenta de IA específica do seu papel, e cada transição de risco é protegida por um *validation gate* (Tarefa 2). O fluxo herda as decisões do Cenário 1: stack TypeScript/React/Bicep, pipeline Azure AI Search + Azure OpenAI, context budget da ADR-0002 e tratamento de contradições da ADR-0003.

**Atribuição de ferramentas por papel** (não se misturam):

| Papel | Ferramenta de IA primária | Por quê |
|-------|---------------------------|---------|
| Product Specialist | Claude Cowork + Claude Design | Redige requirements e mockups; não escreve código |
| Tech Lead | GitHub Copilot + Claude (chat) | Desenha o `plan.md` e revisa arquitetura; opera Copilot em código |
| Desenvolvedor (pleno/sênior) | GitHub Copilot | Decompõe tasks e implementa em TypeScript |
| QA | Claude Cowork + Claude (chat) | Gera specs de teste e critérios; valida cobertura |
| Delivery Manager | Claude Cowork | Orquestra gates, board de specs e tracking — não gera código nem spec de produto |

## 1.2 Matriz Papel × Etapa do Ciclo

Legenda: **R** = Responsável (executa) · **A** = Aprova (gate) · **C** = Consultado · **—** = não atua.
A ferramenta de IA aparece entre parênteses quando o papel atua na etapa.

| Papel \ Etapa | **Spec** | **Plan** | **Tasks** | **Implement** | **Review** | **Deploy** |
|---------------|----------|----------|-----------|---------------|------------|------------|
| **Product Specialist** | **R** (Cowork + Design) | C | C | — | C (aderência ao requirements) | — |
| **Tech Lead** | A (aprova spec) | **R** (Copilot + Claude) | A (valida tasks) | C | **R/A** (code review, Copilot) | A (autoriza deploy) |
| **Dev Sênior** | C | C | **R** (Copilot) | **R** (Copilot) | C (peer review) | C |
| **Dev Pleno** | — | C | **R** (Copilot) | **R** (Copilot) | C (peer review) | — |
| **QA** | C (testabilidade) | C | C | C | **R** (gera testes, Cowork) | A (valida suite) |
| **Delivery Manager** | C (governa board) | C | C | — | C | **R** (orquestra gate, Cowork) |

### Leitura por etapa

1. **Spec** — Product Specialist redige `requirements.md` no Cowork (mockup no Design). QA revisa testabilidade; Tech Lead aprova. → **Gate 1**
2. **Plan** — Tech Lead transforma a spec aprovada em `plan.md` (arquitetura, contratos, stack) com Copilot + Claude.
3. **Tasks** — Devs decompõem o plan em `tasks.md` (unidades atômicas) com Copilot. Tech Lead valida. → **Gate 2**
4. **Implement** — Devs implementam em TypeScript com Copilot, seguindo o plan e os paths do repositório.
5. **Review** — Tech Lead faz code review do código gerado por agente; QA gera e roda a suite (Cowork). → **Gate 3** (merge) e **Gate 4** (testes→deploy)
6. **Deploy** — Delivery Manager orquestra o gate final; Tech Lead autoriza; CI/CD (GitHub) publica via Bicep.

## 1.3 Diagrama do fluxo

Ver arquivo `fluxo-ai-first.svg` (anexo). Representação textual:

```
  [SPEC]          [PLAN]         [TASKS]        [IMPLEMENT]      [REVIEW]        [DEPLOY]
   PS               TL             Devs            Devs           TL + QA          DM/TL
 Cowork+         Copilot+        Copilot         Copilot       Copilot+Cowork   Cowork+CI/CD
  Design          Claude
    │               │               │               │               │              │
    └──[GATE 1]─────┘               └──[GATE 2]──────┘               ├──[GATE 3]────┤
       Spec→Plan                       Tasks→Implement                │  merge       │
       aprova: TL                      aprova: TL                     └──[GATE 4]────┘
                                                                         testes→deploy
                                                                         aprova: QA
```

## 1.4 Equilíbrio velocidade × segurança

A IA acelera as quatro etapas de maior esforço (Spec, Plan, Tasks, Implement) gerando rascunhos que o humano refina. A segurança vem de concentrar a validação humana obrigatória só nas transições de risco real — entrada de requisito, entrada de implementação, entrada na `main`, entrada em produção — deixando as transições reversíveis (Plan→Tasks) fluírem sem cerimônia. Nenhum gate é "revisar antes de continuar" genérico, e nenhum ponto de risco fica sem dono.

---

<a name="tarefa-2"></a>
# Tarefa 2 — Template de checklist de validation gates (Cowork)

> **Natureza:** Template **reutilizável**. Copie esta seção para cada módulo/spec (`gates-<modulo>.md`) e preencha os campos `[ ]`. Não é um registro único.

## Como usar

1. Ao iniciar um módulo (ex: *Query Endpoint*), duplique este checklist e renomeie.
2. Um gate só é "passado" quando todos os itens `[ ]` estiverem `[x]` e o aprovador registrar nome + data.
3. Prazos contam em **horas úteis** (relógio pausa fora de 08h–18h em dias úteis, conforme lógica de SLA da NovaTech).
4. Reprovação nunca é "volta tudo": cada gate define a ação concreta de retorno.

**Identificação do item avaliado**

| Campo | Valor |
|-------|-------|
| Módulo / Spec | `[ex: API de busca — query endpoint]` |
| Branch / PR | `[ex: feat/query-endpoint #00]` |
| Data de entrada no fluxo | `[dd/mm/aaaa]` |

### GATE 1 — Spec → Plan

**Pergunta-chave:** a spec está madura o suficiente para o Tech Lead construir o plano em cima dela?

- **Quem aprova:** Tech Lead (accountable). Consultado: QA (testabilidade).
- **Prazo (SLA do gate):** até **8h úteis** após a spec entrar em "Em Revisão".
- **Se reprovar:** spec volta ao Product Specialist com comentários inline; não gera `plan.md` até reentrar aprovada.

**Verificar:**
- [ ] Cada requisito tem ao menos um **critério de aceite verificável** (QA escreve teste binário).
- [ ] A spec referencia as **decisões do Cenário 1** aplicáveis (ADRs, context budget, contradições).
- [ ] Termos conforme a **linguagem ubíqua** ("carga perigosa = classes 1–6 ANTT"; "frete especial = acima de 500kg"; tiers Gold/Silver/Standard).
- [ ] Escopo e fora-de-escopo declarados.
- [ ] Sem requisito ambíguo ("responder bem" / "ser rápido" sem número).

**Aprovação:** `[ ]` Aprovado por: __________ Data: ______ | `[ ]` Reprovado — motivo: __________

### GATE 2 — Tasks (geradas por IA) → Implement

**Pergunta-chave:** as tasks decompostas pelo Copilot fazem sentido e são implementáveis isoladamente?

- **Quem aprova:** Tech Lead (accountable). Consultado: Dev que vai implementar.
- **Prazo (SLA do gate):** até **4h úteis** após `tasks.md` ser gerado.
- **Se reprovar:** tasks voltam ao Dev para re-decomposição; Tech Lead aponta quais quebrar/fundir.

**Verificar:**
- [ ] Cada task é **atômica** — implementável e testável de forma independente.
- [ ] Cada task tem **ID, dependências e critério de aceite** próprios.
- [ ] Nenhuma task "fantasma" sem origem no `plan.md` (rastreabilidade plan → task).
- [ ] Tasks respeitam a stack (TypeScript, Zod, Azure Functions v4) — sem tecnologia fora do plan.
- [ ] Estimativa plausível (nenhuma task gigante disfarçada de atômica).

**Aprovação:** `[ ]` Aprovado por: __________ Data: ______ | `[ ]` Reprovado — motivo: __________

### GATE 3 — Código (gerado por agente) → Merge na `main`

**Pergunta-chave:** o código gerado pelo Copilot é seguro, correto e mantível para entrar na `main`? *(Gate mais rígido — irreversível na linha principal.)*

- **Quem aprova:** Tech Lead (accountable) **+ peer review** de um Dev (não o autor da task).
- **Prazo (SLA do gate):** até **8h úteis** após o PR ser aberto; bloqueia merge automático.
- **Se reprovar:** PR volta com *change requests*; CI permanece como required check. Sem merge sem 2 aprovações + CI verde.

**Verificar:**
- [ ] Código segue o `plan.md` e está no **path correto** (Anexo C).
- [ ] Sem anti-padrões de IA: `as any`, `console.log` em produção, `require` dinâmico, validação ausente.
- [ ] **Validação de entrada com Zod** onde há input externo.
- [ ] Toda resposta do assistente inclui o campo **`source_document`** (rastreabilidade de fonte).
- [ ] CI verde (build + lint + testes unitários).
- [ ] Code review tratou os pontos de risco (sem aprovação automática).

**Aprovação:** `[ ]` TL: __________ `[ ]` Peer: __________ Data: ______ | `[ ]` Reprovado — motivo: __________

### GATE 4 — Testes (gerados por IA) → Deploy

**Pergunta-chave:** a suite cobre os cenários que importam, ou só os caminhos felizes da IA? *(Gate mais rígido — entrada em produção.)*

- **Quem aprova:** QA (accountable). Consultado: Tech Lead (autoriza deploy técnico). Orquestra: Delivery Manager.
- **Prazo (SLA do gate):** até **8h úteis** após a suite ser gerada/atualizada.
- **Se reprovar:** suite volta ao QA para complementar cenários; deploy bloqueado.

**Verificar:**
- [ ] Cobertura dos **cenários de falha do Cenário 1** (alucinação, contradição de versões, tier inexistente, pergunta sem cobertura).
- [ ] Testes com **dados reais do domínio** (carga perigosa, SLA Gold, frete 600kg Manaus) — não "test"/"hello".
- [ ] Ao menos um teste de **robustez de IA** (pergunta sobre tier Platinum → negativa, não SLA inventado).
- [ ] Assertions verificam **conteúdo da resposta**, não apenas `toBeDefined()`.
- [ ] Sem regressão na suite existente.

**Aprovação:** `[ ]` QA: __________ `[ ]` TL (deploy): __________ Data: ______ | `[ ]` Reprovado — motivo: __________

### Resumo dos gates

| Gate | Transição | Aprova | Prazo | Se reprovar |
|------|-----------|--------|-------|-------------|
| **G1** | Spec → Plan | Tech Lead | 8h úteis | Volta ao PS; sem `plan.md` |
| **G2** | Tasks (IA) → Implement | Tech Lead | 4h úteis | Re-decomposição pelo Dev |
| **G3** | Código (agente) → Merge | TL + peer Dev | 8h úteis | Change requests; sem merge |
| **G4** | Testes (IA) → Deploy | QA (+ TL) | 8h úteis | Complementar suite; deploy bloqueado |

> **Proporcionalidade ao risco:** G3 (merge na `main`) e G4 (deploy) exigem dupla aprovação e CI verde. G2 é mais leve (4h, aprovação única) por ser transição interna e reversível.

---

<a name="tarefa-3"></a>
# Tarefa 3 — Evidência de uso das ferramentas

> O enunciado pede evidência de que Claude e Claude Cowork foram efetivamente usados e iterados, não um prompt único aceito acriticamente. Abaixo, o registro do processo.

## 3.1 Claude (chat) — elaboração do fluxo (Tarefa 1)

**Sequência de prompts e iteração:**

| # | Prompt (resumo) | O que o output trouxe | Refinamento aplicado |
|---|------------------|------------------------|----------------------|
| 1 | "Mapeie, para o time NovaTech, quais ferramentas de IA cada papel usa nas etapas Spec→Plan→Tasks→Implement→Review→Deploy." | Primeira matriz papel×etapa | Output inicial misturava ferramentas entre papéis (todos com Copilot). Corrigido para diferenciar: Cowork p/ gestão, Design p/ produto. |
| 2 | "Os 2 devs são pleno e sênior — diferencie a atuação deles na matriz." | Distinção pleno × sênior (sênior também consultado em Spec/Deploy) | Incorporado: sênior participa de peer review e é consultado em deploy. |
| 3 | "Onde exatamente entram os 4 gates obrigatórios do enunciado? Eles ficam nas etapas ou nas transições?" | Gates posicionados nas transições | Refinado: explicitado que Plan→Tasks **não** tem gate (proporcionalidade ao risco). |

**Decisão de contexto:** forneci ao Claude apenas o time real e as 6 etapas nomeadas do enunciado, não a documentação inteira da NovaTech — para o fluxo, o Anexo A é ruído. Isso é coerente com a engenharia de contexto da fase anterior (orçamento de atenção).

## 3.2 Claude Cowork — checklist de validation gates (Tarefa 2)

**Como foi usado:** o Cowork gerou o template a partir da instrução "checklist reutilizável de 4 gates, cada um com quem aprova / o que verifica / prazo / o que acontece se reprovar". Iterações:

- **v1:** checklist com 3 campos por gate (sem prazo). → **Faltava o "quanto tempo tem"** exigido na Tarefa 1.3.
- **v2:** adicionado o campo **Prazo (SLA do gate)** em horas úteis, amarrado à lógica de SLA da NovaTech.
- **v3:** itens de verificação genéricos ("código está bom") reescritos para **específicos do domínio** (campo `source_document`, anti-padrões `as any`/`console.log`, teste de tier Platinum).

**Por que template e não preenchido:** o Cowork produziu um formulário com campos `[ ]` em branco e instruções de duplicação, de modo que o time aplique a cada módulo — atende ao critério "executável por qualquer membro".

## 3.3 Julgamento próprio (não aceitação acrítica)

Pontos em que **divergi ou corrigi** o output da IA:
- Recusei a sugestão inicial de colocar gate em **todas** as transições — isso violaria o equilíbrio velocidade/segurança. Mantive gate só onde o risco é real e/ou irreversível.
- Ajustei os prazos propostos pela IA (que eram em dias) para **horas úteis**, coerentes com os SLAs Gold/Silver da própria NovaTech.
- Mantive as referências cruzadas a artefatos de outros papéis (Anexo C, guardrails do PS, cenários de falha do QA) como integração intencional — decisão de Delivery, não da IA.

---

## Anexos

- `fluxo-ai-first.svg` — diagrama visual do fluxo de 6 etapas com os 4 gates.
